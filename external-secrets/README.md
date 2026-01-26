# External Secrets Operator Helm Chart

This is a wrapper Helm chart for External Secrets Operator with HashiCorp Vault integration, pre-configured for OpenShift and standard Kubernetes environments.

> [!NOTE]
> **Installation Sequence**: For a complete infrastructure setup, required by the Demo / POC Apps in this GitHub Org. follow this order:
>
> 1. [Cert-Manager](../cert-manager/README.md)
> 2. [Headlamp](../headlamp/README.md) (Optional for OpenShift)
> 3. [HashiCorp Vault](../hashicorp-vault/README.md)
> 4. **External Secrets** (Current)

## Overview

This chart deploys the External Secrets Operator and configures a `ClusterSecretStore` for HashiCorp Vault integration. It allows you to sync secrets from Vault into Kubernetes/OpenShift secrets across any namespace.

> [!WARNING]
> Using a `ClusterSecretStore` is not recommended for production in multi-tenant clusters. For production, consider namespace-scoped `SecretStore` resources. We use `ClusterSecretStore` here for demo simplicity.

## Prerequisites

- Kubernetes Cluster (OpenShift or Standard).
- Helm 3+ installed.
- [HashiCorp Vault](../hashicorp-vault/README.md) installed and unsealed.

## Deployment Phases

This guide is divided into two phases to ensure proper dependency management:

1. **Phase 1: Installation** - Installing the External Secrets Operator (Controller & CRDs).
2. **Phase 2: Integration** - Configuring the connection to HashiCorp Vault.

## Phase 1: Installation

In this phase, we install the operator and its CRDs. By default, the `ClusterSecretStore` integration is **disabled** (`clusterSecretStore.required: false`) to allow the ServiceAccount to be created first.

### Setup

Enter the chart directory and download dependencies.

```bash
cd infra-charts/external-secrets

# Add the official repository
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# Download the subchart
helm dependency update .
```

### Helm Chart Installation and CRD Management

The `external-secrets.installCRDs` setting in values.yaml controls how Custom Resource Definitions are managed:

### Option 1: Helm-Managed CRDs (Default)

> **Note**: If you were installing the upstream chart directly, you would use:
> `helm install external-secrets external-secrets/external-secrets --set installCRDs=true`
>
> But with this wrapper chart, you configure it in values.yaml:

```yaml
external-secrets:
  installCRDs: true  # or omit this property to use chart default `true`
```

**Behavior:**

- Helm installs CRDs during initial `helm install`
- CRDs are NOT upgraded during `helm upgrade` (Helm limitation)
- CRDs are deleted during `helm uninstall`, which cascades to all ExternalSecret resources and their synced Kubernetes secrets if retention is not configured

**Use when:**

- Quick testing or development environments
- You accept the risk of losing all secrets on uninstall
- You don't mind manually upgrading CRDs before helm upgrade of this chart

**Installation:**

```bash
# Install the chart (ClusterSecretStore is disabled by default)
helm install external-secrets . -n external-secrets --create-namespace

# Verify pods are running
oc get pods -n external-secrets
```

**Upgrade:**

```bash
# CRDs must be upgraded manually BEFORE helm upgrade
# NOTE: The version v0.10.3 must match the chart version
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/v0.10.3/deploy/crds/bundle.yaml

# Then upgrade the chart
helm upgrade external-secrets . -n external-secrets
```

**Uninstall:**

```bash
# WARNING: This will delete all ExternalSecret resources and might cascade to Kubernetes Secrets
helm uninstall external-secrets -n external-secrets
```

### Option 2: Manual CRD Management (Recommended for Production)

Set the below property in values.yaml:

```yaml
external-secrets:
  installCRDs: false
```

**Behavior:**

- Helm does NOT install or manage CRDs
- You must manually install CRDs before first installation
- CRDs survive `helm uninstall`
- You control when CRD upgrades happen independently

