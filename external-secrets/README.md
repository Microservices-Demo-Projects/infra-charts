# External Secrets Operator Wrapper Chart

This is a wrapper Helm chart for External Secrets Operator with HashiCorp Vault integration, pre-configured for OpenShift and standard Kubernetes.

> [!NOTE]
> **Installation Sequence**: For a complete infrastructure setup, required by the Demo / POC Apps in this GitHub Org. follow this order:
>
> 1. [Cert-Manager](../cert-manager/README.md)
> 2. [Headlamp](../headlamp/README.md) (Optional for Standard Kubernetes, and not needed for OpenShift)
> 3. [HashiCorp Vault](../hashicorp-vault/README.md)
> 4. **External Secrets** (Current)
> 5. [PostgreSQL](../postgres/README.md)

> [!WARNING]
> While following the below instructions, use the appropriate `kubectl` commands instead of `oc` if you are not in an OpenShift environment.

## Overview

This chart deploys the External Secrets Operator (ESO) and configures a `ClusterSecretStore` for secure mTLS integration with HashiCorp Vault. 

> [!WARNING]
> Using a `ClusterSecretStore` is not recommended in multi-tenant Kubernetes clusters. For more safer and restricted access to secrets, consider namespace-scoped `SecretStore` resources. This chart uses `ClusterSecretStore` for demo simplicity.

## Prerequisites

- Kubernetes Cluster (OpenShift or Standard).
- Helm 3+ installed.
- [Cert-Manager](../cert-manager/README.md) installed with `demo-ca` ClusterIssuer.
- [HashiCorp Vault](../hashicorp-vault/README.md) installed and unsealed.

## Platform Configuration

### OpenShift and Standard Kubernetes

This chart works on OpenShift and Standard Kubernetes by default no customization is required.

## Installation

### Setup


```bash
# Add the official repository
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# Change directory to the external-secrets chart directory
# Download the subchart
helm dependency update .
```

### ESO Controller & CRD Deployment


The `external-secrets.installCRDs` setting in values.yaml controls how Custom Resource Definitions are managed:

#### Option 1: Helm-Managed CRDs (Default)

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

```bash
# Install the chart
helm upgrade --install external-secrets . -n external-secrets --create-namespace
```

> [!NOTE]
> **Process to Upgrade Helm when installed with Option 1 (Helm Managed CRD):**
>
>
> ```bash
> # CRDs must be upgraded manually BEFORE helm upgrade
> # NOTE: The version v0.10.3 must match the chart version
> kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/v0.10.3/deploy/crds/bundle.yaml
> 
> # Then upgrade the chart
> helm upgrade external-secrets . -n external-secrets
> ```


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

```bash
# 1. Install CRDs first
# NOTE: The version v0.10.3 must match the chart version
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/v0.10.3/deploy/crds/bundle.yaml

# 2. Then install the chart
helm upgrade --install external-secrets . -n external-secrets --create-namespace --set external-secrets.installCRDs=false
```

> [!NOTE]
> **Process to Upgrade Helm when installed with Option 2 (Manual CRD Management):**
> 
> ```bash
> # 1. Upgrade CRDs if needed (check release notes)
> kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/v<new-version>/deploy/crds/bundle.yaml
> 
> # 2. Upgrade the chart
> helm upgrade external-secrets . -n external-secrets
> ```

## Vault External Secret Store Integration

After the ESO Controller & CRD are installed successfully, Configure HashiCorp Vault to trust the `external-secrets` ServiceAccount.

> [!IMPORTANT]
> Ensure that the Hashicorp Vault application is initialized and unsealed before proceeding. Follow the [HashiCorp Vault README](../hashicorp-vault/README.md) to details on this.

### 1. Configure Vault

