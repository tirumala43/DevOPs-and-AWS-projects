Deploying MongoDB Replica Set with StatefulSets on Kubernetes (with Authentication)
This document ensures your MongoDB deployment includes authentication from the start, requiring a username and password to connect to the database.

Table of Contents
Prerequisites

Deployment Components

1. Namespace

2. Secret (for MongoDB Credentials - AUTHENTICATION)

3. Headless Service

4. StatefulSet (Authentication Configuration inside)

Option A: Using standard StorageClass

Option B: Using microk8s-hostpath StorageClass (for MicroK8s)

Deployment Steps

Verification and Basic Operations (Connecting with Auth)

Cleanup

Important Considerations

1. Prerequisites
A running Kubernetes cluster (e.g., MicroK8s, Minikube, Kind, GKE, EKS, AKS).

kubectl command-line tool configured to interact with your cluster.

For MicroK8s users targeting microk8s-hostpath StorageClass: Ensure the storage addon is enabled:

Bash

microk8s enable storage
2. Deployment Components
1. Namespace (mongodb-ns.yaml)
YAML

# mongodb-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mongodb-ns
2. Secret (for MongoDB Credentials - AUTHENTICATION) (mongodb-secret.yaml)
This Secret object is where you store your desired MongoDB root username and password. Kubernetes will inject these as environment variables into your MongoDB containers.

Action Required:

Replace <BASE64_ENCODED_USERNAME> and <BASE64_ENCODED_PASSWORD> with your actual desired credentials, encoded in Base64.

How to get Base64 encoded values:
On your terminal, run:

Bash

echo -n 'your_desired_username' | base64
echo -n 'your_desired_password' | base64
Then, copy the output and paste it into the mongodb-secret.yaml file.

YAML

# mongodb-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-credentials # This name is referenced in the StatefulSet
  namespace: mongodb-ns
type: Opaque
data:
  # Base64 encoded values for username and password
  # Example: echo -n 'admin' | base64 -> YWRtaW4=
  # Example: echo -n 'your_strong_password' | base64 -> eW91cl9zdHJvbmdfcGFzc3dvcmQ=
  MONGO_ROOT_USERNAME: <BASE64_ENCODED_USERNAME> # This key will hold your username
  MONGO_ROOT_PASSWORD: <BASE64_ENCODED_PASSWORD> # This key will hold your password
3. Headless Service (mongo-svc.yml)
YAML

# mongo-svc.yml
apiVersion: v1
kind: Service
metadata:
  name: svc-mongodb
  namespace: mongodb-ns
  labels:
    app: mongo-db
spec:
  ports:
    - port: 27017
      targetPort: 27017
  clusterIP: None
  selector:
    app: mongo-db
4. StatefulSet (Authentication Configuration inside) (mongodb-statefulset.yaml)
The crucial part for authentication in the StatefulSet is the env section within the MongoDB container specification. The official MongoDB Docker image uses the MONGO_INITDB_ROOT_USERNAME and MONGO_INITDB_ROOT_PASSWORD environment variables to automatically create a root user with the specified credentials when the container is first initialized on an empty data directory.

Choose ONE of the following options (A or B) for your mongodb-statefulset.yaml file.

Option A: Using standard StorageClass
YAML

# mongodb-statefulset-standard.yaml (Save as mongodb-statefulset.yaml for deployment)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-sfs
  namespace: mongodb-ns
  labels:
    app: mongo-db
