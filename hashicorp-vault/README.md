# HashiCorp Vault Wrapper Chart

This chart is a wrapper around the official [`hashicorp/vault`](https://artifacthub.io/packages/helm/hashicorp/vault) chart, pre-configured for OpenShift and standard Kubernetes environments.

> [!NOTE]
> **Installation Sequence**: For a complete infrastructure setup, required by the Demo / POC Apps in this GitHub Org. follow this order:
>
> 1. [Cert-Manager](../cert-manager/README.md)
> 2. [Headlamp](../headlamp/README.md) (Optional for OpenShift)
> 3. **HashiCorp Vault** (Current)
> 4. [External Secrets](../external-secrets/README.md)

## Overview

This deployment includes:
- `global.openshift: true` enabled by default.
- **Standalone Mode** (HA disabled) for simpler development.
- Persistent Storage enabled (5Gi).
- OpenShift Route enabled for UI access.

## Prerequisites

- Kubernetes Cluster (OpenShift or Standard).
- Helm 3+ installed.
- [Cert-Manager](../cert-manager/README.md) installed.
- **ClusterIssuer** named `demo-ca` (used for Vault TLS).

## Platform Configuration

### OpenShift (Default)

By default, this chart is configured for OpenShift:

- `global.openshift: true`
- `server.route.enabled: true`
- `server.ingress.enabled: false`

#### Accessing Vault UI In OpenShift

Vault UI will be accessible via the automatically created OpenShift Route.

### Standard Kubernetes

To deploy on standard Kubernetes, update `values.yaml` or pass flags:

- `vault.global.openshift: false`
- `vault.server.route.enabled: false`
- `vault.server.ingress.enabled: false` (Use port-forwarding for access)

**Note:** For standard Kubernetes, we use the official `hashicorp/vault` image instead of the Red Hat UBI image to avoid potential security context issues that can occur in standard Kubernetes clusters.

#### Accessing Vault UI In Standard Kubernetes

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

Vault starts in a **sealed** and **uninitialized** state. The pod will NOT be "Ready" until it is initialized and unsealed.

1. **Verify TLS Certificate Creation:**
   Wait for cert-manager to issue the certificate.

   ```bash
   oc get certificate -n vault
   oc get secret vault-tls -n vault
   ```

2. **Initialize Vault:**

   ```bash
   oc get pods -n vault
   
   oc exec -ti vault-0 -n vault -- vault operator init
   ```

   **IMPORTANT:** Save the "Unseal Keys" and "Initial Root Token" immediately! You will need them for the next step.

3. **Unseal Vault:**
   Run this command 3 times (or however many your threshold is), providing a different Unseal Key each time:

   ```bash
   oc exec -ti vault-0 -n vault -- vault operator unseal
   ```

4. **Verify:**
   After unsealing, the pod should become Ready.

   ```bash
   oc get pods -n vault
   ```

## Verification

Verify the deployment is running and accessible over HTTPS.

```bash
# Check Pod status
oc get pods -n vault

# Check the Route (URL) for OpenShift
oc get route -n vault
# Check the Ingress (URL) for Standard Kubernetes
kubectl get ingress -n vault
# Note: For standard Kubernetes, you access Vault via the Ingress host (e.g., https://valut-demo.local).
# Ensure your /etc/hosts or DNS resolves this domain to your Ingress Controller IP.

# Check Vault Status (inside the pod, using the configured CA)
oc exec -ti vault-0 -n vault -- vault status
```

## Configuration

This chart is a **wrapper** around the official [hashicorp/vault](https://artifacthub.io/packages/helm/hashicorp/vault) chart. This means you can use ANY configuration option supported by the official chart in your `values.yaml`.

#### **How to Customize**

1. **Check Official Docs**: Visit the [Vault Helm Configuration Page](https://developer.hashicorp.com/vault/docs/platform/k8s/helm/configuration) to see all available options (e.g., resource limits, ingress settings, audit logs).
2. **Edit `values.yaml`**: Add the settings you need under the `server` (or other) sections.
    - *Example: Changing Resource Limits*

        ```yaml
        server:
           resources:
             requests:
               memory: "512Mi"
               cpu: "500m"
        ```

3. **Apply Changes**:

    ```bash
    helm upgrade vault . --namespace vault
    ```

#### **Common Configurations**

- **Image Tag**: Change `server.image.tag` to upgrade Vault versions.
- **Storage Size**: Change `server.dataStorage.size` (Default: 5Gi).
- **Service Type**: Change `ui.serviceType` (Default: ClusterIP).

## Troubleshooting

If things aren't working as expected:

```bash
# Get events in the namespace
oc get events -n vault --sort-by='.lastTimestamp'

# Check logs of the vault-0 pod
oc logs vault-0 -n vault

# Describe the pod to see failure reasons (like scheduling issues or image pull errors)
oc describe pod vault-0 -n vault
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

*Note: Since HA is disabled, we scale back to 1 replica.*

**IMPORTANT:** When Vault restarts (after scaling up or a cluster restart), it interprets this as a "system restart" and will **seal itself**. You **MUST** run the unseal commands (Step 3.2 above) again to make it operational. You do not need to re-initialize; just unseal with your existing keys.

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

## Cleanup

To remove the deployment completely:

```bash
helm uninstall vault --namespace vault

# Optional: Delete the namespace and PVCs (Wait for uninstall to finish first)
oc delete project vault
# or for standard kubernetes
kubectl delete namespace vault
```

---

## Next Steps

Once Vault is running and unsealed, you can integrate it with other services:

- **External Secrets Operator**: Automatically sync secrets from Vault to Kubernetes Secrets
  - See [External Secrets README](../external-secrets/README.md) for setup instructions
  - Make sure to save your root token - you'll need it for the integration steps