**Use when:**

- Production environments
- Multiple releases share the same CRDs
- You want CRDs to persist after chart uninstallation
- You need fine-grained control over CRD versions

**Installation:**

```bash
# 1. Install CRDs first
# NOTE: The version v0.10.3 must match the chart version
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/v0.10.3/deploy/crds/bundle.yaml

# 2. Then install the chart
helm install external-secrets . -n external-secrets --create-namespace
```

**Upgrade:**

```bash
# 1. Upgrade CRDs if needed (check release notes)
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/v<new-version>/deploy/crds/bundle.yaml

# 2. Upgrade the chart
helm upgrade external-secrets . -n external-secrets
```

**Uninstall:**

```bash
# 1. Uninstall the chart (CRDs remain)
helm uninstall external-secrets -n external-secrets

# 2. Optionally delete CRDs manually
# WARNING: This will delete all ExternalSecret resources and might cascade to Kubernetes Secrets
kubectl delete -f https://raw.githubusercontent.com/external-secrets/external-secrets/v0.10.3/deploy/crds/bundle.yaml
```

---

## Phase 2: External Secret Store Integration

Now that the operator is running and the `external-secrets` ServiceAccount has been created, we can configure HashiCorp Vault to trust it.

### Prerequisites

> [!IMPORTANT]
> Complete these prerequisites before proceeding:

1. **HashiCorp Vault Setup**: Follow the [HashiCorp Vault README](../hashicorp-vault/README.md) to:
   - Deploy Vault to the `vault` namespace
   - Initialize and unseal Vault
   - Save your root token - you'll need it for the integration steps below
2. **External Secrets Operator**: Installed in `external-secrets` namespace (Phase 1 of this guide completed)

### Step 1: Prepare Vault

We will configure the local HashiCorp Vault (deployed in the `vault` namespace) to enable Kubernetes authentication and trust the `external-secrets` ServiceAccount.

#### Login to Vault

You must authenticate as a root user (or a user with sufficient permissions) to configure auth methods.

```bash
# Log in to Vault interactively
oc exec -ti vault-0 -n vault -- vault login
# (Paste your Initial Root Token when prompted)
```

**Expected Output:**

```text
Token (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                <root-token>
token_accessor       <token-accessor>
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

#### Configure Vault

Run the following commands to set up Kubernetes authentication:

```bash
# 1. Enable the Kubernetes auth method (if not already enabled)
oc exec -ti vault-0 -n vault -- vault auth enable kubernetes

# 2. Configure the auth method with Kubernetes API details
# This uses Vault's own ServiceAccount credentials to authenticate to the Kubernetes API
oc exec -ti vault-0 -n vault -- sh -c 'vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token'

# 3. Enable KV engine at path 'kv' (if not already enabled)
oc exec -ti vault-0 -n vault -- /bin/sh -c 'vault secrets list | grep -q "^kv/" || vault secrets enable -path=kv kv'

# 4. Create a Vault policy for External Secrets
# The policy controls what secrets the External Secrets Operator can access.
# Choose one of the examples below based on your requirements and secret engine types:

# Example 1: KV Version 2 Secret Engine (Default - Recommended for Production)
# KV v2 stores data at 'path/data/*' and metadata at 'path/metadata/*'
# This is the most common secret engine for static key-value secrets
echo 'path "kv/data/*" {
  capabilities = ["read"]
}' | oc exec -i vault-0 -n vault -- vault policy write external-secrets-readonly-policy -

# Example 2: All Secret Engines (POC/Development Only)
# WARNING: This grants read access to ALL secrets in Vault - use only for POC/testing!
# This is useful when you want to quickly test without worrying about specific paths
echo 'path "*" {
  capabilities = ["read", "list"]
}' | oc exec -i vault-0 -n vault -- vault policy write external-secrets-readonly-policy -