```bash
# Login to Vault
oc exec -ti vault-0 -n vault -- vault login
# (Paste your Initial Root Token when prompted)

# Enable Kubernetes authentication
oc exec -ti vault-0 -n vault -- vault auth enable kubernetes

# Configure the auth method with Kubernetes API details
# This uses Vault's own ServiceAccount credentials to authenticate to the Kubernetes API
oc exec -ti vault-0 -n vault -- sh -c 'vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token'

# Create a Vault policy for External Secrets
# The policy controls what secrets the External Secrets Operator can access.
#--------------------------------------------------------------
# This policy allows access to all secrets in Vault.
# Use this for testing purposes only.
echo 'path "*" {
  capabilities = ["read", "list"]
}' | oc exec -i vault-0 -n vault -- vault policy write external-secrets-readonly-policy -

#--------------------------------------------------------------
# Example 1: KV Secret Engine (Supports both v1 (kv/*) and v2 (kv/data/*))
# This policy is more inclusive and works for both engine versions.

# echo 'path "kv/*" {
#   capabilities = ["read", "list"]
# }
# path "kv/data/*" {
#   capabilities = ["read", "list"]
# }' | oc exec -i vault-0 -n vault -- vault policy write external-secrets-readonly-policy -

#--------------------------------------------------------------
# Example 2: Multiple Secret Engine Types (Recommended for Production Multi-Service Environments)
# This sample policy combines access to different types of secret engines
# Useful when your application needs various types of secrets

# echo '# KV v2 for static application secrets
# path "kv/data/app/*" {
#   capabilities = ["read", "list"]
# }

# # Database engine for dynamic database credentials
# path "database/creds/postgres-readonly" {
#   capabilities = ["read", "list"]
# }
# path "database/creds/mysql-app" {
#   capabilities = ["read", "list"]
# }

# # AWS engine for dynamic AWS credentials
# path "aws/creds/s3-readonly" {
#   capabilities = ["read", "list"]
# }

# # PKI for certificate issuance
# path "pki/issue/app-cert" {
#   capabilities = ["create", "update"]
# }

# # Transit for encryption operations
# path "transit/encrypt/app-key" {
#   capabilities = ["update"]
# }
# path "transit/decrypt/app-key" {
#   capabilities = ["update"]
# }' | oc exec -i vault-0 -n vault -- vault policy write external-secrets-readonly-policy -


# Create the role 'external-secrets' binding the ServiceAccount to the policy
oc exec -ti vault-0 -n vault -- vault write auth/kubernetes/role/external-secrets \
    bound_service_account_names=external-secrets \
    bound_service_account_namespaces=external-secrets \
    policies=external-secrets-readonly-policy \
    ttl=24h

# Verify the policies attached to the role
oc exec -ti vault-0 -n vault -- vault read auth/kubernetes/role/external-secrets
```

> [!NOTE]
>
> Attaching Multiple Vault Policies to a Service Account Token
> 
> Vault supports attaching multiple policies to a single Kubernetes role, which allows fine-grained > access control by combining different policy permissions. This is useful when you want to:
> 
> - Separate concerns (e.g., one policy for app secrets, another for database credentials)
> - Gradually add permissions without modifying existing policies
> - Implement least-privilege access by composing small, focused policies
> 
> **How It Works:**
> 
> When multiple policies are attached to a role, the resulting token will have the **union** of all > permissions from all attached policies. This means the token can access any path allowed by any of the > policies.
> 
> **Example: Creating and Attaching Multiple Policies**
> 
> ```bash
> # Create a first policy for application secrets (KV v2 engine)
> echo 'path "kv/data/app/*" {
>   capabilities = ["read"]
> }' | oc exec -i vault-0 -n vault -- vault policy write app-secrets-policy -
> 
> # Create a second policy for database credentials (Database engine)
> echo 'path "database/creds/postgres-readonly" {
>   capabilities = ["read"]
> }
> path "database/creds/mysql-app" {
>   capabilities = ["read"]
> }' | oc exec -i vault-0 -n vault -- vault policy write db-creds-policy -
> 
> # Create a third policy for AWS credentials (AWS engine)
> echo 'path "aws/creds/s3-readonly" {
>   capabilities = ["read"]
> }
> path "aws/sts/deploy-role" {
>   capabilities = ["read"]
> }' | oc exec -i vault-0 -n vault -- vault policy write aws-creds-policy -
> 
> # Attach all three policies to the external-secrets role
> # Note: Policies are specified as a comma-separated list
> oc exec -ti vault-0 -n vault -- vault write auth/kubernetes/role/external-secrets \
>     bound_service_account_names=external-secrets \
>     bound_service_account_namespaces=external-secrets \
>     policies=app-secrets-policy,db-creds-policy,aws-creds-policy \
>     ttl=24h
> ```
> 
> **Updating an Existing Role with Additional Policies:**
> 
> If you already have a role configured and want to add more policies:
> 
> ```bash
> # Read the current role configuration to see existing policies
> oc exec -ti vault-0 -n vault -- vault read auth/kubernetes/role/external-secrets
> 
> # Update the role with additional policies (this replaces the policies list)
> # Make sure to include ALL policies you want, not just the new ones
> oc exec -ti vault-0 -n vault -- vault write auth/kubernetes/role/external-secrets \
>     bound_service_account_names=external-secrets \
>     bound_service_account_namespaces=external-secrets \
>     policies=external-secrets-readonly-policy,new-policy-name \
>     ttl=24h
> ```

