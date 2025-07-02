MongoDB Replica Set on Kubernetes
This repository contains Kubernetes YAML configurations to deploy a highly available MongoDB replica set (3 members) using StatefulSets. It includes persistent storage, a dedicated namespace, a headless service, and an init container to automatically initiate the replica set. Authentication is configured from the start.

Table of Contents
Prerequisites

Deployment Guide

Troubleshooting Common Issues

3.1 Pods Stuck in Pending

3.2 Pods Stuck in ContainerCreating or CrashLoopBackOff

3.3 Authentication Issues (Cannot Connect)

3.4 Replica Set Not Forming Correctly

General Troubleshooting Tips

Cleanup

Important Considerations

1. Prerequisites
A running Kubernetes cluster (e.g., MicroK8s, Minikube, Kind, GKE, EKS, AKS).

kubectl command-line tool configured to interact with your cluster.

For MicroK8s users aiming for microk8s-hostpath StorageClass: Ensure the storage addon is enabled:

Bash

microk8s enable storage
2. Deployment Guide
This section assumes you have your YAML files (mongodb-ns.yaml, mongodb-secret.yaml, mongo-svc.yml, mongodb-statefulset.yaml) prepared.

Prepare mongodb-secret.yaml:

Replace <BASE64_ENCODED_USERNAME> and <BASE64_ENCODED_PASSWORD> with your actual desired username and password, Base64 encoded.

Bash

echo -n 'your_desired_username' | base64
echo -n 'your_desired_password' | base64
Choose your mongodb-statefulset.yaml:

Option A (standard StorageClass): Recommended for cloud environments. Ensure your cluster has a standard StorageClass.

Option B (microk8s-hostpath StorageClass): Specifically for MicroK8s setups. Ensure microk8s enable storage has been run.

Apply the resources in order:

Bash

kubectl apply -f mongodb-ns.yaml
kubectl apply -f mongodb-secret.yaml
kubectl apply -f mongo-svc.yml
kubectl apply -f mongodb-statefulset.yaml
Monitor Deployment:

Bash

kubectl get pvc -n mongodb-ns
kubectl get pods -n mongodb-ns -o wide
Wait until all three pods (mongodb-sfs-0, mongodb-sfs-1, mongodb-sfs-2) show 1/1 Running.

3. Troubleshooting Common Issues
When things don't go as planned, kubectl describe and kubectl logs are your best friends.

3.1 Pods Stuck in Pending
Symptoms:

kubectl get pods -n mongodb-ns shows pods with STATUS: Pending.

IP and NODE columns are <none>.

READY column is 0/1.

Diagnosis:
The scheduler cannot find a suitable node to run the pod.

Check Pod Events: This is the most crucial step.

Bash

kubectl describe pod mongodb-sfs-0 -n mongodb-ns # Or any pending pod
Look at the Events section at the bottom.

Common Causes & Solutions:

Persistent Volume Claim (PVC) is Pending:

Event Message often seen: FailedScheduling: 0/X nodes are available: X PersistentVolumeClaim is not bound.

Diagnosis: kubectl get pvc -n mongodb-ns. If your PVC (e.g., mongodb-persistent-storage-mongodb-sfs-0) is Pending.

Solution (for MicroK8s hostpath): This usually means the hostpath provisioner isn't working or wasn't enabled.

Ensure microk8s enable storage was run successfully.

Verify the StorageClass exists: kubectl get sc. Confirm microk8s-hostpath is listed.

Delete and re-apply the StatefulSet to trigger new PVC creation attempts:

Bash

kubectl delete statefulset mongodb-sfs -n mongodb-ns
kubectl apply -f mongodb-statefulset.yaml
Solution (for other environments): Ensure your chosen storageClassName (e.g., standard) is correct and that your cluster has a provisioner for it. Check kubectl get sc to see available StorageClasses.

Insufficient Resources (CPU/Memory):

Event Message often seen: FailedScheduling: 0/X nodes are available: X Insufficient cpu, X Insufficient memory.

Diagnosis: Your cluster nodes don't have enough available CPU or memory to satisfy the pod's requests in the StatefulSet.

Solution:

Increase cluster resources (add more nodes or larger nodes).

