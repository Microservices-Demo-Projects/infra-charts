# Stakater Reloader Wrapper Chart

This chart is a wrapper around the official [`stakater/reloader`](https://github.com/stakater/Reloader) chart, configured to work cleanly on **OpenShift** and **standard Kubernetes**.

> [!NOTE]
> **Installation Sequence**: For a complete platform setup, required by the Demo / POC Apps in this GitHub Org. follow this order:
>
> 1. [Cert-Manager](../cert-manager/README.md)  
> 2. [Headlamp](../headlamp/README.md) (Optional for OpenShift)  
> 3. [HashiCorp Vault](../hashicorp-vault/README.md)  
> 4. [External Secrets](../external-secrets/README.md)  
> 5. **Stakater Reloader** (Current)

## Overview

Reloader automatically triggers rolling upgrades of workloads when referenced **ConfigMaps** or **Secrets** change.  

This is especially useful when using:

- External Secrets syncing from Vault  
- ConfigMaps changed during CI/CD  
- TLS certificates rotated automatically by cert-manager

Supports all typical workload types like: **Deployments**, **StatefulSets**, **DaemonSets**, etc.

## Chart Configuration

### OpenShift (Default)

The chart is pre-configured for OpenShift with the necessary `values.yaml` overrides—no additional setup needed.

### Standard Kubernetes

To run on Kubernetes, disable OpenShift mode and enable stricter security:

Update the `values.yaml` as:
```yaml
stakater-reloader:
  reloader:
    isOpenshift: false
    readOnlyRootFileSystem: true
  
  deployment:
    securityContext:
      runAsUser: 65534   # nobody user
    
    containerSecurityContext:
      readOnlyRootFilesystem: true
```

## Installation

### 1. Setup

```bash
cd platform-charts/skater-reloader

# Add repository
helm repo add stakater https://stakater.github.io/stakater-charts
helm repo update

# Download dependencies
helm dependency update .
```

### 2. Deploy

```bash
helm upgrade --install reloader . \
  --namespace reloader \
  --create-namespace
```

## Verification

```bash
# Check pod status
oc get pods -n reloader

# View logs to verify reload events
oc logs -n reloader -l app=reloader-stakater-reloader --tail=50

# Check RBAC resources
oc get clusterrole reloader-stakater-reloader-role
oc get clusterrolebinding reloader-stakater-reloader-role-binding
```

## Usage

Reloader triggers workload restarts based on annotations.

### 1. Reload on ANY ConfigMap/Secret change

```yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"              # Both Secret & ConfigMap
    # secret.reloader.stakater.com/auto: "true"     # Only Secret
    # configmap.reloader.stakater.com/auto: "true"  # Only ConfigMap
```

### 2. Reload only specific ConfigMaps / Secrets

```yaml
metadata:
  annotations:
    secret.reloader.stakater.com/reload: "my-secret"
    configmap.reloader.stakater.com/reload: "my-config"
```

### 3. Advanced Reload Options

Reloader also supports more advanced reload mechanisms beyond basic annotations.

For deeper or more complex reload patterns, refer to the official Stakater Reloader documentation <https://github.com/stakater/Reloader/tree/master>

## Verification (Test Workflow)

This section helps you verify that **Reloader properly triggers rolling updates** when ConfigMaps or Secrets change using a **temporary test namespace**, **test ConfigMap**, **test Vault secret**, and a **test Deployment** that continuously logs values.

### 1. Create a Test Namespace

```bash
oc create namespace test-reloader-demo
```

### 2. Create a Test ConfigMap

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-config
  namespace: test-reloader-demo
data:
  MESSAGE: "Hello from config v1"
EOF
```

### 3. Create a Test Secret via Vault

Assuming Vault is running in the `vault` namespace:

```bash
oc exec -ti vault-0 -n vault -- vault kv put kv/demo-secret username="admin" password="initial123"
```

### 4. Create an ExternalSecret to Sync the Vault Secret

```bash
oc apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: demo-secret
  namespace: test-reloader-demo
spec:
  refreshInterval: 3s
  secretStoreRef:
    name: local-vault-backend
    kind: ClusterSecretStore
  target:
    name: demo-synced-secret
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: kv/demo-secret
      property: username
  - secretKey: password
    remoteRef:
      key: kv/demo-secret
      property: password
EOF
```

### 5. Create a Test Deployment (3 Replicas) That Logs Values Every Few Seconds

```bash
oc apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reloader-test-app
  namespace: test-reloader-demo
  annotations:
    reloader.stakater.com/auto: "true"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: reloader-test-app
  template:
    metadata:
      labels:
        app: reloader-test-app
    spec:
      containers:
      - name: app
        image: docker.io/library/busybox:1.37.0
        command:
        - /bin/sh
        - -c
        - |
          while true; do
            echo "Config MESSAGE=\$MESSAGE"
            echo "Secret USERNAME=\$(cat /mnt/secret/username)"
            echo "Secret PASSWORD=\$(cat /mnt/secret/password)"
            echo "Timestamp=\$(date)"
            echo "-----"
            sleep 5
          done
        env:
        - name: MESSAGE
          valueFrom:
            configMapKeyRef:
              name: demo-config
              key: MESSAGE
        volumeMounts:
        - name: secret-vol
          mountPath: /mnt/secret
          readOnly: true
      volumes:
      - name: secret-vol
        secret:
          secretName: demo-synced-secret
EOF
```

### 6. Confirm Pods Are Running

```bash
oc get pods -n test-reloader-demo

oc get events -n test-reloader-demo
```

### 7. View Logs (Observe Initial Values)

```bash
oc logs -n test-reloader-demo -l app=reloader-test-app -f
```

The output will show:

```text
Config MESSAGE=Hello from config v1
Secret USERNAME=admin
Secret PASSWORD=initial123
```

### 8. Trigger Reload by Updating ConfigMap

```bash
oc patch configmap demo-config -n test-reloader-demo \
  --type merge -p '{"data":{"MESSAGE":"Hello from config v2"}}'
```

### 9. Trigger Reload by Updating the Vault Secret

```bash
oc exec -ti vault-0 -n vault -- vault kv put kv/demo-secret username="admin2" password="changed456"
```

### 10. Observe Rolling Restart & Updated Log Output

```bash
oc get pods -n test-reloader-demo -w
```

Pods will rotate automatically.  
Logs will now show:

```text
Config MESSAGE=Hello from config v2
Secret USERNAME=admin2
Secret PASSWORD=changed456
```

### 11. Cleanup Test Resources

```bash
oc delete namespace test-reloader-demo
oc exec -ti vault-0 -n vault -- vault kv delete kv/demo-secret
```

## Configuration

> [!NOTE]
> This wrapper chart passes all configuration under `stakater-reloader.reloader.*` to the upstream chart.

To see all available options:

```bash
helm show values stakater/reloader
```

Update your local `values.yaml` and then:

```bash
helm upgrade reloader . -n reloader
```

## Debugging

```bash
# View events
oc get events -n reloader --sort-by=.lastTimestamp

# Describe reloader pod
oc describe pod -n reloader -l app=reloader

# View logs
oc logs -n reloader -l app=reloader --tail=100
```

If workloads are not reloading:

1. Ensure annotations are correct  
2. Ensure Reloader has RBAC access  
3. Check for errors in logs  

## Cleanup (Uninstall)

```bash
helm uninstall reloader --namespace reloader
```

(Optional):

```bash
oc delete namespace reloader
```

## Pause & Resume Development

To temporarily disable reloader without uninstalling:

**Pause:**

```bash
oc scale deployment reloader --replicas=0 -n reloader
```

**Resume:**

```bash
oc scale deployment reloader --replicas=1 -n reloader
```

## Next Steps

Ensure secrets are synced correctly if using External Secrets.  
Ensure certificates or secrets update as expected so Reloader can trigger reloads if using Vault.