spec:
  replicas: 3
  serviceName: svc-mongodb
  selector:
    matchLabels:
      app: mongo-db
  template:
    metadata:
      labels:
        app: mongo-db
    spec:
      initContainers:
        # ... (initContainers remain the same) ...
        - name: init-mongodb
          image: mongo:4.4
          command: ["sh", "-c"]
          args:
            - |
              echo "Initializing MongoDB replica set..."
              until mongo --host mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())" || \
                    mongo --host mongodb-1.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())" || \
                    mongo --host mongodb-2.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())"; do
                echo "Waiting for MongoDB pods to be ready..."
                sleep 5
              done
              echo "MongoDB pods are ready. Configuring replica set if not already configured."
              # Only run rs.initiate() from the first pod if the replica set is not yet configured
              if [ "$(mongo mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "rs.status().ok" | grep -c '0')" -eq 1 ]; then
                echo "Attempting to initiate replica set."
                mongo mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval 'rs.initiate({ _id: "myReplicaSet", members: [ { _id: 0, host: "mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017" }, { _id: 1, host: "mongodb-1.svc-mongodb.mongodb-ns.svc.cluster.local:27017" }, { _id: 2, host: "mongodb-2.svc-mongodb.mongodb-ns.svc.cluster.local:27017" } ]})'
                echo "Replica set initiation command sent. Waiting for replication to stabilize."
                sleep 20 # Give some time for replication to propagate
              else
                echo "Replica set already configured or initiation in progress."
              fi
      containers:
        - name: mongodb-c
          image: mongo:4.4
          ports:
            - containerPort: 27017
          command:
            - mongod
            - "--replSet"
            - "myReplicaSet"
            - "--bind_ip_all"
            - "--dbpath=/data/db"
          env: # <--- AUTHENTICATION: These environment variables are read from the Secret
            - name: MONGO_INITDB_ROOT_USERNAME # MongoDB image uses this to create the root user
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials # References your Secret name
                  key: MONGO_ROOT_USERNAME # References the key within the Secret
            - name: MONGO_INITDB_ROOT_PASSWORD # MongoDB image uses this to set the root password
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials # References your Secret name
                  key: MONGO_ROOT_PASSWORD # References the key within the Secret
          volumeMounts:
            - name: mongodb-persistent-storage
              mountPath: /data/db
          resources:
            requests:
              memory: "250Mi"
              cpu: "200m"
            limits:
              memory: "540Mi"
              cpu: "400m"
      securityContext:
        fsGroup: 1000
  volumeClaimTemplates:
    - metadata:
        name: mongodb-persistent-storage
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: standard # Using the 'standard' StorageClass
        resources:
          requests:
            storage: 1Gi
Option B: Using microk8s-hostpath StorageClass (for MicroK8s)
YAML

# mongodb-statefulset-microk8s.yaml (Save as mongodb-statefulset.yaml for deployment)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-sfs
  namespace: mongodb-ns
  labels:
    app: mongo-db
spec:
  replicas: 3
  serviceName: svc-mongodb
  selector:
    matchLabels:
      app: mongo-db
  template:
    metadata:
      labels:
        app: mongo-db
    spec:
      initContainers:
        # ... (initContainers remain the same) ...
        - name: init-mongodb
          image: mongo:4.4
          command: ["sh", "-c"]
          args:
            - |
              echo "Initializing MongoDB replica set..."
              until mongo --host mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())" || \
                    mongo --host mongodb-1.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())" || \
                    mongo --host mongodb-2.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())"; do
                echo "Waiting for MongoDB pods to be ready..."
                sleep 5
              done
              echo "MongoDB pods are ready. Configuring replica set if not already configured."
              # Only run rs.initiate() from the first pod if the replica set is not yet configured
              if [ "$(mongo mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "rs.status().ok" | grep -c '0')" -eq 1 ]; then
                echo "Attempting to initiate replica set."
                mongo mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval 'rs.initiate({ _id: "myReplicaSet", members: [ { _id: 0, host: "mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017" }, { _id: 1, host: "mongodb-1.svc-mongodb.mongodb-ns.svc.cluster.local:27017" }, { _id: 2, host: "mongodb-2.svc-mongodb.mongodb-ns.svc.cluster.local:27017" } ]})'
                echo "Replica set initiation command sent. Waiting for replication to stabilize."
                sleep 20 # Give some time for replication to propagate
              else
                echo "Replica set already configured or initiation in progress."
              fi
      containers:
        - name: mongodb-c
          image: mongo:4.4
          ports:
            - containerPort: 27017
          command:
            - mongod
            - "--replSet"
            - "myReplicaSet"
            - "--bind_ip_all"
            - "--dbpath=/data/db"
          env: # <--- AUTHENTICATION: These environment variables are read from the Secret
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials
                  key: MONGO_ROOT_USERNAME
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials
                  key: MONGO_ROOT_PASSWORD
          volumeMounts:
            - name: mongodb-persistent-storage
              mountPath: /data/db
          resources:
            requests:
              memory: "250Mi"
              cpu: "200m"
            limits:
              memory: "540Mi"
              cpu: "400m"
      securityContext:
        fsGroup: 1000
  volumeClaimTemplates:
    - metadata:
        name: mongodb-persistent-storage
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: microk8s-hostpath # Using the 'microk8s-hostpath' StorageClass
        resources:
          requests:
            storage: 1Gi
3. Deployment Steps
Follow these steps precisely:

Save the YAML files:

mongodb-ns.yaml

mongodb-secret.yaml

mongo-svc.yml