Reduce the requests in your mongodb-statefulset.yaml (under resources). Be cautious, as this can affect performance.

No Schedulable Nodes:

Event Message often seen: FailedScheduling: 0/X nodes are available: X node(s) had no matching taints, X node(s) had no available schedule targets.

Diagnosis: kubectl get nodes. No nodes are Ready, or all are tainted/cordoned.

Solution: Ensure your Kubernetes nodes are running and in Ready status. If using Minikube/MicroK8s, ensure the VM is running.

3.2 Pods Stuck in ContainerCreating or CrashLoopBackOff
Symptoms:

kubectl get pods -n mongodb-ns shows pods with STATUS: ContainerCreating or CrashLoopBackOff.

READY column is 0/1.

RESTARTS count might be increasing for CrashLoopBackOff.

Diagnosis:
The pod has been scheduled to a node, but the container either failed to start or exited unexpectedly.

Check Pod Description and Logs: Essential for pinpointing the exact failure.

Bash

kubectl describe pod mongodb-sfs-0 -n mongodb-ns # Or any problematic pod
kubectl logs mongodb-sfs-0 -n mongodb-ns -c mongodb-c # Check logs of the main MongoDB container
kubectl logs mongodb-sfs-0 -n mongodb-ns -c init-mongodb # Check logs of the init container
Look for error messages in the logs and the Events section of describe.

Common Causes & Solutions:

Image Pull Issues:

Logs/Events: ErrImagePull, ImagePullBackOff.

Diagnosis: The cluster cannot pull the mongo:4.4 image.

Solution: Check for typos in the image name. If you're using a private registry, ensure your image pull secret is configured. Check network connectivity from your nodes.

Volume Mount or Permissions Issues:

Logs/Events: Errors like "permission denied", "cannot open data directory", "failed to start up WiredTiger", or similar errors related to /data/db.

Diagnosis: The MongoDB process doesn't have write access to its data directory (/data/db).

Solution: Ensure securityContext.fsGroup: 1000 is correctly indented and applied in your StatefulSet YAML. This usually grants the necessary permissions. If using hostpath, ensure the host directory itself has appropriate permissions (though fsGroup often handles this within the container).

MongoDB Application Crash:

Logs: MongoDB-specific errors indicating it failed to start up. This might happen if:

The --dbpath argument is incorrect (--dbpath=/data/db is correct).

Data corruption (less likely on first startup, but possible on restarts).

Configuration issues (though default settings are usually fine).

Diagnosis: Examine MongoDB logs for messages like Failed to start up WiredTiger, BadValue, Invalid command line parameters.

Solution: Double-check your command and args in the StatefulSet YAML, especially the --dbpath parameter.

Init Container Failure:

Logs: Errors from kubectl logs ... -c init-mongodb.

Diagnosis: The init-mongodb container isn't completing successfully, which prevents the main MongoDB container from starting.

Solution:

Ensure network connectivity between MongoDB pods (mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local, etc.).

Verify the svc-mongodb headless service is correct and running.

