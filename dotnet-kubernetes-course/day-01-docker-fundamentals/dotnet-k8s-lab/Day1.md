# Docker + .NET — Day 1 Complete Notes

## Learning Goal

Understand the fundamental Docker concepts required to containerize a .NET application and run multiple containers together.

By the end of Day 1, we should understand:

1. Docker fundamentals
2. Image vs Container
3. Dockerfile
4. Building an image
5. Running a container
6. Port mapping
7. Container lifecycle
8. Container filesystem
9. Docker networking
10. Container-to-container communication
11. Docker DNS
12. `localhost` inside a container
13. Multi-stage Docker builds
14. Persistent storage and volumes
15. Core Docker commands
16. Architect-level relationships between these concepts

---

# 1. Docker Fundamentals

Docker allows applications to be packaged together with their runtime dependencies and run in an isolated environment called a container.

A useful mental model:

```text
Application
     +
Runtime
     +
Dependencies
     +
Configuration
     ↓
Docker Image
     ↓
Docker Container
```

Docker packages the application and its dependencies so that it can run consistently across environments.

---

# 2. Image vs Container

## Docker Image

A Docker image is an immutable package/template used to create containers.

It can contain:

- Application code
- Runtime
- Libraries
- Dependencies
- Configuration
- Files required by the application

Think:

```text
Image = Blueprint
```

Docker images are composed of layers and are immutable once created.

---

## Docker Container

A container is a running instance of an image.

```text
Image
  ↓
Container
```

Multiple containers can be created from the same image:

```text
             Docker Image
                  │
       ┌──────────┼──────────┐
       ↓          ↓          ↓
 Container 1  Container 2  Container 3
```

Each container has its own runtime state and writable filesystem layer.

---

## Important Relationship

```text
Dockerfile
    ↓
docker build
    ↓
Docker Image
    ↓
docker run
    ↓
Docker Container
```

---

# 3. Create the .NET Application

Create the working directory:

```powershell
mkdir dotnet-k8s-lab
cd dotnet-k8s-lab
```

Create the ASP.NET Core Web API:

```powershell
dotnet new webapi -n Orders.Api
```

Enter the project:

```powershell
cd Orders.Api
```

Run the application locally:

```powershell
dotnet run
```

### Purpose

Verify that the .NET application works before introducing Docker.

Mental model:

```text
.NET Application
       ↓
dotnet run
       ↓
Application running directly on host
```

---

# 4. Dockerfile

A Dockerfile contains instructions for building a Docker image.

Our initial Dockerfile:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0

WORKDIR /src

COPY . .

RUN dotnet publish -c Release -o /app

WORKDIR /app

ENTRYPOINT ["dotnet", "Orders.Api.dll"]
```

---

# 5. Dockerfile Instructions

## FROM

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0
```

Specifies the base image.

Here we are using the .NET 9 SDK image.

The SDK contains tools required to:

- Compile
- Restore packages
- Build
- Publish

---

## WORKDIR

```dockerfile
WORKDIR /src
```

Sets the working directory inside the image/container.

Subsequent commands operate relative to this directory unless another `WORKDIR` is specified.

---

## COPY

```dockerfile
COPY . .
```

Copies files from the Docker build context into the image.

The first `.`:

```text
Host/build context
```

The second `.`:

```text
Current WORKDIR inside image
```

---

## RUN

```dockerfile
RUN dotnet publish -c Release -o /app
```

Executes a command while the image is being built.

Here it publishes the .NET application.

---

## ENTRYPOINT

```dockerfile
ENTRYPOINT ["dotnet", "Orders.Api.dll"]
```

Defines the default process that runs when the container starts.

Conceptually:

```text
docker run
    ↓
dotnet Orders.Api.dll
    ↓
Application starts
```

---

# 6. Build the Docker Image

Command:

```powershell
docker build -t orders-api:day1 .
```

Breakdown:

```text
docker build
    → Build an image

-t
    → Assign a name and tag

orders-api
    → Image name

day1
    → Image tag

.
    → Build context = current directory
```

The final `.` tells Docker to use the current directory as the build context.

---

# 7. List Docker Images

```powershell
docker images
```

Shows locally available images.

Typical information:

```text
REPOSITORY
TAG
IMAGE ID
CREATED
SIZE
```