mongodb-statefulset.yaml (choose either Option A or Option B, and save it as this filename)

Apply the resources in order:

Bash

# 1. Create the Namespace
kubectl apply -f mongodb-ns.yaml

# 2. Create the Secret with your credentials
kubectl apply -f mongodb-secret.yaml

# 3. Create the Headless Service
kubectl apply -f mongo-svc.yml

# 4. Create the StatefulSet (make sure you've chosen and saved your preferred option)
kubectl apply -f mongodb-statefulset.yaml
4. Verification and Basic Operations (Connecting with Auth)
Check Namespace and Services:

Bash

kubectl get namespace mongodb-ns
kubectl get svc -n mongodb-ns
Monitor Pod and PVC Status:

Bash

kubectl get pvc -n mongodb-ns
kubectl get pods -n mongodb-ns -o wide
Wait until all three pods show 1/1 Running.

Connect to MongoDB and Perform Basic Operations (with Authentication):
Once all pods are Running and Ready, you must provide the username and password you set in the mongodb-secret.yaml to connect to MongoDB.

Connect to a pod (e.g., mongodb-sfs-0):

Bash

kubectl exec -it mongodb-sfs-0 -n mongodb-ns -- mongo -u <your_username> -p <your_password> --authenticationDatabase admin
Important:

Replace <your_username> with the clear-text username you chose (e.g., admin).

Replace <your_password> with the clear-text password you chose.

--authenticationDatabase admin is important because the root user is typically created in the admin database.

Inside the MongoDB shell (after successful authentication):

Check replica set status:

JavaScript

rs.status()
Look for one member as PRIMARY and the others as SECONDARY.

Insert data (on the primary):

JavaScript

use mydatabase
db.mycollection.insertOne({ "name": "Kubernetes Test", "value": 123 })
Find data (from any replica):

JavaScript

db.mycollection.find().pretty()
Exit the shell:

JavaScript

exit
5. Cleanup
To remove all deployed resources:

Bash

kubectl delete -f mongodb-statefulset.yaml
kubectl delete -f mongo-svc.yml
kubectl delete -f mongodb-secret.yaml
kubectl delete -f mongodb-ns.yaml
6. Important Considerations
Security (Beyond Root User):

This setup secures the root user. For applications, it's best practice to create specific database users with limited privileges rather than using the root user directly.

Consider Kubernetes Network Policies to restrict which pods can connect to your MongoDB replica set.

For production, implement TLS/SSL for encrypted communication between clients and MongoDB.

Storage: The StorageClass choice is critical for production readiness. microk8s-hostpath is typically for local development only.

Resource Limits: Adjust requests and limits in the StatefulSet resources section based on your actual workload and cluster capacity.

Backup & Restore: Implement a robust strategy for backing up and restoring your MongoDB data.

Monitoring: Set up monitoring and alerting for your MongoDB instances.

MongoDB Operator: For advanced features, automated operations (scaling, backups, upgrades, replica set management), and enterprise-grade deployments, consider using a Kubernetes Operator for MongoDB.Deploying MongoDB Replica Set with StatefulSets on Kubernetes (with Authentication)
This document ensures your MongoDB deployment includes authentication from the start, requiring a username and password to connect to the database.

Table of Contents
Prerequisites

Deployment Components

1. Namespace

2. Secret (for MongoDB Credentials - AUTHENTICATION)

3. Headless Service

4. StatefulSet (Authentication Configuration inside)

Option A: Using standard StorageClass

Option B: Using microk8s-hostpath StorageClass (for MicroK8s)

Deployment Steps

Verification and Basic Operations (Connecting with Auth)

Cleanup

Important Considerations

1. Prerequisites
A running Kubernetes cluster (e.g., MicroK8s, Minikube, Kind, GKE, EKS, AKS).

kubectl command-line tool configured to interact with your cluster.

For MicroK8s users targeting microk8s-hostpath StorageClass: Ensure the storage addon is enabled:

Bash

microk8s enable storage
2. Deployment Components
1. Namespace (mongodb-ns.yaml)
YAML

# mongodb-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mongodb-ns
2. Secret (for MongoDB Credentials - AUTHENTICATION) (mongodb-secret.yaml)
This Secret object is where you store your desired MongoDB root username and password. Kubernetes will inject these as environment variables into your MongoDB containers.

Action Required:

Replace <BASE64_ENCODED_USERNAME> and <BASE64_ENCODED_PASSWORD> with your actual desired credentials, encoded in Base64.

How to get Base64 encoded values:
On your terminal, run:

