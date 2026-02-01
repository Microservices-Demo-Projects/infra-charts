# Headlamp Chart Customizations

This chart installs [Headlamp](https://headlamp-k8s.github.io/headlamp/), an easy-to-use and extensible Kubernetes web UI.

> [!WARNING]
> **OpenShift Users**: OpenShift includes a built-in web console. This Headlamp installation is **only for standard Kubernetes clusters**. If you're running OpenShift, skip this installation.

> [!NOTE]
> **Installation Sequence**: For a complete infrastructure setup, required by the Demo / POC Apps in this GitHub Org. follow this order:
>
> 1. [Cert-Manager](../cert-manager/README.md)
> 2. **Headlamp** (Current)
> 3. [HashiCorp Vault](../hashicorp-vault/README.md)
> 4. [External Secrets](../external-secrets/README.md)

## Overview

Headlamp provides a modern, developer-friendly dashboard for managing Kubernetes clusters. This installation is pre-configured with TLS support via Cert-Manager and provides internal service access.

## Prerequisites

- Standard Kubernetes cluster (Headlamp is not needed for OpenShift, skip the steps in this readme if you are running OpenShift).
- Helm 3+ installed.
- [Cert Manager](../cert-manager/README.md) installed with the `demo-ca` ClusterIssuer.

## Installation

### 1. Setup Namespace and Certificates

Create the `headlamp` namespace and a TLS certificate for secure communication.

```bash
kubectl create namespace headlamp

kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: headlamp-tls
  namespace: headlamp
spec:
  secretName: headlamp-tls
  commonName: "headlamp"
  dnsNames:
    - headlamp
    - headlamp.headlamp.svc
    - headlamp.headlamp.svc.cluster.local
    - localhost
  issuerRef:
    name: demo-ca
    kind: ClusterIssuer
EOF
```

### 2. Deploy Headlamp

Add the repository and install the chart using the local `values.yaml`.

```bash
helm repo add headlamp https://headlamp-k8s.github.io/headlamp/
helm repo update

# Check available versions
helm search repo headlamp/headlamp --versions

# Download dependencies (if any)
helm dependency update .

# Install the chart
helm upgrade --install headlamp headlamp/headlamp \
  --namespace headlamp \
  --values values.yaml \
  --version 0.39.0
```

### 3. Apply Probe Fix (Required for TLS)

The current Headlamp chart (v0.39.0) has a limitation where health check probes are hardcoded to `HTTP`. Since we are serving over `HTTPS`, we must patch the deployment to use the correct scheme.

```bash
kubectl patch deployment headlamp -n headlamp --type=json -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/livenessProbe/httpGet/scheme", "value": "HTTPS"},{"op": "replace", "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/scheme", "value": "HTTPS"}]'
```

## Verification

Check the status to ensure the patch was applied and the pod becomes `READY 1/1`.

### 1. Status & Events

```bash
# General status check
kubectl get pods,svc,certificate -n headlamp

# Check events for recent errors or warnings
kubectl get events -n headlamp --sort-by='.lastTimestamp'

# Check service details
kubectl get svc headlamp -n headlamp -o yaml
```

### 2. Certificate Status

```bash
# Verify certificate status
kubectl get certificate headlamp-tls -n headlamp

# Check for certificate secrets
kubectl get secret headlamp-tls -n headlamp
```

### 3. Logs

```bash
kubectl logs -n headlamp -l app.kubernetes.io/name=headlamp
```

## Access the Dashboard

Use port-forwarding to access the UI securely:

```bash
kubectl port-forward -n headlamp svc/headlamp 8443:443
```

Access at: `https://localhost:8443`

### Authentication

To log in, you need an access token. You can use the default admin service account or create a custom one with restricted access.

#### 1. Default Admin Token

The chart creates a `headlamp` ServiceAccount in the `headlamp` namespace with `cluster-admin` access via the `headlamp-admin` ClusterRoleBinding.

#### 2. Custom View-Only Token (Recommended for safety)

For users who only need to view resources without making changes, create a restricted service account using the built-in `view` ClusterRole.

**Apply Configuration:**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: headlamp-view
  namespace: headlamp
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: headlamp-view-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- kind: ServiceAccount
  name: headlamp-view
  namespace: headlamp
EOF
```

#### Token Retrieval Methods

There are two primary ways to retrieve a token for authentication.

| Method | Best For | Security | Persistence |
| :--- | :--- | :--- | :--- |
| **Ephemeral Token** | Manual access, debugging, one-time sessions. | **High**: Token has a limited lifetime and is not stored in the cluster. | Short-lived (Default 1h). |
| **Secret-based Token** | Long-running integrations, CI/CD, or older clients. | **Lower**: Token is stored in a permanent Secret resource. | Persistent until deleted. |

##### Approach A: Ephemeral Token (Recommended)

This is the most secure method because the token is generated on the fly and expires automatically.

```bash
# For Admin access:
kubectl create token headlamp --namespace headlamp

# For View-Only access:
kubectl create token headlamp-view --namespace headlamp
```

##### Approach B: Secret-based Token

Use this if you need a persistent token that doesn't expire.

```bash
# 1. Create the Secret (e.g., for headlamp-view)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: headlamp-view-token
  namespace: headlamp
  annotations:
    kubernetes.io/service-account.name: "headlamp-view"
type: kubernetes.io/service-account-token
EOF

# 2. Retrieve the Token
kubectl get secret headlamp-view-token -n headlamp -o jsonpath="{.data.token}" | base64 --decode
```

## Cleanup

```bash
helm uninstall headlamp -n headlamp
kubectl delete namespace headlamp
```

## Pause & Resume Development

To "turn off" the Headlamp application without deleting configuration or data, you can scale the replicas to 0.

**To Pause (Stop Pods):**

```bash
kubectl scale deployment headlamp --replicas=0 -n headlamp
```

**To Resume (Start Pods):**

```bash
kubectl scale deployment headlamp --replicas=1 -n headlamp
```

*Note: Since Headlamp is a web application, scaling back to 1 replica will restart the application and it will continue to function normally.*

**IMPORTANT:** When Headlamp restarts, it continues to function normally without any additional steps required. The web interface and configurations remain intact.
