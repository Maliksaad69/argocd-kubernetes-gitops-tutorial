# ArgoCD GitOps Tutorial: The Simplest Case (Pure Kubernetes Manifests)

A complete, beginner-friendly guide to setting up **GitOps with ArgoCD** on Kubernetes using raw, plain Kubernetes manifests located in [`gitops/basic`](gitops/basic).

---

## 📖 How It Works

```
                               +-----------------------------+
                               |     GitHub Repository       |
                               |  gitops/basic/ (Manifests)  |
                               +--------------+--------------+
                                              |
                                              | 1. ArgoCD tracks Git (Single Source of Truth)
                                              v
+-----------------------------------------------------------------------------------------+
| Kubernetes Cluster                                                                      |
|                                                                                         |
|   [argocd namespace]                                                                    |
|   +---------------------------------------+                                             |
|   | ArgoCD Controller                     |                                             |
|   | (Application: basic-application.yaml) |                                             |
|   +-------------------+-------------------+                                             |
|                       |                                                                 |
|                       | 2. Auto-syncs / Self-heals                                      |
|                       v                                                                 |
|   [grade-demo namespace]                                                                |
|   +---------------------------------------------------------------------------------+   |
|   |  - Deployment: grade-submission-api (9 Replicas)                                |   |
|   |  - Service:    grade-submission-api (NodePort: 31000 / Port: 3000)              |   |
|   +---------------------------------------------------------------------------------+   |
+-----------------------------------------------------------------------------------------+
```

---

## 🛠️ Prerequisites

Before you begin, ensure you have:

1. **A running Kubernetes cluster**:
   - Local: [Minikube](https://minikube.sigs.k8s.io/), [Kind](https://kind.sigs.k8s.io/), [k3d](https://k3d.io/), or **Docker Desktop** (Kubernetes enabled).
   - Cloud: EKS, GKE, or AKS.
2. **`kubectl`** installed and configured to communicate with your cluster.
3. **`helm`** (v3+) installed.
4. **Git** and a **GitHub account** (fork or push this repository to your own account).

---

## 🚀 Step-by-Step Guide

### Step 1: Install ArgoCD on Your Cluster

Add the official Argo Helm chart repository and install ArgoCD:

```bash
# Add Argo Helm repository and update
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create the dedicated namespace
kubectl create namespace argocd

# Install ArgoCD via Helm
helm install argocd argo/argo-cd --namespace argocd --version 7.7.0
```

Verify that all ArgoCD pods are running:

```bash
kubectl get pods -n argocd
```

*(Wait until all pods show `Running` or `Completed`.)*

---

### Step 2: Access the ArgoCD Web UI

1. **Port-forward the ArgoCD Server**:
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
   Open your browser and navigate to: **[https://localhost:8080](https://localhost:8080)**  
   *(Accept the self-signed TLS warning in your browser).*

2. **Retrieve the `admin` Password**:
   - **On Linux / macOS / Git Bash**:
     ```bash
     kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
     ```
   - **On Windows (PowerShell)**:
     ```powershell
     [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))
     ```

3. **Log in to ArgoCD**:
   - **Username**: `admin`
   - **Password**: *(The string output from the command above)*

---

### Step 3: Configure Your Repository URL

ArgoCD pulls manifests directly from your remote Git repository.

1. Open [`argocd/applications/basic-application.yaml`](argocd/applications/basic-application.yaml).
2. Update `repoURL` with the URL of **your** GitHub repository:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: grade-submission
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR-USERNAME>/argocd-kubernetes-gitops-tutorial.git # <-- UPDATE THIS
    targetRevision: HEAD
    path: gitops/basic
  destination:
    server: https://kubernetes.default.svc
    namespace: grade-demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

3. Commit and push any changes to your GitHub repository:
   ```bash
   git add argocd/applications/basic-application.yaml
   git commit -m "Configure personal repository URL"
   git push origin main
   ```

---

### Step 4: Deploy the Simplest Case Application

You can deploy the application either declaratively via CLI or through the ArgoCD Web UI.

#### Option A: Declarative CLI (Recommended)

Apply the Application Custom Resource:

```bash
kubectl apply -f argocd/applications/basic-application.yaml
```

ArgoCD will automatically:
- Create the target namespace `grade-demo`.
- Fetch the manifests from `gitops/basic/`.
- Deploy the `grade-submission-api` Deployment (9 replicas) and Service.

#### Option B: Via ArgoCD Web UI

1. Click **+ NEW APP** in the top left.
2. Fill in the fields:
   - **Application Name**: `grade-submission`
   - **Project Name**: `default`
   - **Sync Policy**: `Automatic` (check `Prune Resources` and `Self Heal`)
   - **Repository URL**: `https://github.com/<YOUR-USERNAME>/argocd-kubernetes-gitops-tutorial.git`
   - **Revision**: `HEAD` (or `main`)
   - **Path**: `gitops/basic`
   - **Cluster URL**: `https://kubernetes.default.svc`
   - **Namespace**: `grade-demo`
   - **Sync Options**: Check `Auto-Create Namespace`
3. Click **CREATE**.

---

### Step 5: Verify & Test the Deployed API

1. **Check the application pods in Kubernetes**:
   ```bash
   kubectl get pods -n grade-demo
   ```
   You should see 9 pods running.

2. **Port-forward the Grade Submission Service**:
   ```bash
   kubectl port-forward svc/grade-submission-api -n grade-demo 3000:3000
   ```

3. **Test the API using `curl`**:

   - **Add grades**:
     ```bash
     curl -X POST http://localhost:3000/grades \
       -H "Content-Type: application/json" \
       -d '{"name": "Harry", "subject": "Defense Against Dark Arts", "score": 95}'

     curl -X POST http://localhost:3000/grades \
       -H "Content-Type: application/json" \
       -d '{"name": "Ron", "subject": "Charms", "score": 82}'

     curl -X POST http://localhost:3000/grades \
       -H "Content-Type: application/json" \
       -d '{"name": "Hermione", "subject": "Potions", "score": 98}'
     ```

   - **Retrieve all grades**:
     ```bash
     curl http://localhost:3000/grades
     ```

---

### Step 6: See GitOps in Action

#### 1. Self-Healing (Cluster Drift Correction)
Manually tamper with the cluster by scaling down the deployment to 1 replica:
```bash
kubectl scale deployment grade-submission-api -n grade-demo --replicas=1
```
Watch the pods:
```bash
kubectl get pods -n grade-demo -w
```
**What happens?** ArgoCD detects drift between the cluster and Git. Within seconds, it self-heals by scaling the deployment back to **9 replicas** as declared in Git.

#### 2. Automated Sync (Git as Single Source of Truth)
1. Open [`gitops/basic/grade-submission-api-deployment.yaml`](gitops/basic/grade-submission-api-deployment.yaml).
2. Change `replicas: 9` to `replicas: 3`.
3. Commit and push:
   ```bash
   git add gitops/basic/grade-submission-api-deployment.yaml
   git commit -m "Scale down to 3 replicas"
   git push origin main
   ```
4. Check ArgoCD UI or run `kubectl get pods -n grade-demo`. ArgoCD detects the new commit and automatically terminates 6 pods to match the Git declaration.

---

### Step 7: Clean Up

To delete the application and automatically clean up all associated Kubernetes resources created in `grade-demo`:

```bash
kubectl delete -f argocd/applications/basic-application.yaml
```

To uninstall ArgoCD:
```bash
helm uninstall argocd -n argocd
kubectl delete namespace argocd
kubectl delete namespace grade-demo
```

---

## 📂 Other Deployment Patterns in this Repo

Once you are comfortable with the basic case, explore the other folders in this repository:
- [`helm/grade-submission/`](helm/grade-submission): Package and manage the app using Helm templates and `values.yaml` (configured via [`argocd/applications/helm-application.yaml`](argocd/applications/helm-application.yaml)).
- [`kustomize/`](kustomize): Manage multi-environment configurations (`dev` vs `prod` overlays) without duplicating manifests (configured via [`argocd/applications/kustomize-application.yaml`](argocd/applications/kustomize-application.yaml)).