# Example 3: Multiple Secret Engine Types (Recommended for Production Multi-Service Environments)
# This sample policy combines access to different types of secret engines
# Useful when your application needs various types of secrets
echo '# KV v2 for static application secrets
path "kv/data/app/*" {
  capabilities = ["read"]
}

# Database engine for dynamic database credentials
path "database/creds/postgres-readonly" {
  capabilities = ["read"]
}
path "database/creds/mysql-app" {
  capabilities = ["read"]
}

# AWS engine for dynamic AWS credentials
path "aws/creds/s3-readonly" {
  capabilities = ["read"]
}

# PKI for certificate issuance
path "pki/issue/app-cert" {
  capabilities = ["create", "update"]
}

# Transit for encryption operations
path "transit/encrypt/app-key" {
  capabilities = ["update"]
}
path "transit/decrypt/app-key" {
  capabilities = ["update"]
}' | oc exec -i vault-0 -n vault -- vault policy write external-secrets-readonly-policy -

# 5. Create the role 'external-secrets' binding the ServiceAccount to the policy
oc exec -ti vault-0 -n vault -- vault write auth/kubernetes/role/external-secrets \
    bound_service_account_names=external-secrets \
    bound_service_account_namespaces=external-secrets \
    policies=external-secrets-readonly-policy \
    ttl=24h
```

#### Attaching Multiple Vault Policies to a Service Account Token

Vault supports attaching multiple policies to a single Kubernetes role, which allows fine-grained access control by combining different policy permissions. This is useful when you want to:

- Separate concerns (e.g., one policy for app secrets, another for database credentials)
- Gradually add permissions without modifying existing policies
- Implement least-privilege access by composing small, focused policies

**How It Works:**

When multiple policies are attached to a role, the resulting token will have the **union** of all permissions from all attached policies. This means the token can access any path allowed by any of the policies.

**Example: Creating and Attaching Multiple Policies**

```bash
# Create a first policy for application secrets (KV v2 engine)
echo 'path "kv/data/app/*" {
  capabilities = ["read"]
}' | oc exec -i vault-0 -n vault -- vault policy write app-secrets-policy -

# Create a second policy for database credentials (Database engine)
echo 'path "database/creds/postgres-readonly" {
  capabilities = ["read"]
}
path "database/creds/mysql-app" {
  capabilities = ["read"]
}' | oc exec -i vault-0 -n vault -- vault policy write db-creds-policy -

# Create a third policy for AWS credentials (AWS engine)
echo 'path "aws/creds/s3-readonly" {
  capabilities = ["read"]
}
path "aws/sts/deploy-role" {
  capabilities = ["read"]
}' | oc exec -i vault-0 -n vault -- vault policy write aws-creds-policy -

# Attach all three policies to the external-secrets role
# Note: Policies are specified as a comma-separated list
oc exec -ti vault-0 -n vault -- vault write auth/kubernetes/role/external-secrets \
    bound_service_account_names=external-secrets \
    bound_service_account_namespaces=external-secrets \
    policies=app-secrets-policy,db-creds-policy,aws-creds-policy \
    ttl=24h
```

**Updating an Existing Role with Additional Policies:**

If you already have a role configured and want to add more policies:

```bash
# Read the current role configuration to see existing policies
oc exec -ti vault-0 -n vault -- vault read auth/kubernetes/role/external-secrets

# Update the role with additional policies (this replaces the policies list)
# Make sure to include ALL policies you want, not just the new ones
oc exec -ti vault-0 -n vault -- vault write auth/kubernetes/role/external-secrets \
    bound_service_account_names=external-secrets \
    bound_service_account_namespaces=external-secrets \
    policies=external-secrets-readonly-policy,new-policy-name \
    ttl=24h
```

**Verification:**

```bash
# Verify the policies attached to the role
oc exec -ti vault-0 -n vault -- vault read auth/kubernetes/role/external-secrets

