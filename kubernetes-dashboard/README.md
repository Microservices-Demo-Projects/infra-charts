# Kubernetes Dashboard Installation

> [!WARNING]
> **OpenShift Users**: OpenShift already includes a built-in web console with dashboard capabilities. This Kubernetes Dashboard installation is **only needed for standard Kubernetes clusters**. If you're running OpenShift, skip this installation and use the OpenShift web console instead.

Simple installation guide for Kubernetes Dashboard with cert-manager integration. We configure the following:

- **Certificates**: The Kong gateway automatically requests certificates from your `demo-ca` ClusterIssuer
- **TLS**: All communication to the dashboard is secured with TLS using your CA
- **Default Settings**: Most settings are kept at their defaults for simplicity

## Prerequisites

Install the [Cert Manager](../cert-manager/README.md) wrapper chart which creates the `demo-ca` ClusterIssuer.

## Installation Steps

### 1. Add the Helm Repository

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm repo update
```

### 2. Install with Custom Values

```bash
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --namespace kubernetes-dashboard \
  --create-namespace \
  --values values.yaml
```

### 3. Verify Installation

```bash
# Check pod status
kubectl get pods -n kubernetes-dashboard

# Verify certificate was created
kubectl get certificate -n kubernetes-dashboard

# Check the certificate details
kubectl describe certificate -n kubernetes-dashboard
```

You should see certificates like:

- `kubernetes-dashboard-kong-admin-cert`
- `kubernetes-dashboard-kong-cluster-cert`
- `kubernetes-dashboard-kong-proxy-cert`

### Verify Certificate Issuer

The certificate should be issued by the `demo-ca` ClusterIssuer we created by installing the [Cert Manager](../cert-manager/README.md) wrapper chart:

```bash
# Check that certificates are issued by demo-ca
kubectl get certificate kubernetes-dashboard-kong-proxy-cert -n kubernetes-dashboard -o yaml
```

Look for the `issuerRef` section - it should show:

```yaml
issuerRef:
  group: cert-manager.io
  kind: ClusterIssuer
  name: demo-ca
```

## Access the Dashboard

### Port Forward (Local Access)

```bash
kubectl port-forward svc/kubernetes-dashboard-kong-proxy 8443:443 -n kubernetes-dashboard
```

Then open: `https://localhost:8443`

### Create an Admin User Token

Create a service account with cluster-admin privileges:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubernetes-dashboard-admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin-user
  namespace: kubernetes-dashboard
---
apiVersion: v1
kind: Secret
metadata:
  name: kubernetes-dashboard-admin-user
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: "kubernetes-dashboard-admin-user"
type: kubernetes.io/service-account-token
EOF
```

### Get the Bearer Token

```bash
kubectl get secret kubernetes-dashboard-admin-user -n kubernetes-dashboard -o jsonpath="{.data.token}" | base64 --decode
```

Copy the token and use it to log in to the dashboard.

## Troubleshooting

If certificates aren't being created:

```bash
# Check cert-manager logs
kubectl logs -n cert-manager -l app.kubernetes.io/instance=cert-manager

# Check certificate status
kubectl describe certificate -n kubernetes-dashboard

# Verify demo-ca ClusterIssuer exists
kubectl get clusterissuer demo-ca
```

## Cleanup

```bash
# Uninstall dashboard
helm uninstall kubernetes-dashboard --namespace kubernetes-dashboard

# Delete namespace
kubectl delete namespace kubernetes-dashboard
```