Example:

```text
orders-api    day1    abc123    ...    250MB
```

---

# 8. Run a Container

```powershell
docker run --name orders-api-container -p 8080:8080 orders-api:day1
```

This:

1. Creates a container
2. Gives it the name `orders-api-container`
3. Maps port 8080
4. Starts the application

The container is created from:

```text
orders-api:day1
```

---

# 9. Port Mapping

The syntax is:

```text
-p HOST_PORT:CONTAINER_PORT
```

Example:

```text
-p 8080:8080
```

means:

```text
Host machine
    port 8080
        │
        ▼
Container
    port 8080
```

Therefore:

```text
http://localhost:8080
```

on the host reaches:

```text
container:8080
```

---

## Another Example

```powershell
docker run -p 9000:8080 orders-api:day1
```

means:

```text
Browser
localhost:9000
      ↓
Host port 9000
      ↓
Container port 8080
```

The application inside the container continues listening on port `8080`.

---

# 10. Important Port Concept

Port mapping does NOT change the port on which the application is listening inside the container.

For:

```text
-p 9000:8080
```

the application still listens on:

```text
8080
```

Docker simply exposes that container port through host port:

```text
9000
```

---

# 11. Detached Mode

Run the container in the background:

```powershell
docker run -d --name orders-api -p 8080:8080 orders-api:day1
```

`-d` means:

```text
Detached mode
```

The terminal is returned to you while the container continues running.

---

# 12. Container Lifecycle

Important commands:

```powershell
docker ps
docker ps -a
docker stop orders-api
docker start orders-api
docker rm orders-api
```

---

## docker ps

```powershell
docker ps
```

Shows currently running containers.

---

## docker ps -a

```powershell
docker ps -a
```

Shows:

- Running containers
- Stopped containers

Important:

```text
docker ps
    ↓
Running containers

docker ps -a
    ↓
All containers
```

---

# 13. Stop a Container

```powershell
docker stop orders-api
```

Stops the container.

But the container still exists.

```text
STOP
 ↓
Container still exists
```

---

# 14. Start a Stopped Container

```powershell
docker start orders-api
```

Starts the existing container again.

Important:

```text
stop
 ↓
start
```

does not create a new container.

The same container is reused.

---

# 15. Remove a Container

```powershell
docker rm orders-api
```

Removes the container.

If it is still running:

```powershell
docker rm -f orders-api
```

`-f` means force removal.

Important:

```text
docker rm
    ↓
Container removed

Docker image
    ↓
Still exists
```

Therefore another container can still be created from the same image.

---

# 16. Image vs Container — Lifecycle

```text
Dockerfile
    ↓
docker build
    ↓
IMAGE
    ↓
docker run
    ↓
CONTAINER
    ↓
docker stop
    ↓
Stopped container
    ↓
docker start
    ↓
Running again
    ↓
docker rm
    ↓
Container removed
```

The image remains unless explicitly removed.

---

# 17. Execute Commands Inside a Container

```powershell
docker exec -it orders-api /bin/sh
```

Breakdown:

```text
docker exec
    → Execute command inside a running container

-i
    → Interactive

-t
    → Allocate terminal

/bin/sh
    → Start shell
```

Exit:

```sh
exit
```

---

# 18. Container Hostname

Inside the container:

```sh
hostname
```

Example:

```text
852103db9aa3
```

This is the container's hostname.

---

# 19. `/etc/hosts`

Inside the container:

```sh
cat /etc/hosts
```

Typical entries include:

```text
127.0.0.1       localhost
::1             localhost
172.17.0.2      <container-hostname>
```

---

# 20. IPv4 and IPv6 Loopback

```text
127.0.0.1
```

is the IPv4 loopback address.

```text
::1
```

is the IPv6 loopback address.

They both mean:

```text
"This machine/container itself"
```

Inside a container:

```text
localhost
127.0.0.1
::1
```

refer to the container itself.

---

# 21. What Does `::1` Mean?

IPv6 addresses contain eight groups of hexadecimal numbers.

The IPv6 loopback address is:

```text
0000:0000:0000:0000:0000:0000:0000:0001
```

IPv6 allows consecutive zero groups to be compressed:

```text
::1
```

Therefore:

```text
127.0.0.1
```

is the IPv4 loopback equivalent of:

