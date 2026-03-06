# Prefect Server — Local Deployment on EKS with External RDS PostgreSQL

## Prerequisites
- `kubectl` configured and pointing at the EKS cluster
- `helm` v3 installed
- Access to the RDS PostgreSQL instance via a DB client

---

## Step 1: Prepare the PostgreSQL Database

Connect to your RDS instance using any DB client and run:

```sql
-- Create an isolated schema for Prefect (avoids conflicts with existing tables)
CREATE SCHEMA prefect;

-- Set the default schema for the user so Prefect creates tables there
ALTER USER agupta SET search_path = prefect;
```

> **Why?** The `postgres` database already had a `log` table from another app. Prefect also creates a `log` table during migrations. Using a dedicated schema (`prefect`) isolates Prefect's tables without needing a separate database.

---

## Step 2: Create the Kubernetes Secret

Prefect needs a connection string to reach RDS. Special characters in the password must be **URL-encoded**:

| Character | Encoded |
|-----------|---------|
| `#`       | `%23`   |
| `*`       | `%2A`   |
| `&`       | `%26`   |
| `!`       | `%21`   |

Create the secret in the `rm-dev` namespace:

```bash
kubectl create secret generic prefect-server-db-secret \
  -n rm-dev \
  --from-literal=connection-string='postgresql+asyncpg://agupta:Sv%23%2AmN%26v%21@ltaw-pgdb-153.cluster-chpakk1zvegf.us-east-1.rds.amazonaws.com:5432/postgres'
```

> The secret key must be named `connection-string` — that's what the Helm chart looks for.

---

## Step 3: Create the Values File

The file `charts/prefect-server/local-values.yaml` is already in this repo:

```yaml
# Disable bundled PostgreSQL subchart — using external RDS
postgresql:
  enabled: false

# Reference the manually created secret (do not auto-create)
secret:
  create: false
  name: prefect-server-db-secret

# Single-replica server
server:
  replicaCount: 1
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi

# No Redis or background services
redis:
  enabled: false

backgroundServices:
  runAsSeparateDeployment: false
```

---

## Step 4: Fetch Helm Chart Dependencies

Run once from the repo root:

```bash
helm dependency update charts/prefect-server
```

---

## Step 5: Install the Helm Chart

```bash
helm upgrade --install prefect-server ./charts/prefect-server \
  -f charts/prefect-server/local-values.yaml \
  --set secret.create=false \
  --set secret.name=prefect-server-db-secret \
  --namespace rm-dev \
  --wait
```

---

## Step 6: Verify the Pod is Running

```bash
kubectl get pods -n rm-dev -l app.kubernetes.io/name=prefect-server
```

Expected output:
```
NAME                              READY   STATUS    RESTARTS   AGE
prefect-server-xxxxxxxxx-xxxxx    1/1     Running   0          Xs
```

Check logs for errors:
```bash
kubectl logs -n rm-dev -l app.kubernetes.io/name=prefect-server
```

---

## Step 7: Access the Prefect UI

Run this in a terminal and **keep it open**:

```bash
kubectl port-forward svc/prefect-server -n rm-dev 4200:4200
```

Then open: **http://localhost:4200**

> The tunnel is only active while this command is running. Closing the terminal will close the connection.

---

## Troubleshooting

### `DuplicateTableError: relation "log" already exists`
The target database/schema already has a conflicting table from another app.
**Fix:** Create a new schema and set it as the default search path (see Step 1).

### `invalid literal for int() with base 10: 'Sv'`
Special characters in the password (e.g. `#`, `*`, `&`, `!`) are breaking the URL parser.
**Fix:** URL-encode the password in the connection string (see Step 2).

### `CrashLoopBackOff`
Check logs with `kubectl logs -n rm-dev -l app.kubernetes.io/name=prefect-server` for the specific error.

### Re-deploying after changes
```bash
# Update the secret
kubectl create secret generic prefect-server-db-secret -n rm-dev \
  --from-literal=connection-string='...' \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart the deployment
kubectl rollout restart deployment/prefect-server -n rm-dev
```
