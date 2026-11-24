# Docker & Containers — Quick Notes

> A concise reference covering container fundamentals, Dockerfile structure, image building, and container management.

---

## 1. Containers

### What is a container?

A **container** is a packaging of everything required to run an application.

## 2. Docker Fundamentals

- **One container, one application.** Keep the database separate.
- Docker describes:
  - How to build the application.
  - How to run the application.

## 3. Selecting a Docker Image

- Always choose an **official Docker image** or a **hardened image**.

## 4. Writing a Dockerfile

A Dockerfile defines the runtime, application files, dependencies, configuration, health check, exposed port, and startup command.

### Example Dockerfile

```dockerfile
# Start with the runtime
FROM node:26-bookworm

# Define how to build the application
# Create /app and make it the working directory
WORKDIR /app

# Copy the application files into /app
COPY package.json .
COPY package-lock.json .
COPY main.js .
COPY publick public
COPY views views

# Reinstall dependencies inside the image
RUN npm install

# Define application configuration
ENV APP_PORT="3000"

# Check whether the application is healthy
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \\
    CMD curl -s -f http://localhost:${APP_PORT}/health > /dev/null || exit 1

# Document the port used by the container
EXPOSE ${APP_PORT}

# Specify the shell used to run commands
SHELL [ "/bin/sh", "-c" ]

# Configure the default startup command
ENTRYPOINT node main.js
```

### Dockerfile sections at a glance

| Instruction | Purpose |
|---|---|
| `FROM` | Selects the base runtime image. |
| `WORKDIR` | Sets the working directory inside the image. |
| `COPY` | Copies application files into the image. |
| `RUN` | Executes build-time commands, such as installing dependencies. |
| `ENV` | Defines an environment variable. |
| `HEALTHCHECK` | Defines how Docker checks application health. |
| `EXPOSE` | Documents the port the application listens on. |
| `SHELL` | Selects the shell used for shell-form commands. |
| `ENTRYPOINT` | Defines the default command used to start the container. |

> **Build reminder:** Files or dependencies that need to be recreated during the image build should not simply be copied from the host. The notes use `npm install` to reinstall the dependencies in the image.

## 5. Building and Managing Docker Images

### Build an image

```bash
docker build -t day01-fortune .
```

### Tag an image

```bash
docker tag weiss/day01-fortune:1.0.0-beta01 ghcr/weiss/day01-fortune:1.0.0-beta01
```

### List images

```bash
docker image ls
```

## 6. Running and Managing Containers

### Run a container

Map host port `9000` to container port `3000`:

```bash
docker run -d -p 9000:3000 weiss/day01-fortune:1.0.0-beta01
```

### Run another container at the same time

Map host port `9001` to container port `3000`:

```bash
docker run -d -p 9001:3000 weiss/day01-fortune:1.0.0-beta01
```

### List all containers

```bash
docker ps -a
```

### List running containers

```bash
docker ps
```

### Open a shell inside a running container

```bash
docker exec -ti b0e8 /bin/sh
```

> This works only for containers that include a shell. If the container has no shell, you cannot use this command to open `/bin/sh` inside it.

## 7. Debugging an Image

Use **Dive** to inspect an image and check whether it has been built correctly:

```bash
dive weiss/day01-fortune:1.0.0-beta01
```

## 8. Setting Environment Variables at Runtime

Example from the notes:

```bash
docker run -d -e APP_PORT=9000 VAR=abc -p 8080:9000 weiss/day01-fortune:1.0.0-beta01
```

This sets runtime environment values and maps host port `8080` to container port `9000`.

## 9. Removing an Image

```bash
docker rmi weiss/day01-fortune:1.0.0-beta01
```

> Docker only allows an image to be removed when it is not being used by a running container.

---

## Quick Command Reference

| Task | Command |
|---|---|
| Build image | `docker build -t day01-fortune .` |
| List images | `docker image ls` |
| Tag image | `docker tag <source> <target>` |
| Run container | `docker run -d -p <host-port>:<container-port> <image>` |
| List all containers | `docker ps -a` |
| List running containers | `docker ps` |
| Execute shell in container | `docker exec -ti <container-id> /bin/sh` |
| Inspect image with Dive | `dive <image>` |
| Remove image | `docker rmi <image>` |
