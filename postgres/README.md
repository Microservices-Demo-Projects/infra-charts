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

### What This Chart Creates

**Automatically Created Resources:**

1. **PostgreSQL StatefulSet**: Bitnami PostgreSQL with custom configuration
2. **TLS Certificate**: cert-manager Certificate resource (`postgres-tls`)
3. **ExternalSecret**: Syncs root password from Vault KV (`postgres-root-password`)
4. **Init Script**: Creates `vault` database user with `CREATEROLE` privilege
5. **pg_hba Configuration**: Layered authentication rules for different user types

**Requires Manual Configuration:**

- Vault KV secret (`kv/db/postgres-root`) - must exist before installation
- Vault database secrets engine configuration (for rotating credentials)
- Application database schema and roles (see Phase 1 of verification)
- ExternalSecrets for rotating credentials (optional, see Phase 3)

### Authentication Architecture

This chart implements a layered security model with three authentication methods:

1. **Vault Service Account (mTLS only)**
   - User: `vault`
   - Authentication: Certificate-based (no password)
   - Purpose: Dynamic credential generation and role management
   - Created by: `01-create-vault-user.sql` init script

2. **Vault-Generated Dynamic Credentials (TLS + Password)**
   - Users: `vlt_app_ro_*`, `vlt_app_rw_*`, `vlt_setup_*`
   - Authentication: TLS with password (scram-sha-256)
   - Purpose: Application database access with automatic rotation
   - Lifecycle: Managed by Vault (1-hour default TTL)

3. **Static Credentials (TLS + Password or mTLS)**
   - User: `postgres` (superuser)
   - Authentication: TLS with password or certificate
   - Purpose: Administrative tasks and break-glass access
   - Credential: Stored in Vault KV engine, synced via ExternalSecret

## Prerequisites

- Kubernetes Cluster (OpenShift or Standard).
- Helm 3+ installed.
- [Cert-Manager](../cert-manager/README.md) installed with `demo-ca` ClusterIssuer.
- [HashiCorp Vault](../hashicorp-vault/README.md) installed and unsealed.
- [External Secrets](../external-secrets/README.md) installed and integrated with Vault.

### Vault Prerequisites (REQUIRED)

Before installing this chart, you must create the PostgreSQL root password in Vault's KV engine. The chart includes an ExternalSecret template that will automatically sync this password.

```bash
# 1. Login to Vault
oc exec -ti vault-0 -n vault -- vault login
# (Enter Root Token)

# 2. Create the root password in Vault KV engine
# This path 'kv/db/postgres-root' matches the ExternalSecret template in the chart
oc exec -ti vault-0 -n vault -- vault kv put kv/db/postgres-root password=$(openssl rand -base64 32 | tr -dc 'A-Za-z0-9' | head -c 24)

# 3. Verify the secret was created
oc exec -ti vault-0 -n vault -- vault kv get kv/db/postgres-root
```

> [!NOTE]
> The chart automatically creates:
>
> - A `Certificate` resource for PostgreSQL TLS (using cert-manager)
> - An `ExternalSecret` resource to sync the password from Vault
>
> You only need to ensure the password exists in Vault before installation.

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

oc get events -n postgres
```

### 2. Verify TLS Certificate

The chart automatically creates a Certificate resource. Verify it was issued successfully:

```bash
oc get certificate postgres-tls -n postgres
# STATUS must be "Ready: True"

# View certificate details
oc describe certificate postgres-tls -n postgres
```

### 3. Verify Secret Sync from Vault

The chart automatically creates an ExternalSecret resource. Verify it synced the password from Vault:

```bash
oc get externalsecret postgres-root-password -n postgres
# STATUS must be "SecretSynced"

# Check the synced secret exists
oc get secret postgres-root-password -n postgres

# Verify synchronization details
oc describe externalsecret postgres-root-password -n postgres
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
-- This shows the data configured in the pgHba file in values.yaml file:
SELECT * FROM pg_hba_file_rules;

-- Creating a separate DB for POC apps
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
-- The WITH ADMIN OPTION allows vault to grant these roles to dynamically created users
GRANT app_ro TO vault WITH ADMIN OPTION;
GRANT app_rw TO vault WITH ADMIN OPTION;

-- Verify the vault user now has these permissions
\du vault

-- You should see vault is a member of app_ro and app_rw with admin option
-- Also verify the vault user exists and can authenticate via cert
SELECT * FROM pg_roles WHERE rolname = 'vault';

