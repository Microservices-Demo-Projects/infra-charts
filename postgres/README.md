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
# Install the chart
helm upgrade --install postgres . --namespace postgres --create-namespace
```

## Verification

### 1. Check Pod Status

```bash
oc get pods -n postgres

oc logs -n postgres -f statefulset/postgres-postgresql
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

#### 1. Create Secret in Vault
```bash
# Login to Vault
oc exec -ti vault-0 -n vault -- vault login
# (Enter Root Token)

# Put secret in KV engine
oc exec -ti vault-0 -n vault -- vault kv put kv/db/postgres-root password=myrootpassword
```

#### 2. Sync Secret with ExternalSecret
The chart includes a template for `postgres-root-password` ExternalSecret pointing to `kv/db/postgres-root`. Ensure it syncs:
```bash
oc describe externalsecret postgres-root-password -n postgres
```

### Phase 2: Using Vault Database Engine (Rotating Creds)

#### 1. Configure Vault Database Secrets Engine
```bash
# 1. Enable Database secrets engine
oc exec -ti vault-0 -n vault -- vault secrets enable database

# 2. Configure connection to Postgres
# Note: Use the internal ClusterIP service URL: postgres.postgres.svc.cluster.local
oc exec -ti vault-0 -n vault -- vault write database/config/postgresql \
    plugin_name=postgresql-database-plugin \
    allowed_roles="privileged-user" \
    connection_url="postgresql://{{username}}:{{password}}@postgres.postgres.svc.cluster.local:5432/postgres?sslmode=verify-full&sslrootcert=/vault/userconfig/vault-tls/ca.crt" \
    username="postgres" \
    password="myrootpassword"

# 3. Create a rotating role
oc exec -ti vault-0 -n vault -- vault write database/roles/privileged-user \
    db_name=postgresql \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT ALL PRIVILEGES ON DATABASE postgres TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

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

**Option B: OpenShift Route (If configured)**
*Note: Postgres requires SNI for Route-based access if using PASSTHROUGH TLS. Port-forward is required for database access.*

#### Testing Connection with `psql`
Use the credentials synced from Vault:

```bash
# Get credentials from the synced secret
DB_USER=$(oc get secret postgres-rotating-creds -n postgres -o jsonpath='{.data.username}' | base64 -d)
DB_PASS=$(oc get secret postgres-rotating-creds -n postgres -o jsonpath='{.data.password}' | base64 -d)

# Connect using psql (requires local psql client)
PGPASSWORD=$DB_PASS psql -h localhost -U $DB_USER -p 5432 -d postgres
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
