# 🧩 Kubernetes Crash Course in Kannada (2025)

All Kubernetes YAML manifests and examples used in the **Kubernetes Crash Course in Kannada (2025)** on YouTube.

🎥 **Watch the full video:** [YouTube Link](https://youtu.be/FpeYDZzWPaI)

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

###Reuse an existing GCP Persistent Disk by creating a manual PV, then a PVC that claims it