-- Exit psql
\q
```

**Understanding Authentication Methods (pg_hba.conf)**:

The chart configures PostgreSQL with different authentication methods based on user type:

1. **`vault` user**: mTLS only (cert authentication) - no password fallback
2. **Vault-generated users** (prefix `vlt_`): TLS + password (scram-sha-256)
3. **Other users**: Both mTLS (if cert provided) or TLS + password
4. **Local connections**: Password only (for break-glass access from pod)

### Phase 2: Verify Vault KV Engine Integration (Static Creds)

The chart automatically creates an ExternalSecret to sync the PostgreSQL root password from Vault. Verify this integration is working:

#### 1. Verify ExternalSecret Status

The chart includes an ExternalSecret template that syncs from `kv/db/postgres-root`. Verify it's working:

```bash
# Check ExternalSecret status
oc describe externalsecret postgres-root-password -n postgres

# Verify the secret was created and populated
oc get secret postgres-root-password -n postgres

# (Optional) Retrieve and verify root postgres user's password:
## Directly from vault:
# oc exec vault-0 -n vault -- vault kv get -field=password kv/db/postgres-root
## From the synced secret from vault:
# oc get secret postgres-root-password -o jsonpath='{.data.password}' -n postgres | base64 --decode
```

### Phase 3: Using Vault Database Engine (Rotating Creds)

> [!IMPORTANT]
> **Vault TLS Certificate Prerequisite**: Before configuring the database secrets engine, Vault must have access to TLS certificates to authenticate with PostgreSQL via mTLS. Ensure the `vault-tls` certificate exists in the Vault namespace and is mounted to the Vault pod at `/vault/userconfig/vault-tls/`. This is typically configured in the Vault Helm chart values.

> [!NOTE]
> **What This Chart Provides vs Manual Configuration**:
>
> - **Automatically created by chart**: TLS Certificate, ExternalSecret for root password, `vault` database user (via init script)
> - **Requires manual configuration**: Vault database secrets engine, database roles (app-ro, app-rw, app-setup), ExternalSecrets for rotating credentials
>
> The following steps configure Vault's database secrets engine to generate dynamic, rotating credentials for applications.

#### 1. Configure Vault Database Secrets Engine

Configure the database engine (default path is `database`):

```bash
# 1. Enable Database secrets engine (ignore error if already enabled)
oc exec -ti vault-0 -n vault -- vault secrets enable database

# 2. List the secrets engines to verify
oc exec -ti vault-0 -n vault -- vault secrets list
```

> [!NOTE]
> The `vault` user created by the init DB script authenticates using mutual TLS (mTLS) and does not use a password.
>
> The client supplies the database username (vault), but authentication is performed using the client's private key, client certificate, and a trusted CA certificate. PostgreSQL validates that the client certificate is signed by the trusted CA and that the certificate's Common Name (CN) matches the supplied database username.
>
> The client independently verifies the PostgreSQL server's identity by checking that the server certificate is signed by the trusted CA and that its Subject Alternative Name (SAN) matches the hostname or IP address used in the connection (sslmode=verify-full).

```bash
# 3. Configure connection to Postgres from vault
# Password is not needed in the command below because the chart enforces 'cert' authentication.
oc exec -ti vault-0 -n vault -- vault write database/config/pocdb \
    plugin_name=postgresql-database-plugin \
    allowed_roles="app-ro,app-rw,app-setup" \
    connection_url="postgresql://{{username}}:{{password}}@postgres-postgresql.postgres.svc.cluster.local:5432/pocdb?sslmode=verify-full&sslrootcert=/vault/userconfig/vault-tls/ca.crt&sslcert=/vault/userconfig/vault-tls/tls.crt&sslkey=/vault/userconfig/vault-tls/tls.key" \
    username="vault"

# 3a: App read-only users
# Note: Vault generates unique usernames like "vlt_app_ro_v-root-app-ro-AbCd1234-1234567890"
# The prefix "vlt_app_ro_" is custom and matches the pg_hba rule for vault-generated users
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

# 3c: Setup user (optional Vault role for setup/migration tasks)
oc exec -ti vault-0 -n vault -- vault write database/roles/app-setup \
  db_name=pocdb \
  creation_statements="
    CREATE ROLE \"vlt_setup_{{name}}\" 
    WITH LOGIN PASSWORD '{{password}}'
    VALID UNTIL '{{expiration}}';
    GRANT ALL PRIVILEGES ON DATABASE pocdb TO \"vlt_setup_{{name}}\";
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

# Verify the ExternalSecret
oc get externalsecret -n test-rotating-creds
oc describe externalsecret postgres-rotating-creds -n test-rotating-creds

# View the synced secret
oc get secret postgres-rotating-creds -n test-rotating-creds -o yaml