### 2. Enable ClusterSecretStore

Update the Helm release to enable the `ClusterSecretStore` template creation.

```bash
helm upgrade external-secrets . -n external-secrets \
  --set clusterSecretStore.required=true
```

## Verification

### 1. Check Pod Status
```bash
oc get pods -n external-secrets
```

### 2. Verify TLS Certificate
```bash
oc get certificate external-secrets-tls -n external-secrets
# STATUS must be "Ready: True"
```

### 3. Verify ClusterSecretStore
```bash
oc get clustersecretstore local-vault-backend -o wide
# STATUS must be "Valid", READY must be "True"
```

---

## Verification (Test Workflow)

Verify the integration by syncing a test secret from Vault.

```bash
# 1. Create a Test Secret in Vault
oc exec -ti vault-0 -n vault -- vault kv put kv/test-secret username=admin password=changeit

# 2. Create an ExternalSecret
oc apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: test-secret
  namespace: default
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

# 3. Verify Synchronization
oc get externalsecret test-secret
# STATUS must be "SecretSynced"

# 4. Check the secret data
oc get secret my-synced-secret -o jsonpath='{.data.username}' | base64 -d
oc get secret my-synced-secret -o jsonpath='{.data.password}' | base64  -d
```

## Configuration

> [!NOTE]
> **Subchart Nesting**: All configurations for the operator must be nested under the `external-secrets:` key in `values.yaml`.

Example:
```yaml
external-secrets:
  installCRDs: false
```

Refer to the [Official External Secrets Documentation](https://external-secrets.io/latest/introduction/overview/) for all available parameters.

## Troubleshooting / Debugging

```bash
# Check operator logs
oc logs -n external-secrets -l app.kubernetes.io/name=external-secrets --tail=50

# Check ClusterSecretStore events
oc describe clustersecretstore local-vault-backend

# Check ExternalSecret status
oc describe externalsecret test-secret
```

## Cleanup

> [!WARNING]
> If `Option 1: Helm-Managed CRDs` was used during initial installation then running the below simple helm uninstall will delete all CRDs as well i.e., the ExternalSecret resources will get deleted and will cascade to Kubernetes Secrets, casuing potential disruptions.

```bash
helm uninstall external-secrets -n external-secrets
```

> [!WARNING]
> If `Option 2: Manual CRD Management` was used during initial installation then running the below simple helm uninstall will *`not delete`* all CRDs.
> But again even here deleting the CRDs explicitly will delete all ExternalSecret resources and will cascade to Kubernetes Secrets cluster-wide.

```bash
helm uninstall external-secrets -n external-secrets
oc delete project external-secrets

# In this case the CRDs, will need to be exiplicitly deleted as they are managed outside the scope of helm.
oc delete -f https://raw.githubusercontent.com/external-secrets/external-secrets/v0.10.3/deploy/crds/bundle.yaml
```

## Pause & Resume Development

**To Pause (Stop Pods):**
```bash
oc scale deployment external-secrets --replicas=0 -n external-secrets
```

**To Resume (Start Pods):**
```bash
oc scale deployment external-secrets --replicas=1 -n external-secrets
```
