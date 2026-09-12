# Exercise 3: Scaling a Flask App on a Single Node using ReplicaSets

## Problem Statement

During a flash sale (e.g., Big Billion Days or Prime Day), traffic to a simple Flask service can spike from 100 to 10,000 requests per minute. A single Pod would crash under that load. Use a Kubernetes **ReplicaSet** to run multiple identical Pods of the app, scale them up and down, and observe how Kubernetes keeps the desired number of Pods running.

**Objectives:**
- Understand ReplicaSets and Pods
- Scale the Flask app
- Observe Pod distribution and self-healing

**Environment:** Windows, VS Code integrated terminal (Command Prompt), Minikube (single node, Docker driver)

---

## Solution (Brief)

1. Start a single-node Minikube cluster.
2. Create the Flash Sale Flask app (`app.py`) and a `Dockerfile` for it.
3. Define a ReplicaSet (3 replicas) and a ClusterIP Service in `flashsale-replicaset.yaml`, then apply it.
4. Build the image inside Minikube's Docker daemon so the cluster can use it locally.
5. Scale the ReplicaSet from 3 to 5 replicas.
6. Delete a Pod and observe the ReplicaSet creating a replacement.
7. Check Pod distribution across nodes.

---

## Files Created

**flashsale.py**

```python
from flask import Flask, request
import socket, time, random

app = Flask(__name__)

@app.get("/")
def homepage():
    return {
        "message": "Welcome to Big Sale!",
        "pod": socket.gethostname(),
        "ts": time.time()
    }

@app.get("/buy")
def buy():
    # simulate a flash sale checkout
    item = random.choice(["Smartphone", "Shoes", "Headphones", "Laptop"])
    user = request.args.get("user", f"user{random.randint(1,1000)}")
    return {
        "status": "success",
        "item": item,
        "user": user,
        "served_by_pod": socket.gethostname(),
        "time": time.strftime("%H:%M:%S")
    }

@app.get("/health")
def health():
    return {"status": "healthy", "pod": socket.gethostname()}
```

**Dockerfile**

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY ex3-flash-sale.py .
RUN pip install --no-cache-dir flask gunicorn
CMD ["gunicorn","-b","0.0.0.0:5000","app:app","--workers","1","--threads","2"]
```

**flashsale-replicaset.yaml**

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: flashsale-rs
  labels:
    app: flashsale
spec:
  replicas: 3  ## Number of replicas (copies of flashsale app) to run
  selector:
    matchLabels:
      app: flashsale
  template:
    metadata:
      labels:
        app: flashsale
    spec:
      containers:
      - name: flashsale-container
        image: flashsale:1.0
        ports:
        - containerPort: 5000
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 2
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 10
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: flashsale-svc
spec:
  selector:
    app: flashsale
  ports:
  - name: http
    port: 80
    targetPort: 5000
  type: ClusterIP
```

---

## Commands Executed and Output

### Step 1: Start Minikube with a single node

```cmd
minikube start --nodes=1
```

Cluster started successfully (output omitted for brevity).

### Step 2: Check the nodes

```cmd
kubectl get nodes
```

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   41s   v1.35.1
```

### Step 3: Apply the ReplicaSet and Service configuration

```cmd
kubectl apply -f flashsale-replicaset.yaml
```

```
replicaset.apps/flashsale-rs created
service/flashsale-svc created
```

### Step 4: Point the Docker CLI to Minikube's Docker daemon

```cmd
@FOR /f "tokens=*" %i IN ('minikube -p minikube docker-env --shell cmd') DO @%i
```

### Step 5: Build the Docker image

```cmd
docker build -t flashsale:1.0 .
```

```
[+] Building 2.1s (10/10) FINISHED   docker:default
```

### Step 6: Verify the Pods and the ReplicaSet

```cmd
kubectl get pods
```

```
NAME                 READY   STATUS    RESTARTS   AGE
flashsale-rs-9jkkn   1/1     Running   0          11m
flashsale-rs-f9m2t   1/1     Running   0          10m
flashsale-rs-xbscs   1/1     Running   0          14m
```

```cmd
kubectl get rs
```

```
NAME           DESIRED   CURRENT   READY   AGE
flashsale-rs   3         3         3       14m
```


### Step 7: Scale the ReplicaSet to 5 replicas

```cmd
kubectl scale rs flashsale-rs --replicas=5
```

```
replicaset.apps/flashsale-rs scaled
```

### Step 8: Verify the updated ReplicaSet

```cmd
kubectl get rs
```

```
NAME           DESIRED   CURRENT   READY   AGE
flashsale-rs   5         5         5       14m
```


### Step 9: Verify the updated Pods

```cmd
kubectl get pods
```

```
NAME                 READY   STATUS    RESTARTS   AGE
flashsale-rs-9jkkn   1/1     Running   0          14m
flashsale-rs-f9m2t   1/1     Running   0          13m
flashsale-rs-g27kl   1/1     Running   0          17m
flashsale-rs-lf9l7   1/1     Running   0          17m
flashsale-rs-xbscs   1/1     Running   0          17m
```

### Step 10: Delete one Pod

```cmd
kubectl delete pod flashsale-rs-g27kl
```

```
pod "flashsale-rs-g27kl" deleted from default namespace
```

### Step 11: Verify that the ReplicaSet created a replacement Pod

```cmd
kubectl get pods
```

```
NAME                 READY   STATUS    RESTARTS   AGE
flashsale-rs-9jkkn   1/1     Running   0          18m
flashsale-rs-f9m2t   1/1     Running   0          17m
flashsale-rs-lf9l7   1/1     Running   0          20m
flashsale-rs-xbscs   1/1     Running   0          20m
flashsale-rs-xjhsw   1/1     Running   0          2m40s
```

The deleted Pod `flashsale-rs-g27kl` was replaced by a new Pod `flashsale-rs-xjhsw`, so the ReplicaSet kept 5 Pods as desired.

---

## Result

A ReplicaSet with 3 replicas was created and scaled to 5 using `kubectl scale`. Deleting a Pod caused the ReplicaSet to create a replacement automatically, keeping the desired count. 

---

## Key Observations

- **Pod distribution:** each Pod is an identical clone of the app.
- **Resiliency:** if a Pod fails or is deleted, the ReplicaSet creates a new one automatically.
- **Efficiency:** add Pods when demand spikes and remove them when demand is low, instead of over-provisioning servers.