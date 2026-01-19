# External Secrets Operator Helm Chart

This is a wrapper Helm chart for External Secrets Operator with HashiCorp Vault integration for OpenShift.

This chart deploys the External Secrets Operator and configures a `ClusterSecretStore` for HashiCorp Vault integration. It allows you to sync secrets from Vault into Kubernetes/OpenShift secrets across any namespace.

> **Note**, that using a ClusterSecretStore is not recommended for production for a multi-tenant cluster. It is better to use a SecretStore which is namespace. We are using a ClusterSecretStore here just for demo purposes as using a ClusterSecretStore means that users having access to create ExternalSecrets in any namespace can access all the secrets from Vault even if they are not needed for the apps running in that namespace.

This guide is divided into two phases to ensure proper dependency management (specifically, the ServiceAccount must exist before Vault configuration):

1. **Phase 1: Installation**: Installing the External Secrets Operator (Controller & CRDs).
2. **Phase 2: Integration**: Configuring the connection to HashiCorp Vault.

---

# Phase 1: Installation

In this phase, we install the operator and its CRDs. By default, the `ClusterSecretStore` integration is **disabled** (`clusterSecretStore.required: false`) to allow the ServiceAccount to be created first.

## Setup

Enter the chart directory and download dependencies.

```shell
cd infra-charts/external-secrets

# Add the official repository
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# Download the subchart
helm dependency update .
```

## Helm Chart Installation and CRD Management

The `external-secrets.installCRDs` setting in the values.yaml controls how Custom Resource Definitions are managed:

### Option 1: Helm-Managed CRDs (Default)

> NOTE: If you were installing the upstream chart directly, you would use:
`helm install external-secrets external-secrets/external-secrets --set installCRDs=true`

But with this wrapper chart, you configure it in values.yaml:

```yaml
external-secrets:
  installCRDs: true  # or omit this property to use chart default `true`
```

**Behavior:**

- Helm installs CRDs during initial `helm install`
- CRDs are NOT upgraded during `helm upgrade` (Helm limitation)
- CRDs are deleted during `helm uninstall`. This cascades to all ExternalSecret resources and their synced Kubernetes secrets if reatin is not configured, so ensure you have backups or alternative secret management before uninstalling ESO.

**Use when:**

- Quick testing or development environments
- You accept the risk of losing all secrets on uninstall
- Or you don't mind manually upgrading CRDs before ever helm upgrade of this chart.

**Installation process:**

```bash
# Install the chart (ClusterSecretStore is disabled by default)
helm install external-secrets . -n external-secrets --create-namespace

# Verify pods are running:
oc get pods -n external-secrets
```

**Upgrade process:**

```bash
# CRDs must be upgraded manually BEFORE helm upgrade
# NOTE: The version v0.10.3 must be the same as chart version. 
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/v0.10.3/deploy/crds/bundle.yaml

# If the chart version is being upgraded apply the corresponding the CRDs manually and then upgrade the chart
helm upgrade external-secrets . -n external-secrets
```

### Option 2: Manual CRD Management (Recommended for Production)

Set the below property in the values.yaml of this chart:

```yaml
external-secrets:
  installCRDs: false
```

**Behavior:**

- Helm does NOT install or manage CRDs
- You must manually install CRDs before first installation as well.
- So CRDs survive `helm uninstall`
- You control when CRD upgrades happen independently.

**Use when:**

- Production environments
- Multiple releases share the same CRDs
- You want CRDs to persist after chart uninstallation
- You need fine-grained control over CRD versions

**Installation process:**

```bash
# 1. Install CRDs first
# NOTE: The version v0.10.3 must be the same as chart version. 
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/v0.10.3/deploy/crds/bundle.yaml

# 2. Then install the chart
helm install external-secrets . -n external-secrets --create-namespace
```

**Upgrade process:**

```bash
# 1. Upgrade CRDs if needed (check release notes)
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/v<new-version>/deploy/crds/bundle.yaml

# 2. Upgrade the chart
helm upgrade external-secrets . -n external-secrets
```

**Uninstall process:**

```bash
# 1. CRDs remain even after uninstalling the chart with this approach.
helm uninstall external-secrets -n external-secrets

# 2. Optionally delete CRDs manually (WARNING: This will delete all ExternalSecret resources and might cascades to Kubernetes Secrets too.)
kubectl delete -f https://raw.githubusercontent.com/external-secrets/external-secrets/v0.10.3/deploy/crds/bundle.yaml
```

---

# Phase 2: External Secret Store Integration

Now that the operator is running and the `external-secrets` ServiceAccount has been created, we can configure HashiCorp Vault to trust it.

## Step 1: Prepare Vault

We will configure the local HashiCorp Vault (deployed in the `vault` namespace) to enable Kubernetes authentication and trust the `external-secrets` ServiceAccount.

**Prerequisites:**

- HashiCorp Vault installed and unsealed in `vault` namespace (see `infra-charts/hashicorp-vault/README.md`).
- External Secrets Operator installed in `external-secrets` namespace (Phase 1 of this readme completed).

Run the following commands:

```bash
# 1. Enable the Kubernetes auth method (if not already enabled)
oc exec -ti vault-0 -n vault -- vault auth enable kubernetes

# 2. Configure the auth method to talk to the local Kubernetes API
oc exec -ti vault-0 -n vault -- vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

# 3. Create a policy named 'secret-policy' that allows reading secrets
echo 'path "secret/*" {
  capabilities = ["read"]
}' | oc exec -i vault-0 -n vault -- vault policy write secret-policy -

# 4. Create the role 'external-secrets' binding the ServiceAccount to the policy
oc exec -ti vault-0 -n vault -- vault write auth/kubernetes/role/external-secrets \
    bound_service_account_names=external-secrets \
    bound_service_account_namespaces=external-secrets \
    policies=external-secrets-readonly \
    ttl=24h
```

## Step 2: Enable ClusterSecretStore

Now that Vault is ready, we update the Helm release to create the `ClusterSecretStore` resource.

**Option A: Command Line Flag**

```bash
# Upgrade the release enabling the store
helm upgrade external-secrets . -n external-secrets \
  --set clusterSecretStore.required=true
```

**Option B: Values File (Recommended)**

1. Edit `values.yaml` and set:

   ```yaml
   clusterSecretStore:
     required: true
   ```

2. Apply the change:

   ```bash
   helm upgrade external-secrets . -n external-secrets
   ```

## Step 3: Verify Integration

Check if the ClusterSecretStore is valid and ready:

```bash
oc get clustersecretstore local-vault-backend -o wide
# STATUS should be "Valid"
```
