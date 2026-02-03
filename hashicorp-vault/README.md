# HashiCorp Vault Wrapper Chart

This chart is a wrapper around the official [`hashicorp/vault`](https://artifacthub.io/packages/helm/hashicorp/vault) chart, configured for OpenShift and standard Kubernetes environments.

> [!NOTE]
> **Installation Sequence**: For a complete platform setup, required by the Demo / POC Apps in this GitHub Org. follow this order:
>
> 1. [Cert-Manager](../cert-manager/README.md)
> 2. [Headlamp](../headlamp/README.md) (Optional for Standard Kubernetes, and not needed for OpenShift)
> 3. **HashiCorp Vault** (Current)
> 4. [External Secrets](../external-secrets/README.md)
> 5. [PostgreSQL](../postgres/README.md)

> [!WARNING]
> While following the below instructions, use the appropriate `kubectl` commands instead of `oc` if you are not in an OpenShift environment.

## Overview

This deployment includes:
- **OpenShift Optimized**: `global.openshift: true` enabled by default.
- **Standalone Mode**: (HA disabled) for simpler development and testing.
- **Persistent Storage**: Enabled (5Gi).
- **OpenShift Route**: Enabled for UI access.
- **Secure by Default**: TLS enabled via cert-manager.

## Prerequisites

- Kubernetes Cluster (OpenShift or Standard).
- Helm 3+ installed.
- [Cert-Manager](../cert-manager/README.md) installed with `demo-ca` ClusterIssuer.

## Platform Configuration

### OpenShift (Default)

By default, this chart is configured for OpenShift:
- `global.openshift: true`
- `server.route.enabled: true`
- `server.ingress.enabled: false`

Vault UI is accessible via the automatically created OpenShift Route.

### Standard Kubernetes

To deploy on standard Kubernetes, update `values.yaml` or pass flags:
- `vault.global.openshift: false`
- `vault.server.route.enabled: false`
- `vault.server.ingress.enabled: false`
- `vault.server.image.repository: hashicorp/vault`
- `vault.server.image.tag: 1.21.2`

Access Vault UI via port-forwarding:
```bash
kubectl port-forward svc/vault-ui 8200:8200 -n vault
```
Then access `https://127.0.0.1:8200` in your browser.

## Installation

### 1. Setup

```bash
# Add repository
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Download dependencies
helm dependency update .
```

### 2. Deploy

```bash
## Option-1: For OpenShift
helm upgrade --install vault . --namespace vault --create-namespace

## Option-2: For Standard Kubernetes
helm upgrade --install vault . --namespace vault --create-namespace \
  --set global.openshift=false \
  --set vault.server.route.enabled=false \
  --set vault.server.ingress.enabled=false \
  --set vault.server.image.repository=hashicorp/vault \
  --set vault.server.image.tag=1.21.2
```

## Post-Deployment: Initialize and Unseal (REQUIRED)

Vault starts in a **sealed** and **uninitialized** state. The pod stays in non-ready state until it is initialized and unsealed.

1. **Verify TLS Certificate Creation**:
   ```bash
   oc get certificate vault-tls -n vault
   oc get secret vault-tls -n vault
   ```

2. **Initialize Vault**:
   ```bash
   oc exec -ti vault-0 -n vault -- vault operator init
   ```
   **IMPORTANT**: Save the "Unseal Keys" and "Initial Root Token" immediately.

3. **Unseal Vault**:
   Run this command 3 times, providing a different Unseal Key each time:
   ```bash
   oc exec -ti vault-0 -n vault -- vault operator unseal
   ```

## Verification

### 1. Check Pod Status
```bash
oc get pods -n vault
```

### 2. Verify Route (OpenShift)
```bash
oc get route vault -n vault
```

### 3. Check Vault Status
```bash
oc exec -ti vault-0 -n vault -- vault status
```

---

## Verification (Test Workflow)

Test Vault integration by creating and retrieving a static secret.

```bash
# 1. Login to Vault
oc exec -ti vault-0 -n vault -- vault login
# (Enter Root Token)

# 2. Enable KV Engine
oc exec -ti vault-0 -n vault -- vault auth enable kubernetes
oc exec -ti vault-0 -n vault -- vault secrets enable -path=kv kv

# 3. Put and Get Secret
oc exec -ti vault-0 -n vault -- vault kv put kv/test-secret message="Hello Vault"
oc exec -ti vault-0 -n vault -- vault kv get kv/test-secret
```

## Configuration

> [!NOTE]
> **Subchart Nesting**: All configurations must be nested under the `vault:` key in `values.yaml`.

Example:
```yaml
vault:
  server:
    dataStorage:
      size: 10Gi
```

Refer to the [Official Vault Helm Configuration](https://developer.hashicorp.com/vault/docs/platform/k8s/helm/configuration) for all available parameters.

## Troubleshooting / Debugging

```bash
# Get events
oc get events -n vault --sort-by='.lastTimestamp'

# Check logs
oc logs vault-0 -n vault

# Describe pod
oc describe pod vault-0 -n vault
```

## Cleanup

```bash
helm uninstall vault --namespace vault
oc delete project vault
```

## Pause & Resume Development

To "turn off" the application without deleting configuration or data (Persistent Volumes), you can scale the replicas to 0.

**To Pause (Stop Pods):**
```bash
oc scale statefulset vault --replicas=0 -n vault
```

**To Resume (Start Pods):**
```bash
oc scale statefulset vault --replicas=1 -n vault
```
**IMPORTANT**: When Vault restarts, it will **seal itself**. You MUST run the unseal commands again.

## Additional Information


## Advanced Operations: Sealing

You might want to manually seal the Vault if you suspect a security intrusion or want to "lock" the vault without stopping the pod.

**When to Seal:**

- **Security Emergency:** You suspect unauthorized access.
- **Maintenance:** You are about to perform sensitive operations.
- **End of Session:** You want to secure keys before leaving your workstation (though pausing is often better for dev).

**How to Seal:**

```bash
oc exec -ti vault-0 -n vault -- vault operator seal
```

*Result: The Vault API stops servicing requests. Use the `unseal` command (Step 3.2) to unlock it again.*

**What happens when Sealed:**

- All secret access is completely blocked.
- Applications cannot authenticate.
- Dynamic secret generation stops.
- Lease renewals fail.
- **Your apps will start failing immediately.**

## Advanced Operations: Rekeying

**Note:** You do **NOT** seal the vault to execute this. Rekeying is done while the vault is **unsealed and running**.

**When to Rekey:**

- **Personnel Change:** A key holder leaves the team or company.
- **Key Compromise:** You suspect a key shard has been exposed/lost.
- **Security Compliance:** Regular rotation policy (e.g., quarterly).
- **Change Thresholds:** You want to change the number of shares (e.g., 5 to 7) or the threshold (e.g., 3 to 5) to increase security.

1. **Initialize Rekeying:**

    ```bash
    oc exec -ti vault-0 -n vault -- vault operator rekey -init -key-shares=5 -key-threshold=3
    ```

2. **Provide Existing Keys:**
    Run this command 3 times (entering a diff *existing* key each time):

    ```bash
    oc exec -ti vault-0 -n vault -- vault operator rekey
    ```

3. **Get New Keys:**
    After the threshold is met, the **NEW** keys will be printed. **Save them immediately!** The old keys are now invalid.