Bash

echo -n 'your_desired_username' | base64
echo -n 'your_desired_password' | base64
Then, copy the output and paste it into the mongodb-secret.yaml file.

YAML

# mongodb-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-credentials # This name is referenced in the StatefulSet
  namespace: mongodb-ns
type: Opaque
data:
  # Base64 encoded values for username and password
  # Example: echo -n 'admin' | base64 -> YWRtaW4=
  # Example: echo -n 'your_strong_password' | base64 -> eW91cl9zdHJvbmdfcGFzc3dvcmQ=
  MONGO_ROOT_USERNAME: <BASE64_ENCODED_USERNAME> # This key will hold your username
  MONGO_ROOT_PASSWORD: <BASE64_ENCODED_PASSWORD> # This key will hold your password
3. Headless Service (mongo-svc.yml)
YAML

# mongo-svc.yml
apiVersion: v1
kind: Service
metadata:
  name: svc-mongodb
  namespace: mongodb-ns
  labels:
    app: mongo-db
spec:
  ports:
    - port: 27017
      targetPort: 27017
  clusterIP: None
  selector:
    app: mongo-db
4. StatefulSet (Authentication Configuration inside) (mongodb-statefulset.yaml)
The crucial part for authentication in the StatefulSet is the env section within the MongoDB container specification. The official MongoDB Docker image uses the MONGO_INITDB_ROOT_USERNAME and MONGO_INITDB_ROOT_PASSWORD environment variables to automatically create a root user with the specified credentials when the container is first initialized on an empty data directory.

Choose ONE of the following options (A or B) for your mongodb-statefulset.yaml file.

Option A: Using standard StorageClass
YAML

# mongodb-statefulset-standard.yaml (Save as mongodb-statefulset.yaml for deployment)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-sfs
  namespace: mongodb-ns
  labels:
    app: mongo-db
spec:
  replicas: 3
  serviceName: svc-mongodb
  selector:
    matchLabels:
      app: mongo-db
  template:
    metadata:
      labels:
        app: mongo-db
    spec:
      initContainers:
        # ... (initContainers remain the same) ...
        - name: init-mongodb
          image: mongo:4.4
          command: ["sh", "-c"]
          args:
            - |
              echo "Initializing MongoDB replica set..."
              until mongo --host mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())" || \
                    mongo --host mongodb-1.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())" || \
                    mongo --host mongodb-2.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())"; do
                echo "Waiting for MongoDB pods to be ready..."
                sleep 5
              done
              echo "MongoDB pods are ready. Configuring replica set if not already configured."
              # Only run rs.initiate() from the first pod if the replica set is not yet configured
              if [ "$(mongo mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "rs.status().ok" | grep -c '0')" -eq 1 ]; then
                echo "Attempting to initiate replica set."
                mongo mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval 'rs.initiate({ _id: "myReplicaSet", members: [ { _id: 0, host: "mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017" }, { _id: 1, host: "mongodb-1.svc-mongodb.mongodb-ns.svc.cluster.local:27017" }, { _id: 2, host: "mongodb-2.svc-mongodb.mongodb-ns.svc.cluster.local:27017" } ]})'
                echo "Replica set initiation command sent. Waiting for replication to stabilize."
                sleep 20 # Give some time for replication to propagate
              else
                echo "Replica set already configured or initiation in progress."
              fi
      containers:
        - name: mongodb-c
          image: mongo:4.4
          ports:
            - containerPort: 27017
          command:
            - mongod
            - "--replSet"
            - "myReplicaSet"
            - "--bind_ip_all"
            - "--dbpath=/data/db"
          env: # <--- AUTHENTICATION: These environment variables are read from the Secret
            - name: MONGO_INITDB_ROOT_USERNAME # MongoDB image uses this to create the root user
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials # References your Secret name
                  key: MONGO_ROOT_USERNAME # References the key within the Secret
            - name: MONGO_INITDB_ROOT_PASSWORD # MongoDB image uses this to set the root password
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials # References your Secret name
                  key: MONGO_ROOT_PASSWORD # References the key within the Secret
          volumeMounts:
            - name: mongodb-persistent-storage
              mountPath: /data/db
          resources:
            requests:
              memory: "250Mi"
              cpu: "200m"
            limits:
              memory: "540Mi"
              cpu: "400m"
      securityContext:
        fsGroup: 1000
  volumeClaimTemplates:
    - metadata:
        name: mongodb-persistent-storage
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: standard # Using the 'standard' StorageClass
        resources:
          requests:
            storage: 1Gi
