# Cert-Manager Wrapper Chart

This chart is a wrapper around the official [`jetstack/cert-manager`](https://artifacthub.io/packages/helm/cert-manager/cert-manager) chart, configured for OpenShift and standard Kubernetes environments.

> [!NOTE]
> **Installation Sequence**: For a complete infrastructure setup, required by the Demo / POC Apps in this GitHub Org. follow this order:
>
> 1. **Cert-Manager** (Current)
> 2. [Headlamp](../headlamp/README.md) (Optional for Standard Kubernetes, and not needed for OpenShift)
> 3. [HashiCorp Vault](../hashicorp-vault/README.md)
> 4. [External Secrets](../external-secrets/README.md)
> 5. [PostgreSQL](../postgres/README.md)

> [!WARNING]
> While following the below instructions, use the appropriate `kubectl` commands instead of `oc` if you are not in an OpenShift environment.

## Overview

The `cert-manager` automates certificate management in Kubernetes, issuing certificates from various sources like Let's Encrypt, HashiCorp Vault, or self-signed.

This wrapper chart includes the following ClusterIssuers and CA certificate templates:
- **ClusterIssuer**: `self-signed` - A self-signed ClusterIssuer.
- **ClusterIssuer**: `demo-ca` - A ClusterIssuer that uses a root CA certificate.
- **Certificate**: `demo-root-ca` - A root CA certificate used to sign other certificates.

These resources are created using Helm hooks to ensure they're installed after cert-manager CRDs are available.

## Prerequisites

- Kubernetes Cluster (OpenShift or Standard).
- Helm 3+ installed.

## Platform Configuration

### OpenShift and Standard Kubernetes

This chart works on OpenShift and Standard Kubernetes by default no customization is required.

## Installation

### 1. Setup

```bash
# Add repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Change directory to the cert-manager chart directory
# Download dependencies
helm dependency update .
```

### 2. Deploy

```bash
# Install the chart
helm upgrade --install cert-manager . --namespace cert-manager --create-namespace
```

> [!WARNING]
> **Helm Hooks**: The custom templates in this wrapper chart creating demo CA resources use Helm `post-install` and `post-upgrade` hooks to ensure they are created after cert-manager's CRDs are installed.

## Verification

### 1. Check Pod Status
```bash
oc get pods -n cert-manager
```

### 2. Verify CRDs
```bash
oc get crd | grep cert-manager
```

### 3. Wait for Readiness
```bash
oc wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=120s
```

### 4. Verify Demo CA Resources
```bash
oc get clusterissuer
oc get certificate demo-root-ca -n cert-manager
```

---

## Verification (Test Workflow)

After installing cert-manager, create a test certificate using the demo CA issuer.

```bash
# 1. Create a test namespace
oc create project test-certs

# 2. Apply a test Certificate
oc apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-cert
  namespace: test-certs
spec:
  secretName: example-cert-tls
  duration: 2160h # 90 days
  renewBefore: 240h # 10 days
  privateKey:
    algorithm: RSA
    size: 4096
    rotationPolicy: Always
  commonName: "example.mydomain.com"
  dnsNames:
    - "example.mydomain.com"
    - "www.example.mydomain.com"
  subject:
    organizationalUnits:
    - "Demo Apps"
    organizations:
    - "Demo Apps"
    countries:
    - "CA"
  usages:
    - server auth
    - client auth
    - digital signature
    - key encipherment
  issuerRef:
    name: demo-ca
    kind: ClusterIssuer
    group: cert-manager.io
EOF

# 3. Verify the certificate
oc get certificate example-cert -n test-certs
# STATUS must be "Ready: True"

oc get secret example-cert-tls -n test-certs
```

## Configuration

> [!NOTE]
> **Subchart Nesting**: Since this chart is a **wrapper** around the official [jetstack/cert-manager](https://artifacthub.io/packages/helm/cert-manager/cert-manager) chart. All configurations of the `cert-manager` must be nested under the `cert-manager:` key of `values.yaml`. This is a standard Helm pattern for wrapper charts, where you place any configuration intended for a subchart under a key matching its `dependencies[].name` mentioned in `Chart.yaml` file.
>
> For example: In the official `jetstack/cert-manager` chart, `crds.enabled` is a root-level property. In this wrapper chart, that same property must be at `cert-manager.crds.enabled` so Helm knows to "pass it down" to the `cert-manager` dependency.
>
> If you were to move `crds.enabled` to the top level (outside the `cert-manager:` block), the subchart would ignore it and wouldn't install the CRDs because the value would be outside its scope.


Refer to the [Official cert-manager Documentation](https://artifacthub.io/packages/helm/cert-manager/cert-manager) for all available parameters.

Alternatively, you can also see all available configuration properties that you can override from your terminal by running:

```bash
helm show values jetstack/cert-manager --version v1.19.2
```

## Troubleshooting / Debugging

```bash
# Get events
oc get events -n cert-manager --sort-by='.lastTimestamp'

# Check logs
oc logs -n cert-manager -l app.kubernetes.io/instance=cert-manager

# Describe CRDs
oc describe crd certificates.cert-manager.io
```

## Cleanup

```bash
helm uninstall cert-manager --namespace cert-manager
oc delete project cert-manager

# CRDs MUST be deleted manually for a complete cleanup, as they are not deleted by default when the chart is uninstalled. This is helm chart's behavior.
oc delete crd $(oc get crd | grep cert-manager | awk '{print $1}')
```

## Pause & Resume Development

To "turn off" the cert-manager application without deleting configuration or data, you can scale the replicas to 0.

**To Pause (Stop Pods):**
```bash
oc scale deployment cert-manager --replicas=0 -n cert-manager
```

**To Resume (Start Pods):**
```bash
oc scale deployment cert-manager --replicas=1 -n cert-manager
```
