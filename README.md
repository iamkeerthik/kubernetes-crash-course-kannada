# 🧩 Kubernetes Crash Course in Kannada (2025)

All Kubernetes YAML manifests and examples used in the **Kubernetes Crash Course in Kannada (2025)** on YouTube.

🎥 **Watch the full video:** [YouTube Link](https://youtu.be/FpeYDZzWPaI)

K8s Document: https://kubernetes.io/docs/home/

Medium : https://medium.com/@iamkeerthik

---

## 🗂️ Index
1. [Pod](#1-pod)
2. [Multi-Container Pod](#2-multi-container-pod)
3. [Namespace](#3-namespace)
4. [Deployment & Rollout](#4-deployment--rollout)
5. [StatefulSet](#5-statefulset)
6. [DaemonSet](#6-daemonset)
7. [Jobs & CronJobs](#7-jobs--cronjobs)
8. [Services](#8-services)
9. [Ingress](#9-ingress)
10. [Volumes & PVC](#10-volumes--pvc)
11. [ConfigMap & Secret](#11-configmap--secret)
12. [RBAC](#12-rbac)
13. [Network Policy](#13-network-policy)
14. [Taints & Tolerations](#14-taints--tolerations)
15. [Resource Limits](#15-resource-limits)
16. [Probes](#16-probes)
17. [HPA](#17-hpa)

---

## 1. Pod
A single Nginx Pod example.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```
## 2. Multi-Container Pod

Two containers sharing the same Pod network namespace.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: alpine:latest
          command: ['sh', '-c', 'while true; do echo "logging" >> /opt/logs.txt; sleep 1; done']
          volumeMounts:
            - name: data
              mountPath: /opt
      initContainers:
        - name: logshipper
          image: alpine:latest
          restartPolicy: Always
          command: ['sh', '-c', 'tail -F /opt/logs.txt']
          volumeMounts:
            - name: data
              mountPath: /opt
      volumes:
        - name: data
          emptyDir: {}
```

## 3. Namespace

Isolates resources logically inside a cluster.
```bash
kubectl create ns dev
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```
 The -A or --all-namespaces flag lets you list resources across all namespaces.

 List all pods in all namespaces:
 ```bash
kubectl get pods -A
```
List all deployments in all namespaces:
```bash
kubectl get deployments --all-namespaces
```
get a cluster-wide overview of almost everything running.
```bash
kubectl get all -A
```

## 4. Deployment & Rollout

Purpose: Manage replicas, rolling updates, and rollbacks.

```bash
kubectl create deployment nginx --image=nginx --replicas=2 --dry-run=client -o yaml > deploy.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

## 5. StatefulSet — “Pods with identity and stable storage”

🎯 Goal: Show that each pod has a unique, stable name and its own storage, unlike Deployments.
 ```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
        command: ["/bin/sh"]
        args: ["-c", "echo 'Hello from $(hostname)' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"]
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Mi
```
```bash
kubectl exec -it web-0 -- cat /usr/share/nginx/html/index.html
```

## 4.DaemonSet — “One pod per node”

🎯 Goal: Show that DaemonSet runs exactly one pod on every node, unlike Deployments which scale by replicas.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
spec:
  selector:
    matchLabels:
      app: monitor
  template:
    metadata:
      labels:
        app: monitor
    spec:
      containers:
      - name: monitor
        image: busybox
        command: ["sh", "-c", "echo Running on $(hostname); sleep 3600"]
```
```bash
kubectl get pods -o wide
```

## 6.Job — “Run once and exit successfully”

🎯 Goal: Show a pod that runs a task and completes — no continuous running.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: date-job
spec:
  template:
    spec:
      containers:
      - name: date
        image: busybox
        command: ["sh", "-c", "echo 'Job started'; date; sleep 5; echo 'Job done'"]
      restartPolicy: Never
  backoffLimit: 2
```

```bash
kubectl apply -f job-demo.yaml
kubectl get jobs
kubectl get pods
kubectl logs <job-pod-name>
```

## 7.(Optional) CronJob — “Run job on schedule”

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cron-date-job
spec:
  schedule: "*/1 * * * *"  # every minute
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cron
            image: busybox
            command: ["sh", "-c", "date; echo 'CronJob executed'"]
          restartPolicy: Never
```

## 8. Services

Purpose: Expose pods inside/outside the cluster.

###ClusterIP
```bash
kubectl expose deploy nginx --port=80 --target-port=80 --type=ClusterIP --name nginx-clusterip
```
```yaml
# ClusterIP Service
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```
###NodePort
```bash
kubectl expose deploy nginx --port=80 --target-port=80 --type=NodePort --name nginx-np
kubectl edit svc nginx-np
```
```yaml
# NodePort Service
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
  type: NodePort
```
###Load Balancer
```bash
kubectl expose deploy nginx --port=80 --target-port=80 --type=LoadBalancer --name nginx-lb
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
  labels:
    app: nginx
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80          # Port exposed externally
      targetPort: 80    # Port on the Pod
```
###Ingress
To expose multipath path (path based routing) on Loadbalacer

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wildcard-host
spec:
  rules:
  - host: "foo.bar.com"
    http:
      paths:
      - pathType: Prefix
        path: "/bar"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: "*.foo.com"
    http:
      paths:
      - pathType: Prefix
        path: "/foo"
        backend:
          service:
            name: service2
            port:
              number: 80
```

 ## 9.Volumes
 ```bash
kubectl get storageclass
```
  How dynamic provisioning works in GKE
  
```bash
  nano pvc.yaml
```
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

 ##Reuse an existing GCP Persistent Disk by creating a manual PV, then a PVC that claims it
 Ensure the GCP disk exists
 ```bash
gcloud compute disks create nginx-disk --size=10GB --zone=<region>
```
PV
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
  labels:
    type: fast
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: standard-rwo
  persistentVolumeReclaimPolicy: Retain
  gcePersistentDisk:
    pdName: nginx-disk
    fsType: ext4
```
PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard-rwo
  selector:
    matchLabels:
      type: fast
```
3️⃣ Key points
	1.	storageClassName must still match between PV and PVC.
	2.	Access modes and size must also satisfy the PVC request.
	3.	Label selector is optional, but great for manual control or grouping PVs.
	4.	Only PVs that satisfy all criteria will be considered.


## 10. Secrets and Configmaps

1️⃣ ConfigMap

Purpose:
	•	Store non-sensitive configuration data (like app settings, environment variables, config files).
	•	Decouples configuration from the container image.

```bash
kubectl create cm appconfig --from-literal=port=80
```
    
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "production"
  APP_PORT: "8080"
```

2️⃣ Secret

Purpose:
	•	Store sensitive data (passwords, API keys, certificates).
	•	Data is base64-encoded by Kubernetes.
	•	Can be mounted as files or used as environment variables.

```bash
kubectl create secret generic appscrt --from-literal=uname=user --from-literal=passwd=password
kubectl get secret db-secret -o yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_USER: YWRtaW4=          # base64 of 'admin'
  DB_PASSWORD: cGFzc3dvcmQ=  # base64 of 'password'
```

 Using --from-file

 	•	Reads content from a file.
	•	The key becomes the filename, and the value becomes the file content.
	•	Ideal for large content like config files, certificates, JSON, etc.
  
```bash
kubectl create configmap app-config --from-file=app-settings.conf
kubectl create secret generic db-secret --from-file=db-user.txt --from-file=db-password.txt
```

Mixing literals and files
```bash
kubectl create configmap app-config \
  --from-literal=APP_MODE=production \
  --from-file=app-settings.conf
```

3️⃣ Deployment using ConfigMap & Secret
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: app-config       # Inject all ConfigMap keys as env vars
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_USER          # Inject secret as env var
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config     # ConfigMap mounted as files
        - name: secret-volume
          mountPath: /etc/secrets    # Secret mounted as files
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: secret-volume
        secret:
          secretName: db-secret
```

 Verify Deployment
 ```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl exec -it <pod-name> -- env | grep APP_
kubectl exec -it <pod-name> -- ls /etc/config
kubectl exec -it <pod-name> -- ls /etc/secrets
```

 ##Using Google Secret Manager (via CSI driver)
 Instead of creating a Kubernetes Secret manually, you can mount a secret from Secret Manager directly into your pod.
 
 Secret Manager (GCP) → SecretProviderClass (K8s CSI) → Pod Volume → Container

 Steps:
	1.	Enable Secret Manager & CSI Driver:
  ```bash
  gcloud container clusters update CLUSTER_NAME \
      --enable-secrets-store \
      --location=LOCATION
  ```
	2.	Create secrets in Secret Manager:
  
  ```bash
  echo -n "admin" | gcloud secrets create DB_USER --data-file=-
  echo -n "password" | gcloud secrets create DB_PASSWORD --data-file=-
  ```
	3.	Create a SecretProviderClass in Kubernetes:
  
  ```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: gcp-secrets
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/PROJECT_ID/secrets/DB_USER"
      - resourceName: "projects/PROJECT_ID/secrets/DB_PASSWORD"
```
	4.	Mount secrets in a pod:
  ```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: nginx:latest
    volumeMounts:
    - name: secrets-store
      mountPath: "/etc/secrets"
      readOnly: true
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "gcp-secrets"
```

## 10.RBAC
RBAC in Kubernetes controls who can do what in your cluster.
It’s built on four main objects:

<img width="1366" height="508" alt="image" src="https://github.com/user-attachments/assets/be277731-58c1-4c51-8425-cbabaf0549af" />

Key principle:
	•	Roles define what actions are allowed.
	•	Bindings define who gets those permissions.


2️⃣ Example Scenario

Imagine we have:
	•	An Nginx deployment in namespace default.
	•	A ConfigMap and Secret for configuration.
	•	A service account nginx-sa which should only be able to read ConfigMaps and Secrets, and list pods in default namespace.

3️⃣ Service Account
 ```bash
kubectl create sa nginx-sa
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-sa
  namespace: default
```

Pods can use this service account to apply restricted permissions.


4️⃣ Role (Namespace Scoped)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: nginx-role
rules:
- apiGroups: [""]          # "" = core API group
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]          
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
```
Explanation:
	•	Allows get and list of pods, configmaps, secrets in default namespace.
	•	Cannot modify resources (no create, delete, update).

Validate Permissions
```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:nginx-sa
kubectl auth can-i delete pods --as=system:serviceaccount:default:nginx-sa
kubectl auth can-i get secrets --as=system:serviceaccount:default:nginx-sa
```

5️⃣ RoleBinding (Namespace Scoped)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nginx-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount
  name: nginx-sa
  namespace: default
roleRef:
  kind: Role
  name: nginx-role
  apiGroup: rbac.authorization.k8s.io
```
Explanation:
	•	Binds nginx-role to the nginx-sa service account only in default namespace.
	•	Pods using nginx-sa inherit these permissions.

6️⃣ ClusterRole (Cluster Scoped)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-pod-reader
rules:
- apiGroups: [""]          
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```
Explanation:
	•	Can be used across all namespaces.
	•	Useful for monitoring tools or cluster-wide read-only access.

7️⃣ ClusterRoleBinding (Cluster Scoped)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-read-pods
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-pod-reader
  apiGroup: rbac.authorization.k8s.io
```
Validate Permissions
```bash
kubectl auth can-i list pods --as=jane
kubectl auth can-i list deployments --as=jane

kubectl auth can-i list pods -n kube-system --as=jane
```

🧩 API Groups in Kubernetes

Every Kubernetes resource belongs to an API group — which helps version and organize them.
You can see all available API groups with:

```bash
kubectl api-resources
```
Example output:
```bash
NAME               APIGROUP                       NAMESPACED   KIND
pods                                               true         Pod
deployments        apps                           true         Deployment
roles              rbac.authorization.k8s.io      true         Role
clusterroles       rbac.authorization.k8s.io      false        ClusterRole
```
🔹 Core (legacy) API Group:
Some basic resources (like pods, services, configmaps, nodes) belong to the core API group, which doesn’t have a name.
That’s why when defining permissions in RBAC, you must leave the apiGroups field as empty quotes ("").

Example — giving a user permission to list Pods:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]     # 👈 Core API group (empty string)
  resources: ["pods"]
  verbs: ["get", "list"]
```
Example — for deployments (which belong to apps group):
```yaml
rules:
- apiGroups: ["apps"]  # 👈 apps group
  resources: ["deployments"]
  verbs: ["get", "list"]
```

2️⃣ Steps to Map KSA to GSA

Pod → KSA → GKE maps to GSA → Access GCP APIs

Step 1: Create a GCP Service Account
```bash
gcloud iam service-accounts create nginx-gsa \
    --description="GSA for nginx pods" \
    --display-name="nginx-gsa"
```

Step 2: Grant IAM roles to the GSA
For example, to allow Secret Manager access:
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:nginx-gsa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

Step 3: Create a Kubernetes Service Account (KSA)
```bash
kubectl create serviceaccount nginx-sa --namespace default
```

Step 4: Bind the KSA to the GSA
```bash
gcloud iam service-accounts add-iam-policy-binding nginx-gsa@PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[default/nginx-sa]"
```
Explanation:
	•	PROJECT_ID.svc.id.goog[namespace/ksa] identifies the KSA.
	•	roles/iam.workloadIdentityUser allows the KSA to impersonate the GSA.

Step 5: Annotate the KSA to use the GSA
```bash
kubectl annotate serviceaccount \
  nginx-sa \
  --namespace default \
  iam.gke.io/gcp-service-account=nginx-gsa@PROJECT_ID.iam.gserviceaccount.com
```

Step 6: Use the KSA in a Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: nginx-sa  # <-- Use the KSA here
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

8️⃣ Imperative Commands
```bash

kubectl create role nginx-role \
  --verb=get,list \
  --resource=pods,configmaps,secrets \
  --namespace=default

kubectl create rolebinding nginx-rolebinding \
  --role=nginx-role \
  --serviceaccount=default:nginx-sa \
  --namespace=default

kubectl create clusterrole cluster-pod-reader \
  --verb=get,list,watch \
  --resource=pods

kubectl create clusterrolebinding cluster-read-pods \
  --clusterrole=cluster-pod-reader \
  --user=jane
```

## 11. NetworkPolicy
•	A NetworkPolicy controls network traffic to/from pods.
•	It acts like a firewall at the pod level.
•	By default, in Kubernetes:
•	Pods can talk to each other freely (no restrictions).
•	NetworkPolicy is opt-in; it only restricts traffic if applied.


•	Frontend: allow port 80 from anywhere
•	Backend: allow port 80 only from frontend
1️⃣ Frontend Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    role: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      role: frontend
  template:
    metadata:
      labels:
        role: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```
2️⃣ Backend Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    role: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      role: backend
  template:
    metadata:
      labels:
        role: backend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Expose Both deployment with ClusterIP
```bash
kubectl expose deploy frontend --port=80 --target-port=80 --name=frontend
kubectl expose deploy backend --port=80 --target-port=80 --name=backend
```

3️⃣ NetworkPolicy for Frontend (allow 80 from everywhere)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-allow-80
spec:
  podSelector:
    matchLabels:
      role: frontend
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - protocol: TCP
      port: 80
```

•	Allows port 80 from any pod or IP to frontend.
•	All other ports are blocked.

	
4️⃣ NetworkPolicy for Backend (allow 80 only from frontend)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-from-frontend
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 80
```

•	Only pods labeled role=frontend can access backend on port 80.
•	All other traffic is blocked.


5️⃣ Temp pod to test (imperative)
```bash
# Test access to frontend port 80 (should succeed)
kubectl run temp-frontend --rm -it --image=curlimages/curl:latest --restart=Never --command -- curl http://frontend.default.svc.cluster.local:80

# Test access to backend port 80 (should fail)
kubectl run temp-backend --rm -it --image=curlimages/curl:latest --restart=Never --command -- curl http://backend.default.svc.cluster.local:80

#Test access from frontend pod to backend pod (should succeed)
kubectl exec -it <frontend-pod> -- curl http://backend.default.svc.cluster.local:80

#Test access to backend port 80 with labelled pod ( Bypass)
kubectl run temp-backend --rm -it --image=curlimages/curl:latest --labels=role=frontend --restart=Never --command -- curl http://backend.default.svc.cluster.local:80
```


## 12. Taints and Tolerations

1️⃣ Node Selector in Deployment
```bash
kubectl label nodes <node-name> role=special
```
•	Node selectors let you choose a node based on labels.
•	Example: if a node has label role=special, you can schedule pods only there.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: special-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: special
  template:
    metadata:
      labels:
        app: special
    spec:
      containers:
      - name: nginx
        image: nginx
      nodeSelector:
        role: special
```

2️⃣ Taint + Toleration in Deployment

	
•	Taints are applied to nodes. They mark a node so that pods won’t schedule on it unless they tolerate the taint.
•	Tolerations are applied to pods. They allow a pod to be scheduled on nodes with matching taints.

Think of it as:

Taint = “Do not disturb unless you tolerate me”
Toleration = “I can tolerate that disturbance”

2️⃣ Example Scenario
	•	You have a node that should only run special workloads, e.g., GPU jobs or high-memory pods.
	•	You taint the node so normal pods don’t schedule there.
	•	Special pods add a toleration to allow scheduling.

Taint a node
```bash
kubectl taint nodes <node-name> special=true:NoSchedule
```
•	If the node is tainted, you must add a toleration in the pod spec.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tainted-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tainted
  template:
    metadata:
      labels:
        app: tainted
    spec:
      containers:
      - name: nginx
        image: nginx
      nodeSelector:
        role: special
      tolerations:
      - key: "special"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
```

1️⃣ Taint Effects
NoSchedule
Pods without matching toleration cannot be scheduled on this node. They remain Pending.

PreferNoSchedule
Kubernetes tries to avoid scheduling pods without matching toleration on this node, but doesn’t strictly block them. It’s a “soft” taint.

NoExecute
Pods without matching toleration are evicted if already running, and new pods cannot schedule.


1️⃣ Check taints on a node
```bash
kubectl describe node <node-name> | grep -i taint -B 10 -A 10
```

2️⃣ Remove (untaint) a node
```bash
kubectl taint nodes <node-name> <key>-
Example:
kubectl taint nodes gke-cluster-1-default-pool-4e1aff24-1zxg special-
```

## 13. HPA
Horizontal Pod Autoscaler (HPA) automatically scales the number of pod replicas in a Deployment, ReplicaSet, or StatefulSet based on CPU/memory utilization or custom metrics.

It ensures your app scales out when load increases and in when load drops — all automatically.

🧠 How it works
	•	The metrics-server collects resource metrics (like CPU/memory) from kubelets.
	•	The HPA controller monitors these metrics.
	•	If actual usage exceeds the target threshold, it increases replicas.
	•	If usage falls below the target, it reduces replicas.

⚙️ Pre-requisite

You must have metrics-server installed:
```bash
kubectl get deployment metrics-server -n kube-system
```

If not installed:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```


🧩 Example Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 100m
          limits:
            cpu: 200m
```

⚙️ Create HPA (Declarative)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deploy
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

👉 This means:
	•	HPA will keep CPU usage around 50% of the requested 100m.
	•	If usage rises above that → new pods are created (up to 5).
	•	If usage drops → pods are removed (down to 1).


⚡ Imperative Way
```bash
kubectl autoscale deployment nginx-deploy --cpu-percent=50 --min=1 --max=5
```

## 14. Deployment with Requests + Limits
In Kubernetes, HPA relies on the requests, but you should always specify both requests and limits in your deployment for better control.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

📊 2. Check Node Usage (used vs available)

```bash
kubectl top nodes
```
📦 3. Check Pod Usage
```bash
kubectl top pods -A
```

## 15. Probes

1. Liveness Probe
	•	periodSeconds: How often Kubernetes checks the container’s health.
	•	Example: periodSeconds: 5 → check every 5 seconds.
	•	failureThreshold: How many consecutive failures before Kubernetes considers the pod unhealthy and restarts it.
	•	Example: failureThreshold: 3 + periodSeconds: 5 → pod restarts after 15 seconds of continuous failure.

2. Readiness Probe
	•	periodSeconds: How often Kubernetes checks if the pod is ready to receive traffic.
	•	failureThreshold: Number of consecutive failures before the pod is removed from Service endpoints.
	•	Behavior: Pod is not restarted; it only stops receiving traffic.

4. Startup Probe
	•	periodSeconds: Interval between checks during container startup.
	•	failureThreshold: Number of failures before the pod is killed and restarted.
	•	Behavior: Only runs at startup. Once it succeeds, liveness probe takes over.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-probes
spec:
  replicas: 1  # Single pod for demo
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0          # No extra pod above desired replicas
      maxUnavailable: 1    # Must be >=1 when maxSurge=0; ensures old pod is replaced immediately
      # Note: This strategy is ONLY for probe testing/demo purposes
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        # ------------------------------
        # Startup Probe (demo purpose)
        # ------------------------------
        startupProbe:
          httpGet:
            path: /wrongpath   # Intentionally wrong path to demonstrate failure
            port: 80
          failureThreshold: 10
          periodSeconds: 5
        # ------------------------------
        # Liveness Probe (demo purpose)
        # ------------------------------
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        # ------------------------------
        # Readiness Probe (demo purpose)
        # ------------------------------
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3

```