Option B: Using microk8s-hostpath StorageClass (for MicroK8s)
YAML

# mongodb-statefulset-microk8s.yaml (Save as mongodb-statefulset.yaml for deployment)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-sfs
  namespace: mongodb-ns
  labels:
    app: mongo-db
spec:
  replicas: 3
  serviceName: svc-mongodb
  selector:
    matchLabels:
      app: mongo-db
  template:
    metadata:
      labels:
        app: mongo-db
    spec:
      initContainers:
        # ... (initContainers remain the same) ...
        - name: init-mongodb
          image: mongo:4.4
          command: ["sh", "-c"]
          args:
            - |
              echo "Initializing MongoDB replica set..."
              until mongo --host mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())" || \
                    mongo --host mongodb-1.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())" || \
                    mongo --host mongodb-2.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())"; do
                echo "Waiting for MongoDB pods to be ready..."
                sleep 5
              done
              echo "MongoDB pods are ready. Configuring replica set if not already configured."
              # Only run rs.initiate() from the first pod if the replica set is not yet configured
              if [ "$(mongo mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "rs.status().ok" | grep -c '0')" -eq 1 ]; then
                echo "Attempting to initiate replica set."
                mongo mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval 'rs.initiate({ _id: "myReplicaSet", members: [ { _id: 0, host: "mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017" }, { _id: 1, host: "mongodb-1.svc-mongodb.mongodb-ns.svc.cluster.local:27017" }, { _id: 2, host: "mongodb-2.svc-mongodb.mongodb-ns.svc.cluster.local:27017" } ]})'
                echo "Replica set initiation command sent. Waiting for replication to stabilize."
                sleep 20 # Give some time for replication to propagate
              else
                echo "Replica set already configured or initiation in progress."
              fi
      containers:
        - name: mongodb-c
          image: mongo:4.4
          ports:
            - containerPort: 27017
          command:
            - mongod
            - "--replSet"
            - "myReplicaSet"
            - "--bind_ip_all"
            - "--dbpath=/data/db"
          env: # <--- AUTHENTICATION: These environment variables are read from the Secret
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials
                  key: MONGO_ROOT_USERNAME
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials
                  key: MONGO_ROOT_PASSWORD
          volumeMounts:
            - name: mongodb-persistent-storage
              mountPath: /data/db
          resources:
            requests:
              memory: "250Mi"
              cpu: "200m"
            limits:
              memory: "540Mi"
              cpu: "400m"
      securityContext:
        fsGroup: 1000
  volumeClaimTemplates:
    - metadata:
        name: mongodb-persistent-storage
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: microk8s-hostpath # Using the 'microk8s-hostpath' StorageClass
        resources:
          requests:
            storage: 1Gi
3. Deployment Steps
Follow these steps precisely:

Save the YAML files:

mongodb-ns.yaml

mongodb-secret.yaml

mongo-svc.yml

mongodb-statefulset.yaml (choose either Option A or Option B, and save it as this filename)

Apply the resources in order:

Bash

# 1. Create the Namespace
kubectl apply -f mongodb-ns.yaml

# 2. Create the Secret with your credentials
kubectl apply -f mongodb-secret.yaml

# 3. Create the Headless Service
kubectl apply -f mongo-svc.yml

# 4. Create the StatefulSet (make sure you've chosen and saved your preferred option)
kubectl apply -f mongodb-statefulset.yaml
4. Verification and Basic Operations (Connecting with Auth)
Check Namespace and Services:

Bash

kubectl get namespace mongodb-ns
kubectl get svc -n mongodb-ns
Monitor Pod and PVC Status:

Bash

kubectl get pvc -n mongodb-ns
kubectl get pods -n mongodb-ns -o wide
Wait until all three pods show 1/1 Running.

Connect to MongoDB and Perform Basic Operations (with Authentication):
Once all pods are Running and Ready, you must provide the username and password you set in the mongodb-secret.yaml to connect to MongoDB.

Connect to a pod (e.g., mongodb-sfs-0):

Bash

kubectl exec -it mongodb-sfs-0 -n mongodb-ns -- mongo -u <your_username> -p <your_password> --authenticationDatabase admin
Important:

Replace <your_username> with the clear-text username you chose (e.g., admin).

Replace <your_password> with the clear-text password you chose.

--authenticationDatabase admin is important because the root user is typically created in the admin database.

Inside the MongoDB shell (after successful authentication):

Check replica set status:

JavaScript

