# Exercise 4: Docker Networking with Multiple Containers

## Problem Statement

Understand Docker networking concepts and configure a multi-container application consisting of:

- A Python Flask web server (container 1)
- A MySQL database (container 2)
- A Redis cache (container 3)

All three containers must be placed on a user-defined bridge network and be able to reach each other by container name.

**Environment:** Windows, VS Code integrated terminal (Command Prompt), Docker

---

## Solution (Brief)

1. Create a user-defined bridge network (`my-bridge-net`) and verify/inspect it.
2. Build a Docker image for a simple Flask REST API.
3. Launch the MySQL, Redis, and Flask containers on `my-bridge-net`.
4. Exec into the Flask container and ping `mysql` and `redis` by name to prove that Docker's built-in DNS resolves container names on a user-defined network.
5. Clean up the containers and the network.

---

## Files Created

**app.py**

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/about', methods=['GET'])
def about():
    return jsonify({
        "name": "Simple REST API",
        "version": "1.0",
        "description": "This is a simple REST API built with Flask."
    })

if __name__ == '__main__':
    app.run(debug=True, port=5001)  # Specify the port number here
```

**requirements.txt**

```
Flask==2.0.1
```

**Dockerfile**

```dockerfile
# Use the official Python image from the Docker Hub
FROM python:3.9-slim

# Set the working directory in the container
WORKDIR /app

# Copy the requirements file and app code to the container
COPY requirements.txt .
COPY app.py .

# Install Flask (and any other dependencies you might have)
RUN pip install --no-cache-dir -r requirements.txt

# Expose the port the app runs on
EXPOSE 5001

# Define the command to run the application
CMD ["python", "app.py"]
```

---

## Commands Executed and Output

### Task 1: Create a bridge network

```cmd
docker network create --driver bridge my-bridge-net
```

```
8a535fd3d50922de366cca4082f0701ed8c71a9cb3b2f4bc8b31c0fa4ee0a649
```

### Task 2: Verify the network

```cmd
docker network ls
```

```
NETWORK ID     NAME            DRIVER    SCOPE
5392df547f79   bridge          bridge    local
8d61cfc0dbc8   host            host      local
e14724fbfacc   minikube        bridge    local
8a535fd3d509   my-bridge-net   bridge    local
d045e5971a70   none            null      local
```

### Task 3: Inspect the network

```cmd
docker network inspect my-bridge-net
```

```
[
    {
        "Name": "my-bridge-net",
        "Id": "8a535fd3d50922de366cca4082f0701ed8c71a9cb3b2f4bc8b31c0fa4ee0a649",
        "Created": "2026-10-13T20:02:40.998879952Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv4": true,
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Options": {
            "com.docker.network.enable_ipv4": "true",
            "com.docker.network.enable_ipv6": "false"
        },
        "Labels": {},
        "Containers": {},
        "Status": {
            "IPAM": {
                "Subnets": {
                    "172.18.0.0/16": {
                        "IPsInUse": 3,
                        "DynamicIPsAvailable": 65533
                    }
                }
            }
        }
    }
]
```

### Task 4: Build the image and launch the containers

Build the Flask image:

```cmd
docker build -t flask-api .
```

```
[+] Building 1.8s (11/11) FINISHED docker:desktop-li
```

Launch the containers in detached mode (`-d`) on the custom network:

```cmd
docker run -d --name mysql --net=my-bridge-net -e MYSQL_ROOT_PASSWORD=root mysql:latest
docker run -d --name redis --net=my-bridge-net redis:latest
docker run -d --name flask --net=my-bridge-net -p 5001:5001 flask-api
```

> The `MYSQL_ROOT_PASSWORD` variable is required. Without it the MySQL image exits immediately, and its name can no longer be resolved from the other containers.

```
8d9abbf48f331bc7c2f632582813ca7f20b8f1275700e75180a720adbd6122dd

6b3c2cecd0515b27482d5a9101d1229d2001203d87caa371fd989e806860357c