# Delete if needed:
# oc delete externalsecret postgres-rotating-creds -n test-rotating-creds
```

### Phase 4: Accessing the Database

#### Internal Access (From another namespace)

Applications within the same cluster access the database using the following details:

- **Host (ClusterIP Service)**: `postgres-postgresql.postgres.svc.cluster.local`
- **Port**: `5432`

#### External Access (From local machine)

To connect using a desktop client (DBeaver, PGAdmin, etc.) use port-forwarding:

> [!NOTE]
> PostgreSQL does not support OpenShift Routes with TLS passthrough effectively due to Server Name Indication (SNI) requirements. Port-forwarding is the recommended method for external database access.

1. **Port Forward**: Run the port-forward command below.

    ```bash
    # Forward local port 5432 to the service
    oc port-forward svc/postgres-postgresql 5432:5432 -n postgres
    ```

2. **Retrieve Credentials**:

    ```bash
    # For Static Root Password:
    oc get secret postgres-root-password -n postgres -o jsonpath='{.data.password}' | base64 -d
    
    # For Rotating Credentials:
    oc get secret postgres-rotating-creds -n test-rotating-creds -o jsonpath='{.data.app-ro-username}' | base64 -d
    oc get secret postgres-rotating-creds -n test-rotating-creds -o jsonpath='{.data.app-ro-password}' | base64 -d
    ```

3. **Connection Settings**:
    - **Host**: `localhost`
    - **Port**: `5432`
    - **Database**: `pocdb` (or `postgres` for the default database)
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

# 2. Get credentials (for read-only user example)
DB_USER=$(oc get secret postgres-rotating-creds -n test-rotating-creds -o jsonpath='{.data.app-ro-username}' | base64 -d)
DB_PASS=$(oc get secret postgres-rotating-creds -n test-rotating-creds -o jsonpath='{.data.app-ro-password}' | base64 -d)

# Extract CA for verification
oc get secret postgres-tls -n postgres -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt

# 3. Connect using psql
PGPASSWORD=$DB_PASS psql "host=localhost port=5432 dbname=pocdb user=$DB_USER sslmode=verify-full sslrootcert=ca.crt"
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

# Test TLS connection locally (requires port-forward running)
openssl s_client -connect localhost:5432 -starttls postgres
```

### Common Issues

#### 1. ExternalSecret fails with "cannot read secret data from Vault"

**Cause**: The Vault policy doesn't allow reading from the database secrets engine path.

**Solution**: Ensure your Vault policy includes:

```hcl
path "database/creds/*" {
  capabilities = ["read"]
}
```

#### 2. Vault returns "is not an allowed role"

**Cause**: The database configuration doesn't include the role in `allowed_roles`.

**Solution**: Update the database config:

```bash
oc exec -ti vault-0 -n vault -- vault write database/config/pocdb \
  allowed_roles="app-ro,app-rw,app-setup" \
  # ... other parameters
```

#### 3. Vault returns "permission denied to grant role"

**Cause**: The `vault` user doesn't have permission to grant the `app_ro` or `app_rw` roles.

**Solution**: Grant the roles with ADMIN OPTION:

```sql
GRANT app_ro TO vault WITH ADMIN OPTION;
GRANT app_rw TO vault WITH ADMIN OPTION;
```

#### 4. Certificate verification fails

**Cause**: Missing or incorrect TLS certificates.

**Solution**: Verify certificates exist:

```bash
# Check PostgreSQL certificate
oc get secret postgres-tls -n postgres

# Check Vault certificate (for mTLS to PostgreSQL)
oc exec -ti vault-0 -n vault -- ls -la /vault/userconfig/vault-tls/
```

#### 5. Connection refused from applications

**Cause**: Wrong service name or namespace.

**Solution**: Use the full FQDN:

```txt
postgres-postgresql.postgres.svc.cluster.local:5432
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

---

## Security Best Practices

### 1. Role Naming Convention

The `vlt_` prefix in Vault-generated usernames (e.g., `vlt_app_ro_`, `vlt_app_rw_`) is critical for pg_hba.conf rules. This prefix ensures:

- Vault-generated users authenticate with TLS + password only
- The `vault` user remains restricted to mTLS authentication
- Clear separation between service accounts and dynamic credentials

### 2. Credential Rotation

- Vault automatically expires credentials based on `default_ttl` (1 hour in the examples)
- ExternalSecrets refreshes credentials every 50 minutes (before expiration)
- Applications must handle credential rotation gracefully (reconnect with new credentials)

### 3. Principle of Least Privilege

- `app_ro`: Read-only access to `app` schema tables
- `app_rw`: Read-write access to `app` schema tables  
- `app_setup`: Full database privileges (use only for migrations/setup)
- Never use `postgres` superuser for applications

### 4. Network Security

- TLS is enforced for all connections (no plaintext)
- mTLS provides mutual authentication for the `vault` user
- Internal applications connect via ClusterIP service (no external exposure by default)

### 5. Audit and Monitoring

```bash
# Monitor active connections
oc exec -it postgres-postgresql-0 -n postgres -- psql -U postgres -c "SELECT usename, application_name, client_addr, state FROM pg_stat_activity;"

# Check Vault lease information
oc exec -ti vault-0 -n vault -- vault list sys/leases/lookup/database/creds/app-ro

# Review PostgreSQL logs for authentication attempts
oc logs postgres-postgresql-0 -n postgres | grep -i "authentication\|connection"
```
