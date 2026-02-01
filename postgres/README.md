# Postgres Wrapper Chart

This is a wrapper Helm chart for [Bitnami PostgreSQL](https://artifacthub.io/packages/helm/bitnami/postgresql), pre-configured for OpenShift and standard Kubernetes environments.

> [!NOTE]
> **Installation Sequence**: For a complete infrastructure setup, required by the Demo / POC Apps in this GitHub Org. follow this order:
>
> 1. [Cert-Manager](../cert-manager/README.md)
> 2. [Headlamp](../headlamp/README.md) (Optional for OpenShift)
> 3. [HashiCorp Vault](../hashicorp-vault/README.md)
> 4. [External Secrets](../external-secrets/README.md)
> 5. **PostgreSQL** (Current)

## Overview

This chart deploys a PostgreSQL instance with:
- **TLS Support**: Secure connections using certificates issued by cert-manager.
- **Vault Integration**: Automated secret synchronization via External Secrets Operator.
- **OpenShift Optimized**: Default security context and volume permission settings for OpenShift SCC compatibility.
- **Cross-Namespace Ready**: Documented access for applications in other namespaces.

## Prerequisites

- Kubernetes Cluster (OpenShift or Standard).
- Helm 3+ installed.
- [Cert-Manager](../cert-manager/README.md) installed with `demo-ca` ClusterIssuer.
- [HashiCorp Vault](../hashicorp-vault/README.md) installed and unsealed.
- [External Secrets](../external-secrets/README.md) installed and integrated with Vault.

### Vault Prerequisites (REQUIRED)

The root password must be created in Vault **before** applying this chart so the ExternalSecret can sync it.

```bash
# 1. Login to Vault
oc exec -ti vault-0 -n vault -- vault login
# (Enter Root Token)

# 2. Put secret in KV engine
# This path 'kv/db/postgres-root' matches the default in externalsecret.yaml
oc exec -ti vault-0 -n vault -- vault kv put kv/db/postgres-root password=myrootpassword
```

## Platform Configuration

### OpenShift (Default)

By default, this chart is configured for OpenShift compatibility:
- `postgresql.podSecurityContext.enabled: false`
- `postgresql.containerSecurityContext.enabled: false`
- `postgresql.volumePermissions.enabled: false`

These settings allow the OpenShift Security Context Constraints (SCC) to manage the security and volume permissions automatically.

### Standard Kubernetes

To deploy on standard Kubernetes, you must re-enable security contexts:

```yaml
postgresql:
  podSecurityContext:
    enabled: true
    fsGroup: 1001
  containerSecurityContext:
    enabled: true
    runAsUser: 1001
  volumePermissions:
    enabled: true
```

## Installation

### 1. Setup

```bash
# Add the Bitnami repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Download the subchart
helm dependency update .
```

### 2. Deploy

```bash
# For OpenShift
helm upgrade --install postgres . --namespace postgres --create-namespace

# For Standard Kubernetes
helm upgrade --install postgres . --namespace postgres --create-namespace \
  --set postgresql.podSecurityContext.enabled=true \
  --set postgresql.podSecurityContext.fsGroup=1001 \
  --set postgresql.containerSecurityContext.enabled=true \
  --set postgresql.containerSecurityContext.runAsUser=1001 \
  --set postgresql.volumePermissions.enabled=true
```

## Verification

### 1. Check Pod Status

```bash
oc get pods -n postgres

oc logs -n postgres -f statefulset/postgres-postgresql

oc events -n postgres
```

### 2. Verify TLS Certificate
```bash
oc get certificate postgres-tls -n postgres
# STATUS must be "Ready: True"
```

### 3. Verify Initial Secret Sync
```bash
oc get externalsecret postgres-root-password -n postgres
# STATUS must be "SecretSynced"
```

---

## Verification (Test Workflow)

This workflow demonstrates how to manage PostgreSQL credentials using HashiCorp Vault (both KV engine and Database Secret engine) and how to access the database.

### Phase 1: Using Vault KV Engine (Static Creds)

#### 1. Sync Secret with ExternalSecret
The chart includes a template for `postgres-root-password` ExternalSecret pointing to `kv/db/postgres-root`. Ensure it syncs:
```bash
oc describe externalsecret postgres-root-password -n postgres
```

### Phase 2: Using Vault Database Engine (Rotating Creds)

#### 1. Configure Vault Database Secrets Engine

Configure the database engine (default path is `database`):
```bash
# 1. Enable Database secrets engine (ignore error if already enabled)
oc exec -ti vault-0 -n vault -- vault secrets enable database

# 2. List the secrets engines to verify
oc exec -ti vault-0 -n vault -- vault secrets list

# 3. Configure connection to Postgres
# Note: Authenticated by client certificate (commonName: vault). 
# Password is NOT needed because the chart enforces 'cert' authentication by default.
# The 'vault' user is automatically created by the Postgres chart's initialization scripts.

# (Optional) Retrieve root password if you ever want to switch to password auth
# ROOT_PASS=$(oc exec vault-0 -n vault -- vault kv get -field=password kv/db/postgres-root)

# Password is not needed in below command because the chart enforces 'cert' authentication by default.
oc exec -ti vault-0 -n vault -- vault write database/config/postgresql \
    plugin_name=postgresql-database-plugin \
    allowed_roles="privileged-user" \
    connection_url="postgresql://{{username}}:{{password}}@postgres-postgresql.postgres.svc.cluster.local:5432/postgres?sslmode=verify-full&sslrootcert=/vault/userconfig/vault-tls/ca.crt&sslcert=/vault/userconfig/vault-tls/tls.crt&sslkey=/vault/userconfig/vault-tls/tls.key" \
    username="vault" # password="$ROOT_PASS"

# 3. Create a rotating role
oc exec -ti vault-0 -n vault -- vault write database/roles/privileged-user \
    db_name=postgresql \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT ALL PRIVILEGES ON DATABASE postgres TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

> [!WARNING]
> **Authentication Method Conflict**: 
> By default, this chart requires Certificate Authentication (`cert`) for all SSL connections. 
> However, Vault's rotating credentials (usernames/passwords generated in this Phase) do **not** have certificates. 
> To allow applications to use these rotating credentials, you MUST eventually modify `pg_hba.conf` to allow `scram-sha-256` (password) authentication over SSL.

#### 2. Create ExternalSecret for Rotating Creds
Apply the following manifest to sync rotating credentials:

```bash
oc apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: postgres-rotating-creds
  namespace: postgres
spec:
  refreshInterval: 50m
  secretStoreRef:
    name: local-vault-backend
    kind: ClusterSecretStore
  target:
    name: postgres-rotating-creds
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: database/creds/privileged-user
        property: username
    - secretKey: password
      remoteRef:
        key: database/creds/privileged-user
        property: password
EOF
```

### Phase 3: Accessing the Database

#### Internal Access (From another namespace)
Applications within the same cluster access the database using the following details:
- **Host**: `postgres-postgresql.postgres.svc.cluster.local`
- **Port**: `5432`

#### External Access (From local machine)

**Option A: Port-Forwarding (Standard K8s / OpenShift)**
```bash
# Forward local port 5432 to the service
oc port-forward svc/postgres-postgresql 5432:5432 -n postgres
```

#### Local GUI Client Access (DBeaver, PGAdmin, etc.)

To connect using a desktop client:

1.  **Port Forward**: Ensure the port-forward command above is running.
2.  **Retrieve Credentials**:
    ```bash
    # For Static Root Password:
    oc get secret postgres-root-password -n postgres -o jsonpath='{.data.password}' | base64 -d
    
    # For Rotating Credentials:
    oc get secret postgres-rotating-creds -n postgres -o jsonpath='{.data.username}' | base64 -d
    oc get secret postgres-rotating-creds -n postgres -o jsonpath='{.data.password}' | base64 -d
    ```
3.  **Connection Settings**:
    - **Host**: `localhost`
    - **Port**: `5432`
    - **Database**: `postgres`
    - **Authentication**: Use the retrieved Username and Password.
4.  **SSL/TLS Configuration**:
    - Since TLS is enabled, you must configure SSL in your client.
    - **SSL Mode**: `verify-full` or `require`.
    - **Root Certificate**: Extract the CA certificate from the `postgres-tls` secret:
      ```bash
      oc get secret postgres-tls -n postgres -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
      ```
    - Provide this `ca.crt` as the **Root Certificate / CA Certificate** in your client's SSL settings.

**Option B: OpenShift Route (If configured)**
*Note: Postgres requires SNI for Route-based access if using PASSTHROUGH TLS. Port-forward is required for database access.*

#### Testing Connection with `psql`
Use the credentials synced from Vault:

```bash
# 1. Port forward (in another terminal)
oc port-forward svc/postgres-postgresql 5432:5432 -n postgres

# 2. Get credentials
DB_USER=$(oc get secret postgres-rotating-creds -n postgres -o jsonpath='{.data.username}' | base64 -d)
DB_PASS=$(oc get secret postgres-rotating-creds -n postgres -o jsonpath='{.data.password}' | base64 -d)
# Extract CA for verification
oc get secret postgres-tls -n postgres -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt

# 3. Connect using psql
# We use 'hostaddr=127.0.0.1' and 'host=postgres-postgresql.postgres.svc.cluster.local' 
# to satisfy SNI/Hostname verification if required, while connecting to localhost.
PGPASSWORD=$DB_PASS psql "host=localhost port=5432 dbname=postgres user=$DB_USER sslmode=verify-full sslrootcert=ca.crt"
```

---

## Configuration

> [!NOTE]
> **Subchart Nesting**: Since this chart is a **wrapper** around the Bitnami PostgreSQL chart, all subchart configurations must be nested under the `postgresql:` key in `values.yaml`.

Example: To change the database name:
```yaml
postgresql:
  auth:
    database: my_custom_db
```

Refer to the [Bitnami PostgreSQL Helm Chart](https://artifacthub.io/packages/helm/bitnami/postgresql) for all available parameters.

## Debugging

```bash
# Check postgres logs
oc logs -n postgres -l app.kubernetes.io/instance=postgres

# Check ExternalSecret synchronization errors
oc describe externalsecret -n postgres

# Test TLS connection locally
openssl s_client -connect localhost:5432 -starttls postgres
```

## Cleanup

```bash
helm uninstall postgres -n postgres
oc delete project postgres
```

## Pause & Resume Development

**To Pause (Stop Pods):**
```bash
oc scale statefulset postgres-postgresql --replicas=0 -n postgres
```

**To Resume (Start Pods):**
```bash
oc scale statefulset postgres-postgresql --replicas=1 -n postgres
```
