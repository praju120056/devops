# Exercise 2: Deploy a Flask App on Minikube using kubectl and YAML

## Problem Statement

Learn Kubernetes basics using Minikube by setting up a single-node cluster and deploying a Python Flask application. The app must be containerized with Docker, deployed using a Kubernetes Deployment YAML, exposed through a Service, and accessed from the host machine.

**Environment:** Windows, VS Code integrated terminal (Command Prompt), Minikube (Docker driver)

---

## Solution (Brief)

1. Start Minikube.
2. Write a simple Flask app (`app.py`) listening on port 15000 and a `Dockerfile` to containerize it.
3. Point the Docker CLI to Minikube's Docker daemon so the image is built inside the cluster's environment.
4. Build the image `flask-app` and deploy it using a Deployment YAML with `imagePullPolicy: Never` (uses the local image instead of pulling from a registry).
5. Verify the Deployment, Pod, and logs.
6. Add a `NodePort` Service to the YAML so the app is reachable from outside the cluster, then access it using `minikube service --url`.

---

## Files Created

**app.py**

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def home():
    return "Hello from Flask on Kubernetes!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=15000)
```

**Dockerfile**

```dockerfile
FROM python:3.8-slim
WORKDIR /app
COPY . /app
RUN pip install flask
CMD ["python", "app.py"]
```

**flask-deployment.yaml** (initial version, Deployment only)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image: flask-app:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 15000
```

---

## Commands Executed and Output

### Step 1: Start Minikube

```cmd
minikube start
```

Cluster started successfully (output omitted for brevity).

### Step 2: Display the Docker environment variables of Minikube

```cmd
minikube docker-env
```

```
SET DOCKER_TLS_VERIFY=1
SET DOCKER_HOST=tcp://127.0.0.1:59856
SET DOCKER_CERT_PATH=C:\Users\prane\.minikube\certs
SET MINIKUBE_ACTIVE_DOCKERD=minikube
REM To point your shell to minikube's docker-daemon, run:
REM @FOR /f "tokens=*" %i IN ('minikube -p minikube docker-env --shell cmd') DO @%i
```

### Step 3: Point the Docker CLI to Minikube's Docker daemon

```cmd
@FOR /f "tokens=*" %i IN ('minikube -p minikube docker-env --shell cmd') DO @%i
```

### Step 4: Build the Docker image

```cmd
docker build -t flask-app .
```

```
[+] Building 28.3s (10/10) FINISHED               docker:default
```

### Step 5: Deploy the application

```cmd
kubectl apply -f flask-deployment.yaml
```

```
deployment.apps/flask-app created
```

### Step 6: Check the Deployment status

```cmd
kubectl get deployments
```

```
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
flask-app   1/1     1            1           6m
```

### Step 7: Verify the Pods created by the Deployment

```cmd
kubectl get pods -l app=flask-app
```

```
NAME                         READY   STATUS    RESTARTS   AGE
flask-app-6d58f88547-dwgkk   1/1     Running   0          6m12s
```

### Step 8: Describe the Deployment

```cmd
kubectl describe deployment flask-app
```

```
Name:                   flask-app
Namespace:              default
CreationTimestamp:      Tue, 13 Oct 2026 20:40:18 +0530
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=flask-app
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=flask-app
  Containers:
   flask-app:
    Image:         flask-app:latest
    Port:          15000/TCP
    Host Port:     0/TCP
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   flask-app-6d58f88547 (1/1 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  6m30s  deployment-controller  Scaled up replica set flask-app-6d58f88547
```

### Step 9: View the Pod logs

```cmd
kubectl logs flask-app-6d58f88547-dwgkk
```

```
 * Serving Flask app 'app'
 * Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:15000
 * Running on http://10.244.0.8:15000
```

Flask is running and listening on port 15000, confirming the app is deployed successfully.

### Step 10: Check Services

```cmd
kubectl get services
```

```
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
hello-k8s    NodePort    10.106.221.130   <none>        80:31836/TCP   52d
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        52d
```

No Service exists yet for `flask-app`, so the app is only reachable inside the cluster. (`hello-k8s` is the service from Exercise 1.)

### Step 11: Add a Service to the YAML

The updated **flask-deployment.yaml** now includes a `NodePort` Service (`port: 15000` mapped to `targetPort: 15000`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image: flask-app:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 15000
---
apiVersion: v1
kind: Service
metadata:
  name: flask-app-service
spec:
  selector:
    app: flask-app
  ports:
  - port: 15000
    targetPort: 15000
  type: NodePort
```

Apply the updated file:

```cmd
kubectl apply -f flask-deployment.yaml
```

```
deployment.apps/flask-app unchanged
service/flask-app-service created
```

### Step 12: Get the service URL

```cmd
minikube service flask-app-service --url
```

```
http://127.0.0.1:57731
❗  Because you are using a Docker driver on windows, the terminal needs to be open to run it.
```

The terminal must stay open while accessing the service.

### Step 13: Access the application (in a new terminal)

```cmd
curl http://127.0.0.1:57731
```

```
Hello from Flask on Kubernetes!
```

---

## Result

The Flask app was containerized, deployed on Minikube using a Deployment, exposed through a NodePort Service, and successfully accessed, returning `Hello from Flask on Kubernetes!`.

---

## Key Concepts

- **port (Service port):** the port exposed by the Kubernetes Service.
- **targetPort (container port):** the port the application listens on inside the container.
- **imagePullPolicy: Never:** Kubernetes uses only the locally built image and never pulls from a registry like Docker Hub.