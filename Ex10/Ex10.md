# Exercise 10: Multi-Node Kubernetes Cluster with Multiple Applications and ReplicaSets

## Problem Statement

An e-commerce platform has two services:

1. **Product Catalog** (AppA): provides information about available products, run with **2 replicas**.
2. **Shopping Cart** (AppB): manages user shopping carts, run with **3 replicas**.

To ensure high availability and fault tolerance, the replicas of each service must be distributed across multiple nodes, so that if one node fails the services stay available. Deploy both applications on a **3-node Minikube cluster** and verify the pod distribution and service access.

**Environment:** Windows, VS Code integrated terminal (Command Prompt), Docker Desktop, Minikube (Docker driver), kubectl

---

## Windows Adjustments

- The `--force` flag (needed only when running as root on Linux) is not used.
- `eval $(minikube docker-env)` is not used. Images are built with the normal Docker CLI and then copied to every node with `minikube image load`.
- `curl` commands use double quotes with escaped inner quotes (`\"`), because single quotes do not work in Command Prompt.
- `minikube service ... --url` is used so the URL is printed in the terminal (the terminal must stay open while the tunnel is running).
- In `shopping_cart_deployment.yaml`, the namespace is `default` instead of `devops-exercise`. That namespace does not exist in the cluster, and the Service in the `default` namespace would not find pods in another namespace.

---

## Solution (Brief)

1. Delete any existing Minikube cluster and start a new 3-node cluster with the profile `devops-multinode`.
2. Enable the registry add-on for the profile.
3. Create two Flask applications (Product Catalog and Shopping Cart) and Dockerize them.
4. Build the images and load them into all Minikube nodes.
5. Deploy both applications as Deployments (2 and 3 replicas) with **pod anti-affinity**, which forces each replica of an app onto a different node.
6. Expose both with NodePort Services, verify the pod distribution, and test the APIs with `curl`.

---

## Files Created

```
Ex10/
├── product_catalog.py
├── shopping_cart.py
├── Dockerfile.product
├── Dockerfile.shopping
├── product_catalog_deployment.yaml
├── shopping_cart_deployment.yaml
├── product_catalog_service.yaml
└── shopping_cart_service.yaml
```

**product_catalog.py**

```python
# product_catalog.py
from flask import Flask, jsonify

app = Flask(__name__)

products = [
    {"id": 1, "name": "Laptop", "price": 1200},
    {"id": 2, "name": "Phone", "price": 800},
    {"id": 3, "name": "Headphones", "price": 150},
]

@app.route("/products", methods=["GET"])
def get_products():
    return jsonify(products)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

**shopping_cart.py**

```python
# shopping_cart.py
from flask import Flask, jsonify, request

app = Flask(__name__)

cart = []

@app.route("/cart", methods=["GET"])
def get_cart():
    return jsonify(cart)

@app.route("/cart", methods=["POST"])
def add_to_cart():
    item = request.json
    cart.append(item)
    return jsonify(cart), 201

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

**Dockerfile.product**

```dockerfile
# Dockerfile for Product Catalog
FROM python:3.9-slim
WORKDIR /app
COPY product_catalog.py /app/
RUN pip install flask
CMD ["python", "product_catalog.py"]
```

**Dockerfile.shopping**

```dockerfile
# Dockerfile for Shopping Cart
FROM python:3.9-slim
WORKDIR /app
COPY shopping_cart.py /app/
RUN pip install flask
CMD ["python", "shopping_cart.py"]
```

**product_catalog_deployment.yaml**

```yaml
# product_catalog_deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-catalog
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: product-catalog
  template:
    metadata:
      labels:
        app: product-catalog
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: product-catalog
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: product-catalog-container
        image: product-catalog:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 80
```

**shopping_cart_deployment.yaml**

```yaml
# shopping_cart_deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopping-cart
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shopping-cart
  template:
    metadata:
      labels:
        app: shopping-cart
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: shopping-cart
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: shopping-cart-container
        image: shopping-cart:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 80
```

**product_catalog_service.yaml**

```yaml
# product_catalog_service.yaml
apiVersion: v1
kind: Service
metadata:
  name: product-catalog-service
  namespace: default
spec:
  selector:
    app: product-catalog
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

**shopping_cart_service.yaml**

```yaml
# shopping_cart_service.yaml
apiVersion: v1
kind: Service
metadata:
  name: shopping-cart-service
  namespace: default