Check if mongo client is failing within the init script (e.g., connection refused, which could indicate MongoDB isn't listening yet, or a network issue).

3.3 Authentication Issues (Cannot Connect)
Symptoms:

Pods are Running and Ready.

You try to connect using kubectl exec ... mongo -u <user> -p <pass> ... but get Authentication failed or Error: Authentication failed.

Diagnosis:
The credentials provided to the mongo shell do not match the credentials set up in the MongoDB instance.

Common Causes & Solutions:

Incorrect Credentials in Secret:

Diagnosis: You used the wrong clear-text username/password when generating the Base64 values for mongodb-secret.yaml.

Solution:

Double-check your desired username and password.

Re-generate the Base64 values carefully.

Edit the secret: kubectl edit secret mongodb-credentials -n mongodb-ns. Update the data fields with the correct Base64 values. (Note: editing secrets directly is usually fine for credentials, but often requires pod restart for changes to take effect).

For changes to take effect: Delete and re-create the MongoDB pods: kubectl delete pod -l app=mongo-db -n mongodb-ns. (StatefulSet will recreate them).

Incorrect Base64 Encoding:

Diagnosis: You might have accidentally encoded with extra newlines, spaces, or used an online encoder that adds padding.

Solution: Use echo -n 'your_value' | base64 as precisely demonstrated to avoid issues.

Wrong authenticationDatabase:

Diagnosis: You're trying to authenticate a root user in the wrong database.

Solution: Always use --authenticationDatabase admin when connecting as the root user created by MONGO_INITDB_ROOT_USERNAME/PASSWORD.

MongoDB Not Fully Initialized:

Diagnosis: You're trying to connect before MongoDB has finished its initial setup and user creation.

Solution: Wait a few more minutes after pods show Running and Ready. Check pod logs for messages indicating successful startup and user creation.

3.4 Replica Set Not Forming Correctly
Symptoms:

All pods are Running and Ready.

kubectl exec ... mongo ... rs.status() shows members not in expected states (e.g., UNKNOWN, STARTUP2, multiple PRIMARY (shouldn't happen with StatefulSet), or stuck SECONDARY).

Diagnosis:
The initContainer might not have completed correctly, or there are network issues preventing replica set communication.

Common Causes & Solutions:

Init Container Not Completing:

Diagnosis: kubectl describe pod mongodb-sfs-0 -n mongodb-ns. Look for init-mongodb container status. If not Completed, check its logs: kubectl logs mongodb-sfs-0 -n mongodb-ns -c init-mongodb.

Solution: Troubleshoot the init container's script. Ensure the MongoDB pods are truly reachable by their FQDNs. The sleep 5 and sleep 20 commands in the script are designed to give time for startup.

Network Issues between Pods:

Diagnosis: MongoDB logs will show connection errors between replica set members.

Solution: Verify your CNI (Container Network Interface) plugin is working correctly. Ensure no Kubernetes Network Policies are inadvertently blocking inter-pod communication.

--replSet Name Mismatch:

Diagnosis: The myReplicaSet name in the StatefulSet command and the rs.initiate() call in the init container are different.

Solution: Double-check that myReplicaSet is consistent in both places.

--bind_ip_all Missing:

Diagnosis: MongoDB pods cannot be reached from other pods or the init container.

Solution: Ensure --bind_ip_all is present in the command arguments of your MongoDB container.

4. General Troubleshooting Tips
Always start with kubectl get: Get an overview of your resources (pods, pvc, svc, statefulsets).

Then kubectl describe: Get detailed information and crucial Events that tell you why something is stuck.

Finally kubectl logs: If a container starts but then crashes or acts unexpectedly, check its logs.

Namespace matters: Always include -n mongodb-ns in your commands.

YAML Consistency: Ensure all names (svc-mongodb, mongodb-sfs, mongodb-credentials, mongodb-persistent-storage) and labels (app: mongo-db) are consistent across all your YAML files.

Clean Slate: For persistent issues, sometimes the quickest fix is to delete all related resources and re-apply them. Use the kubectl delete commands from the cleanup section, ensure they are gone, then re-apply.

5. Cleanup
To remove all deployed resources:

Bash

kubectl delete -f mongodb-statefulset.yaml
kubectl delete -f mongo-svc.yml
kubectl delete -f mongodb-secret.yaml
kubectl delete -f mongodb-ns.yaml
6. Important Considerations
Security: This guide configures a basic root user. For production, consider additional MongoDB authentication mechanisms, client certificate authentication, Kubernetes Network Policies, and TLS/SSL for all connections.

StorageClass Choice: hostpath is for development/testing only. Use cloud-provider-specific StorageClasses (gp2, standard, azure-disk etc.) for production environments as they offer higher availability and data durability.

Resource Management: Carefully adjust CPU/Memory requests and limits in the StatefulSet based on your actual workload to prevent resource exhaustion or performance bottlenecks.

Backup Strategy: Implement a robust backup and disaster recovery plan for your MongoDB data.

Monitoring and Alerting: Set up comprehensive monitoring for your MongoDB instances and Kubernetes cluster to detect and respond to issues proactively.

Kubernetes Operators: For complex, production-grade MongoDB deployments, consider using a Kubernetes Operator for MongoDB (e.g., MongoDB Community Operator, Percona Kubernetes Operator) to automate operational tasks.
