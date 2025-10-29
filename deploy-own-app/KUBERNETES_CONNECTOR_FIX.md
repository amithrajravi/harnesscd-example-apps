# Kubernetes Connector Connection Error Fix

## Error: Failed to connect to localhost:8080

This happens because the Docker delegate doesn't have access to your Kubernetes cluster's API server.

## Quick Fix Steps

### Step 1: Verify Your Delegate Name/Selector

1. Go to Harness UI → **Settings → Delegates**
2. Find your Docker delegate and check its **Name** and **Tags**
3. Note the exact name (it might not be `docker-delegate`)

### Step 2: Update Delegate Selector in Connector

Update `kubernetes-connector.yml` with your actual delegate name/tag.

### Step 3: Choose One of These Solutions

---

## Solution A: Install Delegate Inside Kubernetes (Recommended)

If you have a Kubernetes cluster, install the delegate as a pod inside it:

1. **In Harness UI:**
   - Go to **Settings → Delegates → Install Delegate**
   - Select **Kubernetes**
   - Download the Kubernetes YAML manifest

2. **Apply to your cluster:**
   ```bash
   kubectl apply -f harness-delegate.yaml
   ```

3. **Verify delegate is running:**
   ```bash
   kubectl get pods -n harness-delegate-ng
   ```

4. **Note the delegate selector** from the UI (usually like `k8s-delegate` or similar)

5. **Update connector YAML** with the new selector:
   ```yaml
   delegateSelectors:
     - your-k8s-delegate-selector
   ```

---

## Solution B: Configure Docker Delegate with Kubeconfig

If you want to keep using Docker delegate:

1. **Get your kubeconfig:**
   ```bash
   # On a machine that can access your cluster
   kubectl config view --flatten > kubeconfig.yaml
   ```

2. **Stop current delegate:**
   ```bash
   docker stop harness-delegate  # or your delegate container name
   docker rm harness-delegate
   ```

3. **Run delegate with kube柔软-mounted:**
   ```bash
   docker run -d --name harness-delegate \
     -e DELEGATE_NAME=docker-delegate \
     -e DELEGATE_TOKEN=<your-delegate-token> \
     -e MANAGER_HOST_AND_PORT=https://app.harness.io \
     -v $(pwd)/kubeconfig.yaml:/opt/harness-delegate/.kube/config:ro \
     harness/delegate:latest
   ```

   **Important:** Mount the kubeconfig to `/opt/harness-delegate/.kube/config`

4. **Restart delegate and test connection**

---

## Solution C: Use Explicit Cluster Credentials (For Remote Clusters)

If your cluster is remote or you have explicit credentials:

1. **Get cluster master URL:**
   ```bash
   kubectl cluster-info
   # Note the Kubernetes master URL (e.g., https://xxx.xxx.xxx.xxx:6443)
   ```

2. **Create a Service Account Token:**
   ```bash
   # Create service account
   kubectl create serviceaccount harness-delegate-sa - from default
   
   # Create cluster role binding
   kubectl create clusterrolebinding harness-delegate-binding \
     --clusterrole=cluster-admin \
     --serviceaccount=default:harness-delegate-sa
   
   # Get token
   kubectl get secret $(kubectl get sa harness-delegate-sa -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 -d
   ```

3. **Create secret in Harness:**
   - Go to **Settings → Secrets**
   - Create a **Secret Text** with the token value
   - Name it: `k8s-service-account-token`

4. **Update connector YAML:**
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
           serviceAccountTokenRef: k8s-service-account-token
       delegateSelectors:
         - docker-delegate  # or your delegate name
   ```

---

## Solution D: For Minikube/Local Development

If using Minikube:

1. **Get minikube IP:**
   ```bash
   minikube ip
   ```

2. **Get cluster certificate:**
   ```bash
   # Get CA cert
   cat ~/.minikube/ca.crt | base64 -w 0
   
   # Get client cert
   cat ~/.minikube/profiles/minikube/client.crt | base64 -w 0
   
   # Get client key
   cat ~/.minikube/profiles/minikube/client.key | base64 -w 0
   ```

3. **Create secrets in Harness for CA cert, client cert, and client key**

4. **Update connector to use ClientKeyCert method** (see Solution C format)

---

## Quick Verification

After applying any solution:

1. **Test connection in Harness UI:**
   - Go to Connectors → Your K8s Connector
   - Click **Test**

2. **Or verify from delegate:**
   ```bash
   # If delegate is running as Docker
   docker exec -it harness-delegate bash
   kubectl get nodes  # Should list your cluster nodes
   ```

---

## Common Issues

- **Delegate selector mismatch:** Name must match exactly (case-sensitive)
- **Delegate not connected:** Check delegate status in Harness UI → Delegates
- **Network access:** Delegate must be able to reach cluster API server
- **Permissions:** Service account needs appropriate RBAC permissions

