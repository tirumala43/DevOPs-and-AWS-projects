# 🛠️ Kubernetes DaemonSets

A **DaemonSet** ensures that a copy of a Pod runs on **all (or some) nodes** in a Kubernetes cluster.

## 📌 Use Cases

- Running **log collection agents** (e.g., Fluentd, Filebeat) on every node.
- Deploying **monitoring agents** (e.g., Prometheus Node Exporter).
- Managing **storage daemons** like `glusterd`, `ceph`, etc.
- Running **security tools** like intrusion detection on each node.

---

## 📄 Sample DaemonSet YAML

Here’s an example that deploys a **Prometheus Node Exporter** DaemonSet:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.3.1
        ports:
        - containerPort: 9100
        resources:
          limits:
            memory: "100Mi"
            cpu: "100m"
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys


🔧 How to Create a DaemonSet
Save the YAML file (e.g., node-exporter-daemonset.yaml)

Apply it using kubectl:
kubectl apply -f node-exporter-daemonset.yaml

🔍 Useful Commands

List all DaemonSets in a namespace:
kubectl get daemonsets -n <namespace>

Describe a specific DaemonSet:
kubectl describe daemonset <name> -n <namespace>

View Pods created by the DaemonSet:
kubectl get pods -o wide -l app=node-exporter -n monitoring

Delete a DaemonSet:
kubectl delete daemonset <name> -n <namespace>


🎯 Scheduling Options

You can control where the DaemonSet runs using:

Node Selectors:

spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd


#Tolerations (to run on tainted nodes like master):

spec:
  template:
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
⚠️ Notes
DaemonSet will automatically add Pods to new nodes when they are added to the cluster.

If you delete a DaemonSet, its Pods are deleted too.