```text
::1
```

for IPv6.

---

# 22. Important `localhost` Concept

Suppose we have:

```text
orders-api container
redis container
```

If the API executes:

```text
localhost:6379
```

it means:

```text
orders-api container
       ↑
     itself
```

It does NOT mean:

```text
redis container
```

Therefore this is incorrect for container-to-container communication:

```text
localhost:6379
```

Instead, we use the other container's network name:

```text
redis:6379
```

---

# 23. Docker Network

Create a custom Docker network:

```powershell
docker network create orders-network
```

The network provides a private communication environment for containers attached to it.

---

# 24. List Docker Networks

```powershell
docker network ls
```

Shows available Docker networks.

---

# 25. Inspect a Docker Network

```powershell
docker network inspect orders-network
```

Shows detailed information including connected containers.

---

# 26. Run Redis on the Network

```powershell
docker run -d --name redis --network orders-network redis
```

Important:

```text
--network orders-network
```

connects Redis to our custom network.

---

# 27. Run the API on the Same Network

```powershell
docker run -d --name orders-api --network orders-network -p 8080:8080 orders-api:day1
```

Now:

```text
┌─────────────────────────────────────┐
│       orders-network                │
│                                     │
│   ┌─────────────┐                   │
│   │ orders-api  │                   │
│   └──────┬──────┘                   │
│          │                          │
│          │ redis:6379               │
│          ▼                          │
│   ┌─────────────┐                   │
│   │    redis    │                   │
│   └─────────────┘                   │
│                                     │
└─────────────────────────────────────┘
```

---

# 28. Container-to-Container DNS

Inside the API container:

```powershell
docker exec -it orders-api /bin/sh
```

Then:

```sh
getent hosts redis
```

This asks the container's hostname-resolution mechanism:

```text
"What IP address does redis resolve to?"
```

Conceptually:

```text
redis
   ↓
Docker DNS
   ↓
Redis container IP
```

Therefore the API can connect using:

```text
redis:6379
```

instead of:

```text
172.17.x.x:6379
```

---

# 29. Why Container Names Are Better Than IP Addresses

Avoid:

```text
redis → 172.18.0.3
```

Prefer:

```text
redis → redis:6379
```

Why?

Container IP addresses can change.

The logical name remains stable.

This introduces the concept of:

```text
Logical service name
        ↓
Service discovery
        ↓
Current service instance
```

This will later map directly to Kubernetes:

```text
Kubernetes Service
        ↓
Kubernetes DNS
        ↓
Pods
```

---

# 30. `getent hosts`

Command:

```sh
getent hosts redis
```

Purpose:

Ask the system's configured name-resolution mechanisms to resolve the hostname `redis`.

In our Docker network:

```text
redis
  ↓
Docker embedded DNS
  ↓
Redis container IP
```

This is a useful diagnostic command for container networking.

---

# 31. Inspect a Container

```powershell
docker inspect orders-api
```

Useful for troubleshooting:

- Container ID
- Image
- Network configuration
- IP address
- Environment
- Mounts
- State
- Configuration

When something isn't behaving as expected, `docker inspect` is one of the first commands to consider.

---

# 32. Inspect Image History

```powershell
docker history orders-api:day1
```

Shows the layers/instructions that contributed to the image.

Conceptually:

```text
Dockerfile
    ↓
FROM
    ↓
WORKDIR
    ↓
COPY
    ↓
RUN
    ↓
ENTRYPOINT
```

This becomes important for:

- Image size
- Build caching
- Build performance
- Dockerfile optimization

---

# 33. Why the Initial Dockerfile Isn't Ideal

Our first Dockerfile used:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0
```

for both:

```text
BUILD
+
RUNTIME
```

The SDK contains tools such as:

- Compiler
- MSBuild
- NuGet tooling
- Build utilities

The production application doesn't need these tools to run.

Therefore:

```text
SDK image
    ↓
Large
    ↓
Unnecessary runtime contents
    ↓
Larger attack surface
```

The better approach is to separate:

```text
BUILD ENVIRONMENT
```

from:

```text
RUNTIME ENVIRONMENT
```

---

# 34. Multi-Stage Docker Build

Use separate build and runtime stages:

```dockerfile
# Stage 1: Build

FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build

WORKDIR /src

