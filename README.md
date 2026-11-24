# Containers & Docker Notes

## Containers

**What is a container?** A packaging of everything required to run an application.

## Docker

- 1 container = 1 application; the database should be separated out
- Docker describes two things: how to **build** the application and how to **run** the application

## Selecting a Docker Image

- Always pick a **Docker Official Image** or **Hardened Image**

## Writing the Dockerfile

```dockerfile
# first line is always the runtime
FROM node:26-bookworm

## Define how to build the application
# create a directory for my app
# mkdir /app
WORKDIR /app

# anything below is in /app
COPY package.json .
COPY package-lock.json .
COPY main.js .
COPY publick public
COPY views views

# anything needed to be recreated DONT COPY
# Reinstall the dependencies
RUN npm install

# We have installed the dependencies

## How to run the application
ENV APP_PORT="3000"

HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD curl -s -f http://localhost:${APP_PORT}/health > /dev/null || exit 1

# What ports the container need to open
EXPOSE ${APP_PORT}

# How to run the application

# Specify the shell to run this
SHELL [ "/bin/sh", "-c" ]

# configures the default command to run the container
ENTRYPOINT node main.js
```

## Building the Docker Image

### Build and tag

```bash
# build the image
docker build -t day01-fortune .

# tag the image
docker tag weiss/day01-fortune:1.0.0-beta01 ghcr/weiss/day01-fortune:1.0.0-beta01

# list images
docker image ls
```

### Running containers

```bash
# running the container on port 9000
docker run -d -p 9000:3000 weiss/day01-fortune:1.0.0-beta01
docker run -d --name workshop01 -p 9001:3000 workshop01:v1.0.0

# shows everything
docker ps -a

# shows running containers
docker ps

# running another container simultaneously on port 9001
docker run -d -p 9001:3000 weiss/day01-fortune:1.0.0-beta01
```

### Inspecting and debugging

```bash
# access a running server (container)
# only for containers with a shell; without a shell you CANNOT exec
docker exec -ti b0e8 /bin/sh
```

Use `dive` to debug and check whether the image is built correctly:

```bash
dive weiss/day01-fortune:1.0.0-beta01
```

### Environment variables

```bash
# change the value of environment variable
docker run -d -e APP_PORT=9000 VAR=abc -p 8080:9000 weiss/day01-fortune:1.0.0-beta01
```

### Removing an image

Only allowed for images that are not in use by a running container.

```bash
docker rmi weiss/day01-fortune:1.0.0-beta01
```

## Volumes and Databases

### Create the data volume

```bash
docker volume create data-vol
```

### Create the MySQL container

```bash
docker run -d -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=abc123 \
  --mount type=volume,src=data-vol,dst=/var/lib/mysql \
  mysql:8
```

## Networking

### Run the app container on the network `mynet` with the container name `myapp`

```bash
docker run -d -p 9000:3000 --network mynet --name myapp weiss/day01-fortune:1.0.0-beta01
```

### Spin up `nicolaka/netshoot` in `mynet`

```bash
docker run -ti --rm --network mynet nicolaka/netshoot /bin/bash
```