spec:
  selector:
    app: shopping-cart
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

---

## Commands Executed and Output

### Step 0: Clean up the existing Minikube cluster

This removes the default `minikube` cluster used in the earlier exercises.

```cmd
minikube stop
minikube delete
```

```
✋  Stopping node "minikube"  ...
🛑  1 node stopped.
💀  Removed all traces of the "minikube" cluster.
```

### Step 1: Create the 3-node Minikube cluster

```cmd
minikube start --nodes 3 -p devops-multinode --driver=docker
```

```
🏄  Done! kubectl is now configured to use "devops-multinode" cluster and "default" namespace by default
```

Verify the nodes:

```cmd
kubectl get nodes
```

```
NAME                   STATUS   ROLES           AGE     VERSION
devops-multinode       Ready    control-plane   4m20s   v1.35.1
devops-multinode-m02   Ready    <none>          3m15s   v1.35.1
devops-multinode-m03   Ready    <none>          2m16s   v1.35.1
```

### Step 2: Enable the registry add-on

```cmd
minikube -p devops-multinode addons enable registry
```

```
🌟  The 'registry' addon is enabled
```

### Step 3: Build the Docker images

```cmd
docker build -t product-catalog:latest -f Dockerfile.product .
```

```
[+] Building 9.8s (10/10) FINISHED   docker:desktop-linux
```

```cmd
docker build -t shopping-cart:latest -f Dockerfile.shopping .
```

```
[+] Building 6.9s (9/9) FINISHED     docker:desktop-linux
```

```cmd
docker images | findstr "product-catalog shopping-cart"
```

```
product-catalog:latest                                                                                c526bed693cf        203MB         49.7MB        
shopping-cart:latest                                                                                  f72e46643771        203MB         49.7MB
```

### Step 4: Load the images into the Minikube nodes

```cmd
minikube -p devops-multinode image load product-catalog:latest
minikube -p devops-multinode image load shopping-cart:latest
```


Verify the images on the primary node:

```cmd
minikube -p devops-multinode ssh -- docker images
```

```
  i Info →   U  In Use
IMAGE                           ID             DISK USAGE   CONTENT SIZE   EXTRA
gcr.io/k8s-minikube/storage-provisioner:v5
                                6e38f40d628d       31.5MB             0B    U   
kindest/kindnetd:v20260213-ea8e5717
                                c4409203da81        106MB             0B    U   
product-catalog:latest          bd1fa5fcbfb3        134MB             0B        
registry.k8s.io/coredns/coredns:v1.13.1
                                aa5e3ebc0dfe       78.1MB             0B    U   
registry.k8s.io/etcd:3.6.6-0    0a108f718956       62.5MB             0B    U   
registry.k8s.io/kube-apiserver:v1.35.1
                                6f9eeb0cff98       89.8MB             0B    U   
registry.k8s.io/kube-controller-manager:v1.35.1
                                8d7002962c48       75.8MB             0B    U   
registry.k8s.io/kube-proxy:v1.35.1
                                6521110cdb01       70.4MB             0B    U   
registry.k8s.io/kube-scheduler:v1.35.1
                                5f2a969bc7a4       51.7MB             0B    U   
registry.k8s.io/pause:3.10.1    cd073f4c5f6a        736kB             0B    U   
shopping-cart:latest            2942869f84ed        134MB             0B  
```

Verify that the images are also present on the other nodes (the pods use `imagePullPolicy: Never`, so every node needs a local copy):



```cmd
minikube -p devops-multinode ssh -n devops-multinode-m03 -- docker images --filter reference=product-catalog --filter reference=shopping-cart
```

```
  i Info →   U  In Use
IMAGE                    ID             DISK USAGE   CONTENT SIZE   EXTRA
product-catalog:latest   bd1fa5fcbfb3        134MB             0B        
shopping-cart:latest     2942869f84ed        134MB             0B  
```

### Step 5: Deploy the applications

```cmd
kubectl apply -f product_catalog_deployment.yaml
kubectl apply -f shopping_cart_deployment.yaml
```

```
deployment.apps/product-catalog created
deployment.apps/shopping-cart created
```

### Step 6: Expose the services