rs.status()
Look for one member as PRIMARY and the others as SECONDARY.

Insert data (on the primary):

JavaScript

use mydatabase
db.mycollection.insertOne({ "name": "Kubernetes Test", "value": 123 })
Find data (from any replica):

JavaScript

db.mycollection.find().pretty()
Exit the shell:

JavaScript

exit
5. Cleanup
To remove all deployed resources:

Bash

kubectl delete -f mongodb-statefulset.yaml
kubectl delete -f mongo-svc.yml
kubectl delete -f mongodb-secret.yaml
kubectl delete -f mongodb-ns.yaml
6. Important Considerations
Security (Beyond Root User):

This setup secures the root user. For applications, it's best practice to create specific database users with limited privileges rather than using the root user directly.

Consider Kubernetes Network Policies to restrict which pods can connect to your MongoDB replica set.

For production, implement TLS/SSL for encrypted communication between clients and MongoDB.

Storage: The StorageClass choice is critical for production readiness. microk8s-hostpath is typically for local development only.

Resource Limits: Adjust requests and limits in the StatefulSet resources section based on your actual workload and cluster capacity.

Backup & Restore: Implement a robust strategy for backing up and restoring your MongoDB data.

Monitoring: Set up monitoring and alerting for your MongoDB instances.

MongoDB Operator: For advanced features, automated operations (scaling, backups, upgrades, replica set management), and enterprise-grade deployments, consider using a Kubernetes Operator for MongoDB.Deploying MongoDB Replica Set with StatefulSets on Kubernetes (with Authentication)
This document ensures your MongoDB deployment includes authentication from the start, requiring a username and password to connect to the database.

Table of Contents
Prerequisites

Deployment Components

1. Namespace

2. Secret (for MongoDB Credentials - AUTHENTICATION)

3. Headless Service

4. StatefulSet (Authentication Configuration inside)

Option A: Using standard StorageClass

Option B: Using microk8s-hostpath StorageClass (for MicroK8s)

Deployment Steps

Verification and Basic Operations (Connecting with Auth)

Cleanup

Important Considerations

1. Prerequisites
A running Kubernetes cluster (e.g., MicroK8s, Minikube, Kind, GKE, EKS, AKS).

kubectl command-line tool configured to interact with your cluster.

For MicroK8s users targeting microk8s-hostpath StorageClass: Ensure the storage addon is enabled:

Bash

microk8s enable storage
2. Deployment Components
1. Namespace (mongodb-ns.yaml)
YAML

# mongodb-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mongodb-ns
2. Secret (for MongoDB Credentials - AUTHENTICATION) (mongodb-secret.yaml)
This Secret object is where you store your desired MongoDB root username and password. Kubernetes will inject these as environment variables into your MongoDB containers.

Action Required:

Replace <BASE64_ENCODED_USERNAME> and <BASE64_ENCODED_PASSWORD> with your actual desired credentials, encoded in Base64.

How to get Base64 encoded values:
On your terminal, run:

Bash

echo -n 'your_desired_username' | base64
echo -n 'your_desired_password' | base64
Then, copy the output and paste it into the mongodb-secret.yaml file.

YAML

# mongodb-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-credentials # This name is referenced in the StatefulSet
  namespace: mongodb-ns
type: Opaque
data:
  # Base64 encoded values for username and password
  # Example: echo -n 'admin' | base64 -> YWRtaW4=
  # Example: echo -n 'your_strong_password' | base64 -> eW91cl9zdHJvbmdfcGFzc3dvcmQ=
  MONGO_ROOT_USERNAME: <BASE64_ENCODED_USERNAME> # This key will hold your username
  MONGO_ROOT_PASSWORD: <BASE64_ENCODED_PASSWORD> # This key will hold your password
3. Headless Service (mongo-svc.yml)
YAML

# mongo-svc.yml
apiVersion: v1
kind: Service
metadata:
  name: svc-mongodb
  namespace: mongodb-ns
  labels:
    app: mongo-db
spec:
  ports:
    - port: 27017
      targetPort: 27017
  clusterIP: None
  selector:
    app: mongo-db
4. StatefulSet (Authentication Configuration inside) (mongodb-statefulset.yaml)
The crucial part for authentication in the StatefulSet is the env section within the MongoDB container specification. The official MongoDB Docker image uses the MONGO_INITDB_ROOT_USERNAME and MONGO_INITDB_ROOT_PASSWORD environment variables to automatically create a root user with the specified credentials when the container is first initialized on an empty data directory.

