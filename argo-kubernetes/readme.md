# 🚀 GitOps with Argo CD + GitLab on MicroK8s

This project demonstrates a complete GitOps-based CI/CD workflow using **Argo CD**, **GitLab**, and **MicroK8s**.

It enables declarative, automated Kubernetes deployments by syncing manifests stored in GitLab to a local MicroK8s cluster using Argo CD.

---

## 🔩 Stack

- **Argo CD** – GitOps continuous delivery for Kubernetes
- **GitLab** – Git repository for Kubernetes manifests
- **MicroK8s** – Lightweight Kubernetes distribution for local testing
- **Kustomize** – For overlaying environments (dev/stage/prod)

---

## 📁 Folder Structure

```
.
└── Dev/
    ├── deployment.yaml
    ├── service.yaml
    └── kustomization.yaml
```

---

## ⚙️ Prerequisites

- Ubuntu/macOS/WSL with `kubectl` & `snap` installed
- GitLab account
- Argo CD UI access via `http://argocd.local`

---

## 💠 Setup Instructions

### 1. Install MicroK8s and Enable Required Addons

```bash
sudo snap install microk8s --classic
sudo microk8s enable dns ingress storage metallb
sudo microk8s enable metallb:192.168.1.240-192.168.1.250
```

---

### 2. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

### 3. Get Argo CD Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Login to: `http://argocd.local`\
Username: `admin`\
Password: *(from above)*

---

### 4. Create Dev Namespace

```bash
kubectl create namespace dev
```

---

### 5. Argo CD Application Configuration

```yaml
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: eplant-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.com/your-username/argo-cd.git
    targetRevision: HEAD
    path: Dev
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
kubectl apply -f application.yaml
```

---

### 6. Folder Contents

#### `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eplant-shopping
  labels:
    app: eplant
spec:
  replicas: 1
  selector:
    matchLabels:
      app: eplant
  template:
    metadata:
      labels:
        app: eplant
    spec:
      containers:
        - name: eplant
          image: nginx
          ports:
            - containerPort: 80
```

#### `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: eplant-shopping-service
spec:
  selector:
    app: eplant
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

#### `kustomization.yaml`

```yaml
resources:
  - deployment.yaml
  - service.yaml
namespace: dev
```

---

## 🔁 GitOps Workflow

1. Make changes to manifests in GitLab
2. Argo CD detects changes
3. Automatically syncs to Kubernetes
4. Cluster state matches Git state

---

## 🎯 Benefits

- 💡 Declarative infrastructure
- 🔀 Continuous delivery from Git
- 🔍 Drift detection
- ✅ Self-healing deployments

---

## 📜 License

This project is open-source and available for educational or personal use.

---

## 🤝 Contributing

Feel free to fork, improve, or suggest enhancements via pull requests.

---

Let me know if you'd like to add GitLab CI/CD integration or preview environments! 👨‍💻