```cmd
kubectl apply -f product_catalog_service.yaml
kubectl apply -f shopping_cart_service.yaml
```

```
service/product-catalog-service created
service/shopping-cart-service created
```

```cmd
kubectl get deployments
```

```
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
product-catalog   2/2     2            2           2m33s
shopping-cart     3/3     3            3           2m32s
```

```cmd
kubectl get svc
```

```
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes                ClusterIP   10.96.0.1        <none>        443/TCP        17m
product-catalog-service   NodePort    10.106.86.13     <none>        80:31788/TCP   68s
shopping-cart-service     NodePort    10.104.174.198   <none>        80:30908/TCP   67s
```

### Step 7: Verify the deployment and pod distribution

```cmd
kubectl get pods -o wide
```

```
NAME                               READY   STATUS    RESTARTS   AGE     IP           NODE                   NOMINATED NODE   READINESS GATES
product-catalog-6794cdc954-6q7tv   1/1     Running   0          3m17s   10.244.2.3   devops-multinode-m03   <none>           <none>
product-catalog-6794cdc954-9nhbh   1/1     Running   0          3m17s   10.244.1.4   devops-multinode-m02   <none>           <none>
shopping-cart-7bf585c6f8-dg2nw     1/1     Running   0          3m16s   10.244.2.4   devops-multinode-m03   <none>           <none>
shopping-cart-7bf585c6f8-dt24n     1/1     Running   0          3m16s   10.244.1.5   devops-multinode-m02   <none>           <none>
shopping-cart-7bf585c6f8-fjqt7     1/1     Running   0          3m16s   10.244.0.4   devops-multinode       <none>           <none>
```

### Step 8: Access the services

Open a **new terminal** for each command and keep both terminals open while testing (the tunnel stops when the terminal is closed).

**Product Catalog service**

```cmd
minikube -p devops-multinode service product-catalog-service --url
```

```
http://127.0.0.1:64933
❗  Because you are using a Docker driver on windows, the terminal needs to be open to run it.
```

**Shopping Cart service**

```cmd
minikube -p devops-multinode service shopping-cart-service --url
```

```
http://127.0.0.1:55932
❗  Because you are using a Docker driver on windows, the terminal needs to be open to run it.
```

### Step 9: Test the services with curl

Replace `<PRODUCT-CATALOG-PORT>` and `<SHOPPING-CART-PORT>` with the ports printed in Step 8.

**Product Catalog: retrieve the product list**

```cmd
curl http://127.0.0.1:<PRODUCT-CATALOG-PORT>/products
```

```
[{"id":1,"name":"Laptop","price":1200},{"id":2,"name":"Phone","price":800},{"id":3,"name":"Headphones","price":150}]
```

**Shopping Cart: retrieve the (initially empty) cart**

```cmd
curl http://127.0.0.1:<SHOPPING-CART-PORT>/cart
```

```
[]
```

**Shopping Cart: add an item**

```cmd
curl -X POST http://127.0.0.1:<SHOPPING-CART-PORT>/cart -H "Content-Type: application/json" -d "{\"id\": 1, \"name\": \"Laptop\", \"quantity\": 1}"
```

```
[{"id":1,"name":"Laptop","quantity":1}]
```

### Cleanup (optional)

```cmd
minikube delete -p devops-multinode
```

```
💀  Removed all traces of the "devops-multinode" cluster.
```

---

## Result

A 3-node Minikube cluster (`devops-multinode`) was created and both applications were deployed as Deployments with ReplicaSets (Product Catalog: 2 replicas, Shopping Cart: 3 replicas). Pod anti-affinity placed each replica of an app on a different node, which provides high availability and fault tolerance. Both services were exposed with NodePort Services and tested successfully with `curl`.

---

## Notes

- **podAntiAffinity (`requiredDuringSchedulingIgnoredDuringExecution`)** with `topologyKey: kubernetes.io/hostname` means no two pods of the same app can run on the same node. This is why the Shopping Cart (3 replicas) needs at least 3 nodes.
- **imagePullPolicy: Never** makes Kubernetes use only the locally loaded images instead of pulling from a registry, so the images must be loaded on every node.
- **In-memory cart:** the cart is stored in memory inside each pod, so with 3 replicas, repeated `GET /cart` requests can return different results depending on which replica handles the request.