Choose ONE of the following options (A or B) for your mongodb-statefulset.yaml file.

Option A: Using standard StorageClass
YAML

# mongodb-statefulset-standard.yaml (Save as mongodb-statefulset.yaml for deployment)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-sfs
  namespace: mongodb-ns
  labels:
    app: mongo-db
spec:
  replicas: 3
  serviceName: svc-mongodb
  selector:
    matchLabels:
      app: mongo-db
  template:
    metadata:
      labels:
        app: mongo-db
    spec:
      initContainers:
        # ... (initContainers remain the same) ...
        - name: init-mongodb
          image: mongo:4.4
          command: ["sh", "-c"]
          args:
            - |
              echo "Initializing MongoDB replica set..."
              until mongo --host mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())" || \
                    mongo --host mongodb-1.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())" || \
                    mongo --host mongodb-2.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())"; do
                echo "Waiting for MongoDB pods to be ready..."
                sleep 5
              done
              echo "MongoDB pods are ready. Configuring replica set if not already configured."
              # Only run rs.initiate() from the first pod if the replica set is not yet configured
              if [ "$(mongo mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "rs.status().ok" | grep -c '0')" -eq 1 ]; then
                echo "Attempting to initiate replica set."
                mongo mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval 'rs.initiate({ _id: "myReplicaSet", members: [ { _id: 0, host: "mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017" }, { _id: 1, host: "mongodb-1.svc-mongodb.mongodb-ns.svc.cluster.local:27017" }, { _id: 2, host: "mongodb-2.svc-mongodb.mongodb-ns.svc.cluster.local:27017" } ]})'
                echo "Replica set initiation command sent. Waiting for replication to stabilize."
                sleep 20 # Give some time for replication to propagate
              else
                echo "Replica set already configured or initiation in progress."
              fi
      containers:
        - name: mongodb-c
          image: mongo:4.4
          ports:
            - containerPort: 27017
          command:
            - mongod
            - "--replSet"
            - "myReplicaSet"
            - "--bind_ip_all"
            - "--dbpath=/data/db"
          env: # <--- AUTHENTICATION: These environment variables are read from the Secret
            - name: MONGO_INITDB_ROOT_USERNAME # MongoDB image uses this to create the root user
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials # References your Secret name
                  key: MONGO_ROOT_USERNAME # References the key within the Secret
            - name: MONGO_INITDB_ROOT_PASSWORD # MongoDB image uses this to set the root password
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials # References your Secret name
                  key: MONGO_ROOT_PASSWORD # References the key within the Secret
          volumeMounts:
            - name: mongodb-persistent-storage
              mountPath: /data/db
          resources:
            requests:
              memory: "250Mi"
              cpu: "200m"
            limits:
              memory: "540Mi"
              cpu: "400m"
      securityContext:
        fsGroup: 1000
  volumeClaimTemplates:
    - metadata:
        name: mongodb-persistent-storage
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: standard # Using the 'standard' StorageClass
        resources:
          requests:
            storage: 1Gi
Option B: Using microk8s-hostpath StorageClass (for MicroK8s)
YAML

# mongodb-statefulset-microk8s.yaml (Save as mongodb-statefulset.yaml for deployment)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-sfs
  namespace: mongodb-ns
  labels:
    app: mongo-db
