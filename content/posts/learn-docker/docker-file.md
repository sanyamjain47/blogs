---
title: "Dockerfiles"
summary: "What are Dockerfiles and how to use them."
date: "2023-12-26"
tags: ["Docker", "Dockerfile"]
series: ["Learning Docker"]
author: ["Sanyam Jain"]
draft: false
---

# Dockerfiles
- Dockerfiles are used to build images.
- They are a list of instructions that are used to build an image.
- `docker build -t <image_name> <path_to_dockerfile>` is used to build an image from a Dockerfile.
# Example
> This example is based on https://github.com/sidpalas/devops-directive-docker-course/tree/main/05-example-web-application . This is part of the course that I am using to learn Docker.

## api-node

```dockerfile
FROM ubuntu
RUN apt update
RUN apt install nodejs -y
```
- In the above example, we are using the `ubuntu` image as the base image.
- Currently, we are not specifying the version of the image. This is not recommended. We should always specify the version of the image.
- Also, every command in the Dockerfile creates a new layer. This is not recommended. We should always try to combine multiple commands into a single command.
- We are using the `RUN` command to run the commands.
- Now, let's merge the two `RUN` commands into a single command.
- Copy the files also now.
- We also added `npm` since we forgot to add it earlier.

```dockerfile
FROM ubuntu
RUN apt update && apt install nodejs npm -y
COPY . .
RUN npm install
```
- Now, add the `CMD` command to run the application.

```dockerfile
FROM ubuntu
RUN apt update && apt install nodejs npm -y
COPY . .
RUN npm install
CMD ["npm","run","dev"]
```

- Now let's start optimizing the Dockerfile.
- For starters, since we are only using this image for nodejs, we can use the `node` image as the base image instead of the `ubuntu` image. Also, we should specify the version of the image. Let's use `node:19.6-alpine` as the image.
- Instead of copying the whole source directory, we can copy the `package.json` file first and then run `npm install`. This will allow us to cache the `node_modules` folder. This will speed up the build process.
- We can also use `WORKDIR` to set the working directory to `/app`. This will allow us to run the `npm install` command without specifying the path.
- When using the copy command we can have some files that we don't want to copy. We can use `.dockerignore` to specify the files that we don't want to copy. Or we can use the `COPY` command to copy only the files that we want to copy.
- We now change `npm run dev` to `node index.js`.
- All the changes are shown below.

```dockerfile
FROM node:19.6-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY ./src .
CMD ["node","index.js"]
```

- Up until now, we have been running the commands as a root user. This is not recommended. We should always run the commands as a non-root user.
- We can use the `USER` command to specify the user that we want to run the commands as. This involves creating a user and then switching to that user. Also, while copying the files, we need to make sure that the user has the required permissions to copy the files.
- We can use the `chown` command to change the owner of the files.

```dockerfile
FROM node:19.6-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
USER node
COPY --chown=node:node ./src .
CMD ["node","index.js"]
```

-- Now, We can do some `node.js` specific optimizations.
- We can start by using an `ENV` variable to specify the environment. This will allow us to use different configurations for different environments.
- We can also use `npm ci` instead of `npm install`. This will install the exact versions of the dependencies specified in the `package-lock.json` file. This will speed up the build process. Adding `--only=production` will install only the production dependencies. This will reduce the size of the image.
- Also, we would like to `EXPOSE` the port `3000` so that we can access the application from outside the container.

```dockerfile
FROM node:19.6-alpine
WORKDIR /usr/src/app
ENV NODE_ENV production
COPY package*.json ./
RUN npm ci --only=production
USER node
COPY --chown=node:node ./src .
EXPOSE 3000
CMD ["node","index.js"]
```

> We can change `npm ci --only=production` to `--mount=type=cache,target=/usr/src/app/.npm \ npm set cache /usr/src/app/.npm && \ npm ci --only=production` to use a cache volume. This will speed up the build process even more. 


## go-app

- We start with the golang image. 
- We set the Workdir.
- We go with a naive approach and then optimize it.

