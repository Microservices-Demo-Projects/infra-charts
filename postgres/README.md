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

- **TLS and mTLS Support**: Secure connections using certificates issued by cert-manager.
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
oc exec -ti vault-0 -n vault -- vault kv put kv/db/postgres-root password=$(openssl rand -base64 128 | tr -dc 'A-Za-z0-9' | head -c 24)
```

## Platform Configuration

### OpenShift (Default)

By default, this chart is configured for OpenShift compatibility:

- `postgresql.podSecurityContext.enabled: false`
- `postgresql.containerSecurityContext.enabled: false`
- `postgresql.volumePermissions.enabled: false`

These settings allow the OpenShift Security Context Constraints (SCC) to manage the security and volume permissions automatically.

### Standard Kubernetes

To deploy on standard Kubernetes, you must enable security contexts as follows in the `values.yaml` file:

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

### Phase 1: Setting up a new DB and Roles

```bash
# Get the password for superuser postgres:
oc get secret postgres-root-password -o jsonpath='{.data.password}' -n postgres | base64 --decode

oc exec -it postgres-postgresql-0 -n postgres -- bash
  $ psql -U postgres
  # Password for user postgres:

  # Run the below commands in psql, the command-line interface to PostgreSQL:
```

```sql
-- This shows the data configure in the pgHba file in values.yaml file:
SELECT * FROM pg_hba_file_rules;

-- Creating a spearate DB for POC apps
CREATE DATABASE pocdb;

--List all databases 
\list

--Connect to the new database and verify
\c pocdb

--Check the current database:
SELECT current_database();

-- create application schema
CREATE SCHEMA app;

-- create static roles (NOLOGIN)
CREATE ROLE app_ro NOLOGIN;
CREATE ROLE app_rw NOLOGIN;

-- grant schema access
GRANT USAGE ON SCHEMA app TO app_ro, app_rw;

-- grant table-level privileges for existing tables
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_ro;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_rw;

-- optional: future-proof for new tables
ALTER DEFAULT PRIVILEGES IN SCHEMA app 
GRANT SELECT ON TABLES TO app_ro;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;

-- Grant the vault user permission to grant app_ro and app_rw roles
GRANT app_ro TO vault WITH ADMIN OPTION;
GRANT app_rw TO vault WITH ADMIN OPTION;
-- GRANT app_setup TO vault WITH ADMIN OPTION;

-- Verify the vault user now has these permissions
\du vault


-- You should see vault is a member of app_ro and app_rw with admin option
-- Also verify the vault user exists and can authenticate via cert
SELECT * FROM pg_roles WHERE rolname = 'vault';
```

### Phase 1: Using Vault KV Engine (Static Creds)

#### 1. Sync Secret with ExternalSecret

The chart includes a template for `postgres-root-password` ExternalSecret pointing to `kv/db/postgres-root`. Ensure it syncs:

```bash
oc describe externalsecret postgres-root-password -n postgres

# (Optional) Retrieve and verify root postgres user's password by running:
## Directly from valut:
# oc exec vault-0 -n vault -- vault kv get -field=password kv/db/postgres-root)
## From the synced secret form valut:
# oc get secret postgres-root-password -o jsonpath='{.data.password}' -n postgres | base64 --decode
```

### Phase 2: Using Vault Database Engine (Rotating Creds)

#### 1. Configure Vault Database Secrets Engine

Configure the database engine (default path is `database`):

```bash
# 1. Enable Database secrets engine (ignore error if already enabled)
oc exec -ti vault-0 -n vault -- vault secrets enable database

# 2. List the secrets engines to verify
oc exec -ti vault-0 -n vault -- vault secrets list
```

> [!NOTE]
> The `vault` user created by the init DB script does not have a password and will authenticate using mTLS i.e.,
> instead of password the client's key, cert, and the common CA cert are used to connect to the DB.
> Here, the `commonName` of the client's cert must match the valid target username (commonName: vault)
> in DB, and the SAN list is also verfied by the client, so that the connection request is being made to the valid DB Hostname having the
> correct hostname / IP.

```bash
# 3. Configure connection to Postgres from vault
# Password is not needed in below command because the chart enforces 'cert' authentication.
oc exec -ti vault-0 -n vault -- vault write database/config/pocdb \
    plugin_name=postgresql-database-plugin \
    allowed_roles="app-ro,app-rw,app-setup" \
    connection_url="postgresql://{{username}}:{{password}}@postgres-postgresql.postgres.svc.cluster.local:5432/pocdb?sslmode=verify-full&sslrootcert=/vault/userconfig/vault-tls/ca.crt&sslcert=/vault/userconfig/vault-tls/tls.crt&sslkey=/vault/userconfig/vault-tls/tls.key" \
    username="vault" # password="..."

# 3a: App read-only users
oc exec -ti vault-0 -n vault -- vault write database/roles/app-ro \
  db_name=pocdb \
  creation_statements="
    CREATE ROLE \"vlt_app_ro_{{name}}\" 
    WITH LOGIN PASSWORD '{{password}}'
    VALID UNTIL '{{expiration}}';
    GRANT app_ro TO \"vlt_app_ro_{{name}}\";
  " \
  default_ttl="1h" \
  max_ttl="24h"

