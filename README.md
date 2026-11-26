# Containers, Docker & Kubernetes Notes

## Contents

1. [Core concepts](#1-core-concepts)
2. [Docker](#2-docker)
3. [Kubernetes](#3-kubernetes)

---

## 1. Core concepts

| Term | Meaning |
|---|---|
| **Container** | A running instance of an image: your application packaged with everything it needs to run |
| **Image** | A read-only template that containers are created from |
| **Tag** | The version label on an image, for example `weiss/day01-fortune:1.0.0-beta01` is repository `weiss/day01-fortune` with tag `1.0.0-beta01` |

- One image can have **many containers**. Each container has its own writable layer and state, but the image never changes.
- **Versions belong to the image** (through tags), not to the container. One repository can hold many versions, such as `v1`, `v2` and `latest`.

---

## 2. Docker

Docker builds images and runs containers on a single machine. A Dockerfile describes two things: how to **build** the application and how to **run** it.

### Principles

- One container runs one application. Keep the database in a separate container.
- Always pick a **Docker Official Image** or **Hardened Image** as the base.

### Dockerfile

```dockerfile
# The first line is always the runtime (base image)
FROM node:26-bookworm

## How to build the application

# Create /app; everything below runs in it
WORKDIR /app

COPY package.json .
COPY package-lock.json .
COPY main.js .
COPY publick public
COPY views views

# Don't copy anything that can be recreated (such as node_modules); reinstall it
RUN npm install

## How to run the application
ENV APP_PORT="3000"

HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD curl -s -f http://localhost:${APP_PORT}/health > /dev/null || exit 1

# The port the container listens on
EXPOSE ${APP_PORT}

# The shell used to run the command below
SHELL [ "/bin/sh", "-c" ]

# The default command when the container starts
ENTRYPOINT node main.js
```

### Build and tag

```bash
docker build -t day01-fortune .
docker tag weiss/day01-fortune:1.0.0-beta01 ghcr.io/weiss/day01-fortune:1.0.0-beta01
docker image ls
```

### Run containers

```bash
# -d runs in the background; -p <host port>:<container port>
docker run -d -p 9000:3000 weiss/day01-fortune:1.0.0-beta01

# A second container from the same image, on a different host port
docker run -d -p 9001:3000 weiss/day01-fortune:1.0.0-beta01

# Give a container a name
docker run -d --name workshop01 -p 9001:3000 workshop01:v1.0.0

docker ps        # running containers
docker ps -a     # all containers, including stopped ones
```

### Environment variables

`ENV` in the Dockerfile sets a default, and `-e` overrides it at run time. Each variable needs its own `-e`:

```bash
docker run -d -e APP_PORT=9000 -e VAR=abc -p 8080:9000 weiss/day01-fortune:1.0.0-beta01
```

### Inspect and debug

```bash
# Open a shell in a running container (only works if the image contains a shell)
docker exec -ti <container-id> /bin/sh

# Check that an image is built correctly (shows each layer)
dive weiss/day01-fortune:1.0.0-beta01
```

### Volumes and databases

```bash
docker volume create data-vol

docker run -d -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=abc123 \
  --mount type=volume,src=data-vol,dst=/var/lib/mysql \
  mysql:8
```

### Networking

Containers on the same user-defined network reach each other by container name.

```bash
docker network create mynet

# App container named "myapp" on the network "mynet"
docker run -d -p 9000:3000 --network mynet --name myapp weiss/day01-fortune:1.0.0-beta01

# Throwaway debugging container with network tools, on the same network
docker run -ti --rm --network mynet nicolaka/netshoot /bin/bash
```

### Cleanup

```bash
# An image can only be removed when no running container uses it
docker rmi weiss/day01-fortune:1.0.0-beta01
```

---

## 3. Kubernetes

### Docker vs Kubernetes

| | Docker | Kubernetes |
|---|---|---|
| Builds images | Yes (`docker build`) | No |
| Runs containers | On one machine, by hand | Across a cluster, automatically |
| Where images come from | Built locally or pulled | Pulled from a registry (Docker Hub, `ghcr.io`) |

Kubernetes creates the containers itself from existing images, and then manages them:

- restarts failed containers and replaces them
- scales the number of copies (replicas)
- gives them stable names and network routes (Services, Gateway)
- attaches configuration and storage (ConfigMaps, Secrets, PVCs)
- decides which node each one runs on

Things to remember:

- Kubernetes does not take over containers started with `docker run`. It starts its own from the image.
- Containers run inside **Pods**, using a container runtime such as `containerd`, not Docker itself. Docker images still work because they follow the standard OCI format.
- Usual flow: write a Dockerfile, build and push an image with Docker, then point a Kubernetes Deployment at that image.

In short: **Docker packages the application, and Kubernetes runs and looks after it.**

### Docker concepts and their Kubernetes equivalents

| Docker | Kubernetes |
|---|---|
| `docker run` (one container) | Pod, usually managed by a Deployment |
| Running the same `docker run` several times | `replicas` in a Deployment |
| `-e VAR=value` | `env`, a ConfigMap, or a Secret |
| `--mount type=volume` / `docker volume create` | PersistentVolumeClaim (PVC) |
| `--network` and `--name` (reach another container by name) | Service (reach a pod by the Service name) |
| `-p 9000:3000` | Service (ClusterIP / NodePort), or a Gateway and HTTPRoute |
| `HEALTHCHECK` in the Dockerfile | `readinessProbe` and `livenessProbe` |
| Replacing a container with a new image version | Rolling update (`maxSurge` / `maxUnavailable`) |

### Services: normal vs headless

| | Normal Service (ClusterIP) | Headless Service |
|---|---|---|
| Setting | `clusterIP` assigned automatically | `clusterIP: None` |
| Virtual IP | Yes, one stable IP | None |
| DNS lookup of the Service name | Returns the Service IP | Returns the pod IPs |
| Load balancing | Done by the cluster | None, the client chooses a pod IP |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: umamidb-headless
  namespace: workshop03
spec:
  clusterIP: None
  selector:
    app: umamidb
  ports:
  - port: 5432
    targetPort: db-port
```

- "Hides" only refers to DNS. Pod IPs are not secret, and a pod can use one directly if it knows it. A normal Service puts one stable address in front of them, because pod IPs change whenever a pod is replaced.
- The selector must still match the pod labels, as with any Service.

Use a headless Service for:

- **StatefulSets and databases**, where clients need a specific pod. Each pod gets its own DNS name, such as `mysql-0.mysql-headless.workshop03.svc.cluster.local`.
- **Client-side load balancing**, where the client wants the full list of backends.
- **Service discovery**, where you need all the pod IPs, not just one address.

To compare the two, run `nslookup` from a debug pod:

```bash
kubectl run debug -n workshop03 -ti --rm --image=nicolaka/netshoot -- /bin/bash
nslookup umamidb-svc          # returns one ClusterIP
nslookup umamidb-headless     # returns the pod IPs (one per replica)
```
