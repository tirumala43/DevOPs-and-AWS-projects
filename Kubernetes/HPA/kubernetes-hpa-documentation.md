
# 📘 Kubernetes Horizontal Pod Autoscaler (HPA)

## 🔹 Overview

**Horizontal Pod Autoscaler (HPA)** automatically adjusts the number of pods in a Deployment, ReplicaSet, or StatefulSet based on observed metrics like CPU or memory usage.

It helps ensure applications have enough resources to handle load efficiently while avoiding over-provisioning.

---

## 🔹 How It Works

HPA monitors metrics (like CPU usage) and compares them against defined target thresholds. Based on this, it adjusts the pod count:

- If usage is **above** the target → **scales up**
- If usage is **below** the target → **scales down**

The HPA controller runs a control loop every 15 seconds (by default).

---

## 🔹 Architecture

```
+-----------------------+
|  Metrics Server       |
|  (CPU/Memory Metrics) |
+----------+------------+
           |
           v
+----------------------+
|  Horizontal Pod      |
|  Autoscaler Controller|
+----------+-----------+
           |
           v
+------------------------+
| Deployment/ReplicaSet  |
| (Scaled Resource)      |
+------------------------+
```

---

## 🔹 Prerequisites

- Metrics Server must be installed:
  ```bash
  kubectl top pods
  ```

- Containers must define resource requests and limits:
  ```yaml
  resources:
    requests:
      cpu: 200m
    limits:
      cpu: 500m
  ```

---

## 🔹 HPA Example (YAML)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa
  namespace: hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

---

## 🔹 Useful Commands

| Action                         | Command                                                        |
|-------------------------------|-----------------------------------------------------------------|
| Create HPA                    | `kubectl autoscale deployment <name> --cpu-percent=50 --min=1 --max=5` |
| View HPA                      | `kubectl get hpa`                                              |
| Describe HPA                  | `kubectl describe hpa <name>`                                  |
| View pod metrics              | `kubectl top pods`                                             |

---

## 🔹 Best Practices

- Define CPU/memory **requests and limits** in pod specs.
- Don’t set `minReplicas: 0` unless cold starts are acceptable.
- Monitor metrics regularly with `kubectl top pods`.
- Use **`autoscaling/v2`** for advanced metrics support (e.g., external/custom metrics).

---

## 🔹 Advanced Use Cases

### 🔸 Custom & External Metrics

To scale based on more than CPU (e.g., queue length, requests/sec), use:

- **Prometheus Adapter** for custom metrics
- **KEDA** (Kubernetes Event-driven Autoscaler) for event-based scaling

Supported metric types in `autoscaling/v2`:

```yaml
metrics:
- type: Resource
- type: Pods
- type: Object
- type: External
```

---

## 🔹 Troubleshooting

| Problem                               | Solution                                                  |
|---------------------------------------|-----------------------------------------------------------|
| HPA not scaling                       | Ensure Metrics Server is installed and healthy            |
| Metrics always 0%                     | Check if `resources.requests.cpu` is defined              |
| HPA status shows "Unknown" or "N/A"   | Pod may be initializing or metrics not yet scraped        |

---

## 🔹 Summary

| Feature             | Description                                 |
|---------------------|---------------------------------------------|
| Type                | Horizontal scaling (pods)                   |
| Target Resources    | Deployment, StatefulSet, ReplicaSet         |
| Metrics Supported   | CPU, Memory, Custom, External               |
| API Versions        | `autoscaling/v1`, `autoscaling/v2`          |
| Scaling Controller  | Polls every 15s (default)                   |

---

## 🔹 Resources

- [Kubernetes Official Docs - HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Metrics Server GitHub](https://github.com/kubernetes-sigs/metrics-server)
- [KEDA - Kubernetes Event-driven Autoscaling](https://keda.sh)