# Expected output will show the policies field with all attached policies:
# policies    [app-secrets-policy db-creds-policy aws-creds-policy]
```

### Step 2: Enable ClusterSecretStore

Now that Vault is ready, we update the Helm release to create the `ClusterSecretStore` resource.

#### Option A: Command Line Flag Override

```bash
# Upgrade the release enabling the store
helm upgrade external-secrets . -n external-secrets \
  --set clusterSecretStore.required=true
```

#### Option B: Updating Values File

1. Edit `values.yaml` and set:

   ```yaml
   clusterSecretStore:
     required: true
   ```

2. Apply the change:

   ```bash
   helm upgrade external-secrets . -n external-secrets
   ```

### Step 3: Verify Integration

Check if the ClusterSecretStore is valid and ready:

```bash
oc get clustersecretstore local-vault-backend -o wide
# STATUS should be "Valid", READY should be "True"
```

If the status shows `InvalidProviderConfig`, check the troubleshooting section below.

---

## Verification (Test Workflow)

To verify the integration between External Secrets Operator and HashiCorp Vault, perform a manual test using a temporary secret.

### 1. Create a Test Secret in Vault

First, create a secret in the Vault instance (running in the `vault` namespace).

```bash
# Enable the KV engine (v2) at path 'kv' if not already enabled
oc exec -ti vault-0 -n vault -- /bin/sh -c 'vault secrets list | grep -q "^kv/" || vault secrets enable -path=kv kv'

# Create a secret named 'kv/test-secret' with a username and password
oc exec -ti vault-0 -n vault -- vault kv put kv/test-secret username=admin password=changeit
```

### 2. Create an ExternalSecret

Create a new namespace and apply an `ExternalSecret` resource that fetches the secret we just created.

```bash
# Create a temporary namespace
oc create ns test-external-secrets

# Apply a test ExternalSecret resource
oc apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: test-secret
  namespace: test-external-secrets
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: local-vault-backend
    kind: ClusterSecretStore
  target:
    name: my-synced-secret
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: kv/test-secret
      property: username
  - secretKey: password
    remoteRef:
      key: kv/test-secret
      property: password
EOF
```

### 3. Verify Synchronization

Check if the ExternalSecret is valid and if the Kubernetes Secret has been created.

```bash
# Check ExternalSecret status (Should be "SecretSynced")
oc get externalsecret test-secret -n test-external-secrets

# Describe for detailed status
oc describe externalsecret test-secret -n test-external-secrets

# Check if the secret was created and verify the content
oc get secret my-synced-secret -n test-external-secrets -o jsonpath='{.data.username}' | base64 -d
# Should output: admin
```

### 4. Cleanup Test Resources

Remove the test resources created during verification.

```bash
# Delete the test namespace (deletes the ExternalSecret and the synced Secret)
oc delete ns test-external-secrets

# Delete the secret from Vault
oc exec -ti vault-0 -n vault -- vault kv delete kv/test-secret
```

> [!NOTE]
> This cleanup only removes the test resources. To completely remove the External Secrets integration, see the "Complete Cleanup" section below.

---

## Troubleshooting

### ClusterSecretStore shows "InvalidProviderConfig"

**Symptoms:**
```bash
oc get clustersecretstore local-vault-backend -o wide
# STATUS: InvalidProviderConfig, READY: False
```

**Common Causes:**

1. **Vault not configured properly**: Ensure you completed all steps in "Step 1: Prepare Vault"
2. **Vault auth method not enabled**: Run `oc exec -ti vault-0 -n vault -- vault auth list` to verify `kubernetes/` is listed
3. **Incorrect kubernetes_host**: Should be `https://kubernetes.default.svc:443`
4. **ServiceAccount doesn't exist**: Verify with `oc get sa external-secrets -n external-secrets`

**Debug:**
```bash
# Check ClusterSecretStore events
oc describe clustersecretstore local-vault-backend

# Check external-secrets operator logs
oc logs -n external-secrets -l app.kubernetes.io/name=external-secrets --tail=50
```

