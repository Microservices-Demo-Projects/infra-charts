# Cert-Manager Wrapper Chart

This chart is a wrapper around the official [`jetstack/cert-manager`](https://artifacthub.io/packages/helm/cert-manager/cert-manager) chart, configured to be deployed to both OpenShift and standard Kubernetes environments.

> [!NOTE]
> **Installation Sequence**: For a complete infrastructure setup, required by the Demo / POC Apps in this GitHub Org. follow this order:
>
> 1. **Cert-Manager** (Current)
> 2. [HashiCorp Vault](../hashicorp-vault/README.md)
> 3. [External Secrets](../external-secrets/README.md)

> [!WARNING]
> In below commands I have mostly used `oc` commands. Please use the appropriate `kubectl` commands instead if you are not in OpenShift environment.

## Overview

The `cert-manager` automates certificate management in Kubernetes, issuing certificates from various sources like Let's Encrypt, HashiCorp Vault, or self-signed.

Additionally, this wrapper chart includes ClusterIssuers and CA certificate templates for convenience:

- **ClusterIssuer**: `self-signed` - A self-signed ClusterIssuer
- **ClusterIssuer**: `demo-ca` - A ClusterIssuer that uses a root CA certificate
- **Certificate**: `demo-root-ca` - A root CA certificate that can be used to sign other certificates

These resources are created using Helm hooks to ensure they're installed after cert-manager CRDs are available. They can be disabled by setting `demoClusterRootCA.required` to `false` in the values.yaml file.

## Setup

```bash
# Add repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Download dependencies
helm dependency update .
```

## Installation

```bash
# Install the chart
helm upgrade --install cert-manager . --namespace cert-manager --create-namespace
```

> [!WARNING]
> **Helm Hooks**: The demo CA resources (ClusterIssuers and root CA Certificate) use Helm `post-install` and `post-upgrade` hooks to ensure they're created after cert-manager's CRDs are installed. This prevents the "no matches for kind" error that would occur if these resources were created before the CRDs exist.

## Verification

```bash
# Check pod status
oc get pods -n cert-manager

# Verify CRDs
oc get crd | grep cert-manager

# Wait for cert-manager to be ready
oc wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=120s

# Verify the demo CA resources were created (if enabled)
oc get clusterissuer

# Check root CA certificate status
oc get certificate demo-root-ca -n cert-manager
oc describe certificate demo-root-ca -n cert-manager
oc describe secret demo-root-ca-tls -n cert-manager
```

## Configuration Updates

> [!NOTE]
> **Subchart Nesting**: Since this chart is a **wrapper** around the official [jetstack/cert-manager](https://artifacthub.io/packages/helm/cert-manager/cert-manager) chart. All configurations of the `cert-manager` must be nested under the `cert-manager:` key of `values.yaml`. This is a standard Helm pattern for wrapper charts, where you place any configuration intended for a subchart under a key matching its `dependencies[].name` mentioned in `Chart.yaml` file.
>
> For example: In the official `jetstack/cert-manager` chart, `crds.enabled` is a root-level property. In this wrapper chart, that same property must be at `cert-manager.crds.enabled` so Helm knows to "pass it down" to the `cert-manager` dependency.
>
> If you were to move `crds.enabled` to the top level (outside the `cert-manager:` block), the subchart would ignore it and wouldn't install the CRDs because the value would be outside its scope.

### Customize Configuration

1. **Check Official Docs**: Visit the [Official cert-manager Helm Documentation](https://artifacthub.io/packages/helm/cert-manager/cert-manager?modal=values) to see all available configurable properties.
  1.1 **Discovering configurable properties through CLI** Alternatively, you can also see all available configuration properties that you can override from your terminal by running:

    ```bash
    helm show values jetstack/cert-manager --version v1.19.2
    ```

2. **Edit `values.yaml`**: Add the settings you need under the `cert-manager` block of this wrapper chart's `values.yaml`.
3. **Apply Changes**: Run the helm upgrade command

    ```bash
    helm upgrade cert-manager . --namespace cert-manager
    ```

## Debugging

If things aren't working as expected, try debugging with the help of below commands:

```bash
# Get events in the namespace
oc get events -n cert-manager --sort-by='.lastTimestamp'

# Check logs of the controller pod
oc logs -n cert-manager -l app.kubernetes.io/instance=cert-manager

# Describe CRDs if they are failing to register
oc describe crd certificates.cert-manager.io

# Check Helm hook resources
oc get clusterissuer,certificate -n cert-manager
```

## Cleanup (Uninstall)

To remove `cert-manager` completely run below commands:

```bash
helm uninstall cert-manager --namespace cert-manager

# Optional: Delete the namespace
oc delete project cert-manager
# or
kubectl delete namespace cert-manager

# WARNING: CRDs are NOT deleted by helm uninstall if "crds.keep" was true in values.yaml
# You must delete them manually if you want a complete cleanup.
oc delete crd $(oc get crd | grep cert-manager | awk '{print $1}')
```

## Next Steps

### Creating a Test Certificate

After installing cert-manager, you can create a test certificate using the demo CA issuer. If the demo issuer is not available (because `demoClusterRootCA.required` was set to false), create your own ClusterIssuer first and update the below example-cert Certificate manifest appropriately.

```bash
# Create a test namespace
oc create project test-certs
# or
kubectl create namespace test-certs

# Switch to the test namespace
oc project test-certs
# or
kubectl config set-context --current --namespace=test-certs

# Create a test certificate using the demo CA
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

# Verify the certificate
oc get certificate example-cert -n test-certs
oc get secret example-cert-tls -n test-certs
oc describe secret example-cert-tls -n test-certs

# Cleanup
oc delete project test-certs
# or
kubectl delete namespace test-certs
```