spec:
  replicas: 3
  serviceName: svc-mongodb
  selector:
    matchLabels:
      app: mongo-db
  template:
    metadata:
      labels:
        app: mongo-db
    spec:
      initContainers:
        # ... (initContainers remain the same) ...
        - name: init-mongodb
          image: mongo:4.4
          command: ["sh", "-c"]
          args:
            - |
              echo "Initializing MongoDB replica set..."
              until mongo --host mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())" || \
                    mongo --host mongodb-1.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())" || \
                    mongo --host mongodb-2.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "printjson(db.serverStatus())"; do
                echo "Waiting for MongoDB pods to be ready..."
                sleep 5
              done
              echo "MongoDB pods are ready. Configuring replica set if not already configured."
              # Only run rs.initiate() from the first pod if the replica set is not yet configured
              if [ "$(mongo mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval "rs.status().ok" | grep -c '0')" -eq 1 ]; then
                echo "Attempting to initiate replica set."
                mongo mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017 --eval 'rs.initiate({ _id: "myReplicaSet", members: [ { _id: 0, host: "mongodb-0.svc-mongodb.mongodb-ns.svc.cluster.local:27017" }, { _id: 1, host: "mongodb-1.svc-mongodb.mongodb-ns.svc.cluster.local:27017" }, { _id: 2, host: "mongodb-2.svc-mongodb.mongodb-ns.svc.cluster.local:27017" } ]})'
                echo "Replica set initiation command sent. Waiting for replication to stabilize."
                sleep 20 # Give some time for replication to propagate
              else
                echo "Replica set already configured or initiation in progress."
              fi
      containers:
        - name: mongodb-c
          image: mongo:4.4
          ports:
            - containerPort: 27017
          command:
            - mongod
            - "--replSet"
            - "myReplicaSet"
            - "--bind_ip_all"
            - "--dbpath=/data/db"
          env: # <--- AUTHENTICATION: These environment variables are read from the Secret
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials
                  key: MONGO_ROOT_USERNAME
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials
                  key: MONGO_ROOT_PASSWORD
          volumeMounts:
            - name: mongodb-persistent-storage
              mountPath: /data/db
          resources:
            requests:
              memory: "250Mi"
              cpu: "200m"
            limits:
              memory: "540Mi"
              cpu: "400m"
      securityContext:
        fsGroup: 1000
  volumeClaimTemplates:
    - metadata:
        name: mongodb-persistent-storage
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: microk8s-hostpath # Using the 'microk8s-hostpath' StorageClass
        resources:
          requests:
            storage: 1Gi
3. Deployment Steps
Follow these steps precisely:

Save the YAML files:

mongodb-ns.yaml

mongodb-secret.yaml

mongo-svc.yml

mongodb-statefulset.yaml (choose either Option A or Option B, and save it as this filename)

Apply the resources in order:

Bash

# 1. Create the Namespace
kubectl apply -f mongodb-ns.yaml

# 2. Create the Secret with your credentials
kubectl apply -f mongodb-secret.yaml

# 3. Create the Headless Service
kubectl apply -f mongo-svc.yml

# 4. Create the StatefulSet (make sure you've chosen and saved your preferred option)
kubectl apply -f mongodb-statefulset.yaml
4. Verification and Basic Operations (Connecting with Auth)
Check Namespace and Services:

Bash

kubectl get namespace mongodb-ns
kubectl get svc -n mongodb-ns
Monitor Pod and PVC Status:

Bash

kubectl get pvc -n mongodb-ns
kubectl get pods -n mongodb-ns -o wide
Wait until all three pods show 1/1 Running.

Connect to MongoDB and Perform Basic Operations (with Authentication):
Once all pods are Running and Ready, you must provide the username and password you set in the mongodb-secret.yaml to connect to MongoDB.

Connect to a pod (e.g., mongodb-sfs-0):

Bash

kubectl exec -it mongodb-sfs-0 -n mongodb-ns -- mongo -u <your_username> -p <your_password> --authenticationDatabase admin
Important:

Replace <your_username> with the clear-text username you chose (e.g., admin).

Replace <your_password> with the clear-text password you chose.

--authenticationDatabase admin is important because the root user is typically created in the admin database.

Inside the MongoDB shell (after successful authentication):

Check replica set status:

JavaScript

rs.status()
Look for one member as PRIMARY and the others as SECONDARY.

Insert data (on the primary):

JavaScript

use mydatabase
db.mycollection.insertOne({ "name": "Kubernetes Test", "value": 123 })
Find data (from any replica):

JavaScript

db.mycollection.find().pretty()
Exit the shell:

JavaScript

exit
5. Cleanup
To remove all deployed resources:

Bash

kubectl delete -f mongodb-statefulset.yaml
kubectl delete -f mongo-svc.yml
kubectl delete -f mongodb-secret.yaml
kubectl delete -f mongodb-ns.yaml
6. Important Considerations
Security (Beyond Root User):

This setup secures the root user. For applications, it's best practice to create specific database users with limited privileges rather than using the root user directly.

Consider Kubernetes Network Policies to restrict which pods can connect to your MongoDB replica set.

For production, implement TLS/SSL for encrypted communication between clients and MongoDB.

Storage: The StorageClass choice is critical for production readiness. microk8s-hostpath is typically for local development only.

Resource Limits: Adjust requests and limits in the StatefulSet resources section based on your actual workload and cluster capacity.

Backup & Restore: Implement a robust strategy for backing up and restoring your MongoDB data.

Monitoring: Set up monitoring and alerting for your MongoDB instances.

MongoDB Operator: For advanced features, automated operations (scaling, backups, upgrades, replica set management), and enterprise-grade deployments, consider using a Kubernetes Operator for MongoDB.