COPY . .

RUN dotnet publish -c Release -o /app/publish


# Stage 2: Runtime

FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime

WORKDIR /app

COPY --from=build /app/publish .

ENTRYPOINT ["dotnet", "Orders.Api.dll"]
```

Build:

```powershell
docker build -t orders-api:multistage .
```

---

# 35. Multi-Stage Build Mental Model

```text
              BUILD STAGE
        ┌──────────────────────┐
        │ .NET SDK             │
        │ Compiler             │
        │ MSBuild              │
        │ NuGet                │
        │ Source code          │
        └──────────┬───────────┘
                   │
                   │ dotnet publish
                   ▼
             Published files
                   │
                   ▼
              RUNTIME STAGE
        ┌──────────────────────┐
        │ ASP.NET Runtime      │
        │ Published app        │
        │                      │
        │ No SDK               │
        │ No compiler          │
        │ No build tools       │
        └──────────────────────┘
```

---

# 36. `COPY --from`

This line is critical:

```dockerfile
COPY --from=build /app/publish .
```

It means:

```text
Copy files from the "build" stage
        ↓
From:
/app/publish

        ↓

Into:
current directory of runtime stage
```

The build environment does not become part of the final image.

---

# 37. Benefits of Multi-Stage Builds

### Smaller images

Only runtime dependencies are included.

### Reduced attack surface

Build tools don't need to be present in the production container.

### Faster image transfers

Smaller images require less data to push and pull.

### Better deployment efficiency

Especially useful when:

- Deploying to Kubernetes
- Scaling containers
- Pulling images onto new nodes
- Using CI/CD pipelines

### Separation of concerns

```text
Build environment
        ≠
Runtime environment
```

---

# 38. Container Filesystem

Containers have their own writable filesystem layer.

Experiment:

```powershell
docker run -it --name test-container alpine sh
```

Inside:

```sh
echo "hello docker" > /data.txt
```

Check:

```sh
cat /data.txt
```

Expected:

```text
hello docker
```

Exit:

```sh
exit
```

---

# 39. Stop vs Remove — Filesystem Experiment

Start the same container:

```powershell
docker start test-container
```

Check:

```powershell
docker exec test-container cat /data.txt
```

The file still exists.

Why?

```text
docker stop
    ↓
Container still exists
    ↓
Writable filesystem still exists
```

---

# 40. Remove the Container

```powershell
docker rm -f test-container
```

Create a new container:

```powershell
docker run -it --name test-container alpine sh
```

Try:

```sh
cat /data.txt
```

The file is gone.

Why?

```text
Container removed
      ↓
Container writable layer removed
      ↓
Data lost
```

---

# 41. Docker Volume

Create a named volume:

```powershell
docker volume create orders-data
```

A volume exists independently of a particular container.

---

# 42. Run a Container Using a Volume

```powershell
docker run -it --name test-container --mount source=orders-data,target=/data alpine sh
```

Inside:

```sh
echo "hello persistent storage" > /data/message.txt
```

Exit:

```sh
exit
```

The data is stored in the Docker volume.

---

# 43. Remove the Container

```powershell
docker rm -f test-container
```

The volume remains.

---

# 44. Reuse the Volume

Create another container using the same volume:

```powershell
docker run -it --name test-container2 --mount source=orders-data,target=/data alpine sh
```

Check:

```sh
cat /data/message.txt
```

Expected:

```text
hello persistent storage
```

This demonstrates that the data belongs to the volume rather than the container.

---

# 45. Container Storage vs Volume Storage

## Container filesystem

```text
Container
    ↓
Writable layer
    ↓
Container removed
    ↓
Data lost
```

## Docker Volume

```text
Container
    ↓
Docker Volume
    ↓
Container removed
    ↓
Volume remains
    ↓
New container can reuse it
```

---

# 46. List Volumes

```powershell
docker volume ls
```

Shows Docker volumes.

---

# 47. Inspect a Volume

```powershell
docker volume inspect orders-data
```

Shows detailed information about the volume.

---

# 48. Core Day 1 Commands

These are the commands worth remembering.

## Images

```powershell
docker images
docker build -t orders-api:day1 .
docker history orders-api:day1
```

## Containers

```powershell
docker run
docker ps
docker ps -a
docker stop
docker start
docker rm
docker exec
docker inspect
```

## Networks

```powershell
docker network create orders-network
docker network ls
docker network inspect orders-network
```

## Volumes

```powershell
docker volume create orders-data
docker volume ls
docker volume inspect orders-data
```

---

# 49. Day 1 Command Flow

The overall workflow we practiced:

```text
Create .NET application
        ↓
