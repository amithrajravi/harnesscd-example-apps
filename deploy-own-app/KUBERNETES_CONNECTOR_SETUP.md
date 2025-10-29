# Kubernetes Connector Setup Guide

## Problem: Failed to connect to localhost:8080

This error occurs when the Harness delegate cannot access your Kubernetes cluster API server.

## Solution Options

### Option 1: Configure Delegate with Kubeconfig (Recommended for Local/Remote Clusters)

If your delegate is running outside the cluster or needs to connect to a remote cluster:

1. **Get your kubeconfig file:**
   ```bash
   # On your local machine with kubectl access
   kubectl config view --flatten > kubeconfig.yaml
   ```

2. **Update the delegate to use kubeconfig:**
   
   **If using Docker:**
   ```bash
   # Stop the current delegate
   docker stop <delegate-container-name>
   
   # Run delegate with kubeconfig mounted
   docker run -d --name harness-delegate \
     -e DELEGATE_NAME=docker-delegate \
     -e DELEGATE_TOKEN=<your-delegate-token> \
     -e MANAGER_HOST_AND_PORT=https://app.harness.io \
     -e DELEGATE_PROFILE=KUBERNETES_CLUSTER \
     -v /path/to/kubeconfig.yaml:/opt/harness-delegate/kubeconfig:ro \
     harness/delegate:latest
   ```
   
   **Or set environment variable:**
   ```bash
   docker run -d --name harness-delegate \
     -e DELEGATE_NAME=docker-delegate \
     -e DELEGATE_TOKEN=<your-delegate-token> \
     -e MANAGER_HOST_AND_PORT=https://app.harness.io \
     -e KUBECONFIG=/opt/harness-delegate/kubeconfig \
     -v /path/to/kubeconfig.yaml:/opt/harness-delegate/kubeconfig:ro \
     harness/delegate:latest
   ```

3. **Restart the delegate and test connection**

### Option 2: Install Delegate Inside Kubernetes Cluster (Best Practice)

If you have a Kubernetes cluster, install the delegate as a pod inside it:

1. Go to Harness UI → Settings → Delegates → Install Delegate
2. Choose "Kubernetes" installation
3. Follow the instructions to create the delegate in your cluster
4. Update the connector YAML to use the new delegate selector

### Option 3: Use Explicit Cluster Credentials

Instead of `InheritFromDelegate`, configure explicit credentials in the connector:

**For Service Account Token:**
```yaml
connector:
  name: ownapp_k8sconnector
  identifier: ownappk8sconnector
  type: K8sCluster
  spec:
    credential:
      type: ServiceAccount
      spec:
        masterUrl: https://your-cluster-api-server:6443
        serviceAccountTokenRef: your-k8s-service-account-token-secret
    delegateSelectors:
      - docker-delegateत्य
```

**For Username/Password:**
```yaml
connector:
  name: ownapp_k8sconnector
  identifier: ownappk8sconnector
  type: K8sCluster
  spec:
    credential:
      type: ManualConfig
      spec:
        masterUrl: https://your-cluster-api-server:6443
        username: <username>
        passwordRef: <password-secret-id>
    delegateSelectors:
      - docker-delegate
```

**For Client Key Certificate:**
```yaml
connector:
  name: ownapp_k8sconnector
  identifier: ownappk8sconnector
  type: K8sCluster
  spec:
    credential:
      type: ClientKeyCert
      spec:
        masterUrl: https://your-cluster-api-server:6443
        caCertRef: <ca-cert-secret-id>
        clientCertRef: <client-cert-secret-id>
        clientKeyRef: <client-key-secret-id>
        clientKeyPassphraseRef: <optional-passphrase-secret-id>
        clientKeyAlgo: RSA
    delegateSelectors:
      - docker-delegate
```

### Option 4: For Minikube/Local Clusters

If using Minikube or local Kubernetes:

1. **Get minikube IP and port:**
   ```bash
   kubectl cluster-info
   ```

2. **Update connector with explicit masterUrl:**
   - See Option 3 above
   - Use the cluster URL from `kubectl cluster-info`
   - Use service account token or client certificate credentials

### Quick Fix Checklist

- [ ] Verify delegate selector name matches your actual delegate (`docker-delegate`)
- [ ] Check if delegate can access cluster: `kubectl get nodes` from delegate container
- [ ] Verify cluster API server is accessible from delegate network
- [ ] Check delegate logs: `docker logs <delegate-container-name>`
- [ ] If using local cluster (Minikube/Docker Desktop), use explicit credentials instead of InheritFromDelegate

## Finding Your Cluster API Server URL

```bash
# Get cluster API server URL
kubectl cluster-info

# Example output:
# Kubernetes control plane is running at https://192.168.49.2:8443
```

Use this URL in the `masterUrl` field when using explicit credentials.

