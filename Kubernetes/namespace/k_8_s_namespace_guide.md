# Kubernetes Namespace Guide

Namespaces in Kubernetes provide a way to divide cluster resources between multiple users or applications.

## 📌 Why Use Namespaces?

- **Environment Separation:** dev, staging, prod
- **Multi-tenancy:** isolate teams or projects
- **Resource Quotas:** limit CPU/memory per namespace
- **Access Control:** via Role-Based Access Control (RBAC)

---

## 🔧 Create a Namespace

```bash
kubectl create namespace <namespace-name>
```

**Example:**

```bash
kubectl create namespace dev-env
```

---

## 📋 List All Namespaces

```bash
kubectl get namespaces
```

---

## 🔍 View Resources in a Namespace

```bash
kubectl get all -n <namespace-name>
```

**Example:**

```bash
kubectl get pods -n dev-env
```

---

## ❌ Delete a Namespace

```bash
kubectl delete namespace <namespace-name>
```

---

## ✅ Set a Default Namespace for the Current Context

```bash
kubectl config set-context --current --namespace=<namespace-name>
```

---

## 🚀 Run a Pod in a Namespace

```bash
kubectl run nginx --image=nginx -n <namespace-name>
```

---

## 🧾 Apply YAML to a Namespace

Option 1: CLI

```bash
kubectl apply -f deployment.yaml -n <namespace-name>
```

Option 2: Inside YAML file

```yaml
metadata:
  name: my-app
  namespace: dev-env
```

---

## 📘 Bonus: Check Resource Usage

```bash
kubectl top pod -n <namespace-name>
kubectl top node
```


✅ Correct Command (Fully Compatible)
Use this to set the default namespace in the current context manually:


    kubectl config set-context $(kubectl config current-context) --namespace=<namespace-name>
Example:

    kubectl config set-context $(kubectl config current-context) --namespace=resource-limits

🔧 Update Suggestion for README

I'll update the "Set a Default Namespace for the Current Context" section in your document to include the corrected command.

Shall I go ahead and make that change?


To check the current default namespace used by your kubectl context, run:


kubectl config view --minify --output 'jsonpath={..namespace}'

📝 Output:

If blank → default namespace is default

If set → shows your active namespace (e.g., dev-env, resource-limits, etc.)

✅ Bonus: Get current context with namespace

    kubectl config get-contexts
    
This shows all contexts and their associated namespaces. The current one is marked with *.

Let me know if you want to make this part of your README guide!








---