6b3c2cecd0515b27482d5a9101d1229d2001203d87caa371fd989e806860357c
```

Verify that all three containers are running:

```cmd
docker ps
```

```
CONTAINER ID   IMAGE          COMMAND                  CREATED              STATUS              PORTS                 NAMES
6b3c2cecd051   redis:latest   "docker-entrypoint.s…"   About a minute ago   Up About a minute   6379/tcp              redis
8d9abbf48f33   mysql:latest   "docker-entrypoint.s…"   About a minute ago   Up About a minute   3306/tcp, 33060/tcp   mysql
```

Verify that the containers are attached to the network:

```cmd
docker network inspect my-bridge-net
```

```
[
    {
        "Name": "my-bridge-net",
        "Id": "8a535fd3d50922de366cca4082f0701ed8c71a9cb3b2f4bc8b31c0fa4ee0a649",
        "Created": "2026-10-13T20:02:40.998879952Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv4": true,
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Options": {
            "com.docker.network.enable_ipv4": "true",
            "com.docker.network.enable_ipv6": "false"
        },
        "Labels": {},
        "Containers": {
            "6b3c2cecd0515b27482d5a9101d1229d2001203d87caa371fd989e806860357c": {
                "Name": "redis",
                "EndpointID": "131d9b61ab78cfe27c3c1621f019aed36782a5298c3e5245bf7d7fe924b82ca0",
                "MacAddress": "8e:50:85:dc:57:f9",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            },
            "8d9abbf48f331bc7c2f632582813ca7f20b8f1275700e75180a720adbd6122dd": {
                "Name": "mysql",
                "EndpointID": "8237b186aaa221472778b0259e2eed53b0b4499dad23612dd11fe348929c825e",
                "MacAddress": "fa:94:34:b0:af:4f",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Status": {
            "IPAM": {
                "Subnets": {
                    "172.18.0.0/16": {
                        "IPsInUse": 5,
                        "DynamicIPsAvailable": 65531
                    }
                }
            }
        }
    }
]
```

### Task 5: Test connectivity

Exec into the Flask container:

```cmd
docker exec -it flask bash
```

The `python:3.9-slim` image does not include `ping`, so install it inside the container first:

```bash
apt-get update && apt-get install -y iputils-ping
```



Ping the MySQL container:

```bash
ping -c 4 mysql
```

```
PING mysql (172.18.0.2) 56(84) bytes of data.
64 bytes from mysql.my-bridge-net (172.18.0.2): icmp_seq=1 ttl=64 time=0.300 ms
```

Ping the Redis container:

```bash
ping -c 4 redis
```

```
PING redis (172.18.0.3) 56(84) bytes of data.
64 bytes from redis.my-bridge-net (172.18.0.3): icmp_seq=1 ttl=64 time=0.200 ms
```

Exit the container:

```bash
exit
```

### Task 6: Clean up

Stop and remove the containers:

```cmd
docker stop mysql redis flask && docker rm mysql redis flask
```

```
mysql
redis
flask
mysql
redis
flask
```

Remove the network:

```cmd
docker network rm my-bridge-net
```

```
my-bridge-net
```

---

## Result

A user-defined bridge network was created and the Flask, MySQL, and Redis containers were attached to it. From inside the Flask container, both `mysql` and `redis` were reachable by name, showing that containers on the same user-defined bridge network can communicate using container names (Docker's embedded DNS) instead of IP addresses.

---

## Questions and Answers

**1. What is the purpose of the `--net` flag in `docker run`?**
It specifies the network the container is connected to.

**2. How do containers communicate with each other on the same network?**
By using container names (resolved by Docker's embedded DNS) or IP addresses.

**3. What is the difference between a bridge network and a host network?**
A bridge network gives containers their own isolated network namespace and connects them through a virtual bridge. A host network removes this isolation and makes the container share the host's network stack directly.

**4. How can you expose a container's port to the host machine?**
Use the `-p` flag to publish it, for example `-p 5001:5001` (`host_port:container_port`).