dotnet run
        ↓
Verify application
        ↓
Create Dockerfile
        ↓
docker build
        ↓
Docker Image
        ↓
docker run
        ↓
Container
        ↓
docker ps
        ↓
Inspect container
        ↓
Create Docker network
        ↓
Run Redis
        ↓
Run API
        ↓
Docker DNS
        ↓
redis:6379
        ↓
Test container filesystem
        ↓
Understand ephemeral storage
        ↓
Create Docker volume
        ↓
Persistent data
        ↓
Multi-stage Docker build
```

---

# 50. Day 1 Architecture Mental Model

```text
                         SOURCE CODE
                             │
                             ▼
                         Dockerfile
                             │
                       docker build
                             │
                             ▼
                    ┌─────────────────┐
                    │  Docker Image   │
                    │                 │
                    │ App             │
                    │ Runtime         │
                    │ Dependencies    │
                    └────────┬────────┘
                             │
                         docker run
                             │
                             ▼
                    ┌─────────────────┐
                    │   Container     │
                    │                 │
                    │ App Process     │
                    │ Filesystem      │
                    │ Network         │
                    └───────┬─────────┘
                            │
             ┌──────────────┴──────────────┐
             │                             │
             ▼                             ▼
        Networking                     Storage
             │                             │
             ▼                             ▼
       Docker DNS                 Docker Volume
             │                             │
             ▼                             │
      Container name                       │
             │                             │
             ▼                             │
      Container IP                  Persistent Data
```

---

# 51. Architect-Level Questions

## Question 1 — Image vs Container

### What is the difference between an image and a container?

Answer:

```text
Image
    = Immutable package/template

Container
    = Running instance of an image
```

Multiple containers can be created from the same image.

---

## Question 2 — Port Mapping

### What does this mean?

```text
-p 9000:8080
```

Answer:

```text
Host port 9000
       ↓
Container port 8080
```

---

## Question 3 — Container DNS

### Why can the API use `redis:6379`?

Because containers attached to the same user-defined Docker network can resolve container names using Docker's internal DNS.

```text
redis
  ↓
Docker DNS
  ↓
Redis container IP
```

---

## Question 4 — localhost

### What does localhost mean inside a container?

It refers to the container itself.

```text
localhost
    =
127.0.0.1
    =
this container
```

It does not refer to another container.

---

## Question 5 — Multi-Stage Build

### Why use multi-stage builds?

Because build tools are required to compile/publish the application but aren't required to run the application.

Therefore:

```text
SDK image
    ↓
Build

ASP.NET runtime image
    ↓
Run
```

Benefits:

- Smaller image
- Reduced attack surface
- Less unnecessary software
- More efficient deployment

---

## Question 6 — Persistence

### What happens to data in the container filesystem when the container is removed?

Data in the container's writable layer is removed with the container.

Persistent data should be placed in a volume.

```text
Container filesystem
    → tied to container

Volume
    → independent of container
```

---

# 52. Important Day 1 Concepts to Retain

Do not try to memorize every Docker command.

Make sure these concepts are solid.

## Image → Container

```text
Image
    ↓
Container
```

---

## Host Port → Container Port

```text
Host Port
    ↓
Container Port
```

Example:

```text
9000 → 8080
```

---

## Container → Network

```text
Container
    ↓
Docker Network
```

---

## Container Name → DNS → IP

```text
Container Name
       ↓
Docker DNS
       ↓
Container IP
```

---

## localhost

```text
localhost
    ↓
Current container
```

---

## Build vs Runtime

```text
SDK image
    ↓
Build stage

Runtime image
    ↓
Production stage
```

---

## Ephemeral vs Persistent Storage

```text
Container filesystem
    ↓
Ephemeral

Docker Volume
    ↓
Persistent
```

---

# 53. Connection to Kubernetes

The concepts learned today map directly to Kubernetes.

## Docker Image → Kubernetes Pod

Conceptually:

```text
Docker Image
       ↓