```dockerfile
from golang
WORKDIR /app
COPY . .
RUN go mod download
CMD ["go","run","./main.go"]
```

- Start by adding a version to the image. We use `golang:1.19-alpine` as the image.
- Also, when using go, we don't want to use `go run main.go` as our command. We want to build the binary and then run the binary.
- Also, we want to download the dependencies first and then copy the source code. This will allow us to cache the dependencies. This will speed up the build process.

```dockerfile
FROM golang:1.19-alpine
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o api-golang
CMD ["./api-golang"]
```

- For this image, we can do 1 more thing. We can use a multi-stage build. This will allow us to build the binary in one stage and then copy the binary to the final image. This will reduce the size of the image.
- There is an image called `scratch` that is used to create images from scratch. This is useful when we want to create images that don't have any dependencies. We can use this image as the base image for our final image.

```dockerfile
FROM golang:1.19-buster AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build \
    -ldflags="-linkmode external -extldflags -static" \
    -tags netgo \
    -o api-golang

FROM scratch
COPY --from=build /app/api-golang /app/api-golang
CMD ["/app/api-golang"]
```
- We can also use `ENV` to specify the environment. In `gin` the package used in golang to create the api, we can use `ENV GIN_MODE=release` to specify the environment.
- We can also use `EXPOSE` to expose the port `8080` so that we can access the application from outside the container.
- Also, we can use a cache mount similarly to how we did it in the `api-node` image.
- Note, we have also changed the image from `alpine` to `buster`. This is because `alpine` doesn't have `glibc` installed. This causes issues when running the binary. So, we use `buster` instead. Since, we are using the `buster` image as an intermediate image, this doesn't affect the size of the final image.

```dockerfile
FROM golang:1.19-buster AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go mod download
COPY . .
RUN go build \
    -ldflags="-linkmode external -extldflags -static" \
    -tags netgo \
    -o api-golang

FROM scratch
ENV GIN_MODE release
COPY --from=build /app/api-golang /app/api-golang
EXPOSE 8080
CMD ["/app/api-golang"]
```

- Last thing we can do is to use a non-root user. This is similar to what we did in the `api-node` image.

```dockerfile
FROM golang:1.19-buster AS build
WORKDIR /app
RUN useradd -u 10001 appuser
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go mod download
COPY . .
RUN go build \
    -ldflags="-linkmode external -extldflags -static" \
    -tags netgo \
    -o api-golang

FROM scratch
ENV GIN_MODE release
COPY --from=build /etc/passwd /etc/passwd
COPY --from=build /app/api-golang /app/api-golang
USER appuser
EXPOSE 8080
CMD ["/app/api-golang"]
```

## react-app
- We start with the `node` image. This would be very similar to the `api-node` image.
- Not many optimizations to do here. We learnt alot about optimizing the `api-node` image. We can use the same optimizations here. We can also use a multi-stage build to reduce the size of the image. 

```dockerfile
FROM node:19.4-bullseye as build
WORKDIR /user/src/app
COPY package*.json ./
RUN --mount=type=cache,target=/usr/src/app/.npm \
    npm set cache /usr/src/app/.npm && \
    npm ci
COPY . .
RUN npm run build

###
FROM nginxinc/nginx-unprivileged:1.23-alpine-perl 

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /usr/src/app/dist /usr/share/nginx/html

EXPOSE 8080
```


# General Principles while writing Dockerfiles
- Pin Speicific Versions
    - Base images
    - System Dependencies
    - Application Dependencies
- Use small + secure base images
- Protect the layer cache
    - Use mutli-stage builds
    - Order commands by frequency of change
    - Copy dependency requirements first -> Intall dependencies -> Copy source code
    - Use Cache mounts
    - Use COPY --link
- Be explicit
    - Set working directory
    - Set environment variables
    - Set standard port
- Avoid unnecessary packages
    - COPY specific files
    - Use .dockerignore
- Use non-root user
- Install only production dependencies


## Notes
> `COPY --link`  is very useful in building an image as this makes the copy step an independent step. This means that if the source code changes, the cache for the dependencies will not be invalidated. This will speed up the build process.