### ExternalSecret shows "SecretSyncedError"

**Symptoms:**
```bash
oc get externalsecret test-secret -n test-external-secrets
# STATUS: SecretSyncedError, READY: False
```

**Common Causes:**

1. **ClusterSecretStore not ready**: First fix the ClusterSecretStore issue
2. **Secret doesn't exist in Vault**: Verify the secret path is correct
3. **Policy doesn't allow read**: Check the Vault policy grants read access to the secret path

**Debug:**
```bash
# Check ExternalSecret events
oc describe externalsecret test-secret -n test-external-secrets

# Verify the secret exists in Vault
oc exec -ti vault-0 -n vault -- vault kv get kv/test-secret
```

### Permission Denied from Vault

**Symptoms:**
```
Error: permission denied
Code: 403
```

**Solution:**

Ensure Vault's Kubernetes auth is configured with the correct ServiceAccount token and CA certificate:

```bash
# Reconfigure Vault auth with proper credentials including CA cert and token
oc exec -ti vault-0 -n vault -- sh -c 'vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token'

# Verify the configuration
oc exec -ti vault-0 -n vault -- vault read auth/kubernetes/config
# Check that kubernetes_ca_cert is populated (not "n/a")
```

---

## Cleanup

To completely remove the External Secrets integration and all related resources:

### Step 1: Remove ClusterSecretStore

```bash
# This will prevent any ExternalSecrets from syncing
helm upgrade external-secrets . -n external-secrets \
  --set clusterSecretStore.required=false

# Or delete it directly
oc delete clustersecretstore local-vault-backend
```

### Step 2: Clean Up Vault Configuration

Remove the Kubernetes auth configuration and policies from Vault:

```bash
# Delete the Vault role
oc exec -ti vault-0 -n vault -- vault delete auth/kubernetes/role/external-secrets

# Delete the policy
oc exec -ti vault-0 -n vault -- vault delete sys/policy/external-secrets-readonly-policy

# Optional: Disable the Kubernetes auth method (only if not used by other services)
# oc exec -ti vault-0 -n vault -- vault auth disable kubernetes

# Optional: Disable the KV secrets engine (only if not used for other secrets)
# oc exec -ti vault-0 -n vault -- vault secrets disable kv
```

### Step 3: Uninstall External Secrets Operator

```bash
# Uninstall the Helm release
helm uninstall external-secrets -n external-secrets

# Optional: Delete the namespace
oc delete namespace external-secrets
```

### Step 4: Clean Up CRDs (Optional)

> [!WARNING]
> Only do this if you're completely removing External Secrets and have no other installations using these CRDs.

```bash
# Delete CRDs (this will delete ALL ExternalSecret resources cluster-wide)
kubectl delete -f https://raw.githubusercontent.com/external-secrets/external-secrets/v0.10.3/deploy/crds/bundle.yaml
```

> [!NOTE]
> After cleanup, if you want to redeploy:
>
> 1. Start fresh with Phase 1 (Installation)
> 2. Follow Phase 2 (Integration) to reconfigure Vault

## Pause & Resume Development

To "turn off" the External Secrets Operator application without deleting configuration or data, you can scale the replicas to 0.

**To Pause (Stop Pods):**

```bash
oc scale deployment external-secrets --replicas=0 -n external-secrets
```

**To Resume (Start Pods):**

```bash
oc scale deployment external-secrets --replicas=1 -n external-secrets
```

*Note: Since External Secrets is a controller, scaling back to 1 replica will restart the controller and it will continue to function normally.*

**IMPORTANT:** When External Secrets restarts, it should continue to function normally without any additional steps required. The configurations and secret sync will remain intact.

The operator will continue to monitor for ExternalSecret resources and will resume syncing when the pods are restarted. No reinitialization or reconfiguration is needed.
