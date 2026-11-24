###Writing the DockerFile###
# first line is always the runtime
FROM node:26-bookworm

## Define how to build the application
# create a directory for my app
# mkdir /app
WORKDIR /dov-bear/javascript

# anything below is in /app
COPY package.json .
COPY package-lock.json .
COPY main.js .
COPY public public
COPY views views

# anything needed to be recreated DONT COPY
# Reinstall the dependencies
RUN npm install

# We have installed the dependencies

## How to run the application
ENV PORT="5000"
ENV INSTANCE_NAME="dovbear-cute"
ENV INSTANCE_HASH="hash-bear"

HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD curl -s -f http://localhost:${PORT}/health > /dev/null || exit 1

# What ports the container need to open
EXPOSE ${PORT}

# How to run the application

# Specify the shell to run this
SHELL [ "/bin/sh", "-c" ]

# configures the default command to run the container
ENTRYPOINT node main.js