# 3b: App read-write users
oc exec -ti vault-0 -n vault -- vault write database/roles/app-rw \
  db_name=pocdb \
  creation_statements="
    CREATE ROLE \"vlt_app_rw_{{name}}\" 
    WITH LOGIN PASSWORD '{{password}}'
    VALID UNTIL '{{expiration}}';
    GRANT app_rw TO \"vlt_app_rw_{{name}}\";
  " \
  default_ttl="1h" \
  max_ttl="24h"

# 3c: Setup user (optional Vault role if you want to rotate password)
oc exec -ti vault-0 -n vault -- vault write database/roles/app-setup \
  db_name=pocdb \
  creation_statements="
    CREATE ROLE \"vlt_app_setup_{{name}}\" 
    WITH LOGIN PASSWORD '{{password}}'
    VALID UNTIL '{{expiration}}';
    GRANT ALL PRIVILEGES ON DATABASE pocdb TO \"vlt_app_setup_{{name}}\";
  " \
  default_ttl="1h" \
  max_ttl="24h"

# Test app-ro role
oc exec -ti vault-0 -n vault -- vault read database/creds/app-ro

# Test app-rw role
oc exec -ti vault-0 -n vault -- vault read database/creds/app-rw

# Test app-setup role
oc exec -ti vault-0 -n vault -- vault read database/creds/app-setup
```

> [!NOTE]
> The vault user created during database initialization authenticates using mutual TLS (mTLS) and does not use a password.
>
> The client still supplies the database username (vault), but authentication is performed using the client’s private key, client certificate, and a trusted CA certificate. PostgreSQL validates that the client certificate is signed by the trusted CA and that the certificate’s Common Name (CN) matches the supplied database username.
>
> The client independently verifies the PostgreSQL server’s identity by checking that the server certificate is signed by the trusted CA and that its Subject Alternative Name (SAN) matches the hostname or IP address used in the connection (sslmode=verify-full).

#### 2. Create ExternalSecret for Rotating Creds

Apply the following manifest to sync rotating credentials:

```bash
oc create project test-rotating-creds
# or
kubectl create ns test-rotating-creds

oc apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: postgres-rotating-creds
  namespace: test-rotating-creds
spec:
  refreshInterval: 50m
  secretStoreRef:
    name: local-vault-backend
    kind: ClusterSecretStore
  target:
    name: postgres-rotating-creds
    creationPolicy: Owner
  data:
    - secretKey: app-ro-username
      remoteRef:
        key: database/creds/app-ro
        property: username
    - secretKey: app-ro-password
      remoteRef:
        key: database/creds/app-ro
        property: password
    - secretKey: app-rw-username
      remoteRef:
        key: database/creds/app-rw
        property: username
    - secretKey: app-rw-password
      remoteRef:
        key: database/creds/app-rw
        property: password
    - secretKey: app-setup-username
      remoteRef:
        key: database/creds/app-setup
        property: username
    - secretKey: app-setup-password
      remoteRef:
        key: database/creds/app-setup
        property: password
EOF

# oc delete externalsecret postgres-rotating-creds -n test-rotating-creds
# oc describe externalsecret postgres-rotating-creds -n test-rotating-creds
oc get externalsecret -n test-rotating-creds
oc get secret postgres-rotating-creds -n test-rotating-creds -o yaml

```

### Phase 3: Accessing the Database

#### Internal Access (From another namespace)

Applications within the same cluster access the database using the following details:

- **Host (ClusterIP Service)**: `postgres-postgresql.postgres.svc.cluster.local`
- **Port**: `5432`

#### External Access (From local machine)

To connect using a desktop client (DBeaver, PGAdmin, etc.) we can use:

##### Option A: Port-Forwarding works for Standard K8s / OpenShift

##### Option B: OpenShift Route (If configured)

> Note: Postgres requires Server Name Indication (SNI) for Route-based access if using PASSTHROUGH TLS. Port-forward is required for database access.*

1. **Port Forward**: Ensure the port-forward command below is running.

    ```bash
    # Forward local port 5432 to the service
    oc port-forward svc/postgres-postgresql 5432:5432 -n postgres

    # Or
    # Get the route details:
    oc get route -n postgres
    ```

2. **Retrieve Credentials**:

    ```bash
    # For Static Root Password:
    oc get secret postgres-root-password -n postgres -o jsonpath='{.data.password}' | base64 -d
    
    # For Rotating Credentials:
    oc get secret postgres-rotating-creds -n postgres -o jsonpath='{.data.username}' | base64 -d
    oc get secret postgres-rotating-creds -n postgres -o jsonpath='{.data.password}' | base64 -d
    ```

3. **Connection Settings**:
    - **Host**: `localhost`
    - **Port**: `5432`
    - **Database**: `postgres`
    - **Authentication**: Use the retrieved Username and Password.

4. **SSL/TLS Configuration**:
    - Since TLS is enabled, you must configure SSL in your client.
    - **SSL Mode**: `verify-full` or `require`.
    - **Root Certificate**: Extract the CA certificate from the `postgres-tls` secret:

      ```bash
      oc get secret postgres-tls -n postgres -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
      ```

    - Provide this `ca.crt` as the **Root Certificate / CA Certificate** in your client's SSL settings.

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