Kubernetes
       ↓
Pod
       ↓
Container
```

A Pod is not simply the same thing as a Docker container, but it is the Kubernetes unit that runs one or more containers.

---

## Docker Networking → Kubernetes Networking

```text
Docker Network
       ↓
Kubernetes Network
```

Kubernetes provides networking between Pods and higher-level Service abstractions.

---

## Docker DNS → Kubernetes Service DNS

Docker:

```text
redis:6379
```

Kubernetes:

```text
redis-service:6379
```

or, using the fully qualified service DNS name:

```text
redis-service.namespace.svc.cluster.local
```

The fundamental idea is the same:

```text
Logical name
    ↓
DNS/service discovery
    ↓
Current destination
```

---

## Docker Volume → Kubernetes Persistent Storage

Docker:

```text
Docker Volume
```

Kubernetes:

```text
PersistentVolume
       ↓
PersistentVolumeClaim
```

The Kubernetes model provides a more sophisticated abstraction for persistent storage.

---

# 54. Day 1 Final Mental Model

```text
                    Dockerfile
                         │
                         │ docker build
                         ▼
                    Docker Image
                         │
                         │ docker run
                         ▼
                     Container
                         │
            ┌────────────┴────────────┐
            │                         │
            ▼                         ▼
         Network                   Storage
            │                         │
            ▼                         ▼
       Docker DNS                 Volume
            │                         │
            ▼                         │
      Container name                  │
            │                         │
            ▼                         │
       Container IP            Persistent Data
```

---

# 55. Day 1 Architect Summary

The most important architectural ideas from Day 1 are:

### 1. Images are immutable packages

```text
Dockerfile
    ↓
Image
```

### 2. Containers are runtime instances

```text
Image
    ↓
Container
```

### 3. Containers are isolated

A container has its own:

- Process space
- Filesystem
- Network namespace

### 4. Container networking should use logical names

Prefer:

```text
redis:6379
```

over:

```text
172.18.0.5:6379
```

### 5. localhost means the current container

```text
localhost
    =
this container
```

### 6. Build and runtime environments should be separated

```text
.NET SDK
    ↓
Build

ASP.NET Runtime
    ↓
Run
```

### 7. Container filesystem is not durable storage

For persistent data:

```text
Container
    ↓
Volume
```

---

# 56. Day 1 Completion Checklist

- [x] Create ASP.NET Core Web API
- [x] Run .NET application locally
- [x] Understand Dockerfile
- [x] Understand `FROM`
- [x] Understand `WORKDIR`
- [x] Understand `COPY`
- [x] Understand `RUN`
- [x] Understand `ENTRYPOINT`
- [x] Build Docker image
- [x] Run Docker container
- [x] Understand image vs container
- [x] Understand port mapping
- [x] Understand container lifecycle
- [x] Use `docker ps`
- [x] Use `docker ps -a`
- [x] Stop/start/remove containers
- [x] Use `docker exec`
- [x] Inspect container hostname
- [x] Understand `/etc/hosts`
- [x] Understand `127.0.0.1`
- [x] Understand `::1`
- [x] Understand `localhost`
- [x] Create Docker network
- [x] Connect containers to a network
- [x] Run Redis
- [x] Understand container-to-container communication
- [x] Understand Docker DNS
- [x] Use `getent hosts`
- [x] Inspect Docker networks
- [x] Understand multi-stage builds
- [x] Use `COPY --from`
- [x] Understand container filesystem
- [x] Understand ephemeral storage
- [x] Create Docker volume
- [x] Mount a volume
- [x] Verify persistence after container removal
- [x] Complete architect checkpoint

---

# Day 1 → Day 2

Day 1 established the basic Docker runtime model.

Day 2 will build on it:

```text
DAY 1
────────────────────────────
Application
Dockerfile
Image
Container
Networking
DNS
Volumes
Multi-stage builds
             │
             ▼
DAY 2
────────────────────────────
Image Layers
Layer Caching
Docker Build Cache
.dockerignore
Dockerfile Optimization
Efficient .NET Builds
```

The key Day 2 question:

> Why can changing one line of application code cause Docker to rebuild much more than necessary?

This leads into **image layers and Docker build caching**, which are particularly important for CI/CD and production container builds.