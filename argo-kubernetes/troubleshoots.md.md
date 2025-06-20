# 🚀 GitOps with Argo CD + GitLab on MicroK8s

This project demonstrates a complete GitOps-based CI/CD workflow using **Argo CD**, **GitLab**, and **MicroK8s**.

---

## 🛠️ Troubleshoots & Solutions

### ❌ Error: `namespace from the provided object "argocd" does not match the namespace "dev"`

**Solution:** Use the correct namespace when applying the manifest:

```bash
kubectl apply -f application.yaml -n argocd
```

---

### ❌ Error: `authentication required: HTTP Basic: Access denied`

**Cause:** Repository was private or added with incorrect credentials.

**Solution:**

- Make the GitLab repo public, **or**
- Re-add the repo in Argo CD **without credentials**:
  ```bash
  argocd repo rm https://gitlab.com/your-username/argo-cd.git
  argocd repo add https://gitlab.com/your-username/argo-cd.git
  ```

---

### ❌ Deployment syncs, but Service is missing

**Cause:** `deployment.yaml` and `service.yaml` did not have the correct `namespace`, or `prune` was disabled.

**Solution:**

- Ensure namespace is correctly set in `kustomization.yaml`
- Enable auto-sync and prune in Argo CD application:
  ```yaml
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
  ```

---

### ❌ Error: `spec.ports[0].nodePort: Invalid value: 30123: provided port is already allocated`

**Solution:**

- Use a different NodePort value (between `30000–32767`), or
- Let Kubernetes auto-assign by removing the `nodePort` line.

---

### ❌ GitLab file renamed but Argo CD does not delete the old resource

**Cause:** `prune` is disabled.

**Solution:**

- Manually delete old resource:
  ```bash
  kubectl delete deployment <old-name> -n dev
  ```
- Or enable `prune` in sync policy for automatic cleanup.

---

### ❌ Argo CD UI does not load

**Solution:**

- Check service and ingress configuration.
- Verify you’ve added `argocd.local` in `/etc/hosts` with the correct IP.

---

### ❌ Argo CD doesn’t detect changes after pushing to GitLab

**Solution:**

- Ensure the correct `path` and `targetRevision` are specified in `Application` manifest.
- Trigger a manual sync if auto-sync is disabled.

---

Let me know if you'd like to automate more of these fixes or add GitLab CI/CD triggers!

