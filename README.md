## GitOps Workflow Implementation Using ArgoCD

This repository demonstrates GitOps principles using Argo CD. You’ll install Argo CD via Helm, deploy a simple “hello” app from this repo, then update the app entirely through Git commits.

---

### 📁 Repository Layout

```text
GITOPS-DEMO-1/
├── apps/
│   └── gitops-demo-app/
│       └── application.yaml          # ArgoCD Application resource
├── k8s/
│   └── base/
│       ├── deployment.yaml           # Kubernetes Deployment
│       ├── service.yaml              # Kubernetes Service
│       └── kustomization.yaml        # Kustomize file for resource composition
├── app.js                            # Your Node.js application entrypoint
├── Dockerfile                        # Container definition for your app
└── package.json                      # Node.js dependencies
````

* **argocd/applications/gitops-demo-app.yaml**
  Defines the Argo CD `Application` resource that points to this repo’s `k8s/` folder.
* **k8s/gitops-demo-app.yaml**
  A single file containing your `Deployment` and `Service` manifests for the Node.js demo app.

---

## 🚀 1. Install Argo CD

1. **Add the Argo CD Helm repo & update**

   ```bash
   helm repo add argo https://argoproj.github.io/argo-helm
   helm repo update
   ```

2. **Create the `argocd` namespace**

   ```bash
   kubectl create namespace argocd
   ```

3. **Install Argo CD into that namespace**

   ```bash
   helm install argocd argo/argo-cd \
     --namespace argocd \
     --set server.service.type=LoadBalancer
   ```

   *Note:* Setting `server.service.type=LoadBalancer` lets you expose the Argo CD API/UI via a Kubernetes `LoadBalancer`.

4. **Wait for all Argo CD pods to become `Running`**

   ```bash
   kubectl get pods -n argocd -w
   ```

---

## 🌐 2. Access the Argo CD UI

* **Port-forward (quick local test)**

  ```bash
  kubectl port-forward svc/argocd-server \
    -n argocd 8080:443 &
  ```

  Then browse to:

  > [https://localhost:8080](https://localhost:8080)

---

## 🔐 3. Retrieve Initial Credentials

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

* **Username:** `admin`
* **Password:** *(the string from the command above)*

---

## 📂 4. Deploy the Demo Application

1. **Ensure your repo is up-to-date**

   ```bash
   git clone https://github.com/DineshDhanji/GitOps-Demo.git
   cd GitOps-Demo
   ```

2. **Apply the Application CRD**

   ```bash
   kubectl apply -f argocd/applications/gitops-demo-app.yaml
   ```

3. **In Argo CD UI**, you should now see an Application named `gitops-demo-app`.

   * Click **Sync** (or wait if auto-sync is enabled).
   * Watch it pull the manifests from `k8s/` and create the Deployment + Service.

4. **Verify in Kubernetes**

   ```bash
   kubectl get all -l app=gitops-demo -n default
   ```

---

## ✏️ 5. GitOps in Action

1. **Fork this repo** on GitHub.
2. **Update** `k8s/gitops-demo-app.yaml`:

   ```yaml
   spec:
     replicas: 3    # change this value to 5 (or any other)
   ```
3. **Commit & Push** your change to *your fork*’s `main` branch.
4. **In Argo CD UI**, within 1–2 minutes the Application will re-sync:

   * You’ll see the `Deployment` scale from 3 → 5 pods automatically.
5. **Confirm on the cluster**:

   ```bash
   kubectl get pods -l app=gitops-demo
   ```

*This demonstrates the core GitOps principle: **all changes** to your cluster state flow through Git, and Argo CD continuously reconciles the live state toward the desired state in Git.*

---

## 🧹 6. Cleanup

```bash
# Delete the demo app
kubectl delete -f argocd/applications/gitops-demo-app.yaml

# Uninstall Argo CD
helm uninstall argocd -n argocd
kubectl delete namespace argocd
```

---
Feel free to adjust the replica counts, add a `kustomization.yaml` for overlays, or tie in Helm charts next.
