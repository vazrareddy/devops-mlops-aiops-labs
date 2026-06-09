# Docker Exercises – Answers.md

## Exercise 1 – Pull and Run Your First Container

### 1. Why did the container exit immediately?

A Docker container runs as long as its primary process (PID 1) is running. The BusyBox image executes its default command and exits immediately because there is no long-running foreground process to keep the container alive. Once PID 1 terminates, Docker stops the container.

### 2. What is the difference between `docker ps` and `docker ps -a`?

* `docker ps` displays only currently running containers.
* `docker ps -a` displays all containers, including running, stopped, exited, and failed containers.

---

## Exercise 2 – Run an Interactive Container

### 1. What is the difference between `docker run -it` and `docker run -dt` + `docker exec -it`?

`docker run -it` creates a new container and immediately attaches an interactive terminal session. The container typically stops when the interactive process exits.

`docker run -dt` starts the container in detached mode, allowing it to run independently in the background. `docker exec -it` attaches a new shell to an already running container without affecting the primary container process.

### 2. Why would you prefer the detached approach in a real workflow?

Detached mode is preferred because production applications run as background services. It allows containers to continue running independently of user sessions, supports multiple administrative connections, and aligns with how web servers, APIs, and databases operate in production environments.

---

## Exercise 3 – Pull and Compare Image Sizes

### 1. What is the size difference between BusyBox, Alpine, and Ubuntu?

Typical image sizes:

* BusyBox: ~5–6 MB
* Alpine: ~10–15 MB
* Ubuntu: ~80–100+ MB

The exact size varies by image version and architecture.

### 2. In production, why would you prefer a smaller base image?

Smaller images provide:

* Faster image downloads and deployments
* Reduced storage consumption
* Smaller attack surface
* Faster CI/CD pipeline execution
* Lower bandwidth usage

### 3. If a developer uses Ubuntu as the base image for a Python application, what would you suggest instead?

I would recommend using a minimal runtime image such as:

* python:slim
* python:alpine (when compatible)
* Distroless Python images

These reduce image size, improve security posture, and minimize unnecessary dependencies.

---

## Exercise 4 – Container Lifecycle Management

### 1. What does `docker ps -aq` output? How is it used in the remove command?

`docker ps -aq` returns only container IDs for all containers.

Example:

docker rm -f $(docker ps -aq)

The shell substitutes all container IDs into the command, allowing Docker to remove all containers in a single operation.

### 2. What is the difference between `docker stop` and `docker rm -f`?

`docker stop` gracefully terminates a running container by sending a SIGTERM signal and allows cleanup before stopping.

`docker rm -f` forcefully stops the container and immediately removes it, equivalent to issuing a kill followed by removal.

---

## Exercise 5 – Inspect a Container

### 1. What does "host path" mean in the context of container storage?

A host path is the actual filesystem location on the Docker host where container-related data such as logs, writable layers, bind mounts, and volumes are stored. It represents the physical storage backing the container's virtual filesystem.

### 2. What IP was assigned to the container? What network is it using by default?

The container receives a private IP address from Docker's default bridge network, typically within the 172.17.0.0/16 subnet.

Example:

IPAddress: 172.17.0.2

The container uses the default bridge network unless another network is explicitly specified.

---

## Exercise 6 – Docker Networking (Default Bridge)

### 1. Why does ping by IP work but `nslookup net2` fail?

Ping by IP works because both containers are attached to the same bridge network and can communicate using Layer 3 routing.

`nslookup net2` fails because Docker's default bridge network does not provide automatic DNS-based service discovery for container names.

### 2. What network are both containers using by default?

Both containers are connected to Docker's default bridge network named:

bridge

This network is automatically created when Docker is installed.

---

## Exercise 7 – Custom Bridge Network with DNS

### 1. What changed between Exercise 6 and this exercise that made DNS work?

A user-defined bridge network was created.

Docker automatically enables embedded DNS for user-defined bridge networks, allowing containers to resolve each other using container names instead of IP addresses.

### 2. Where do you see the container names registered in `docker inspect mynetwork`?

Container names appear under the `Containers` section of the network inspection output.

This section contains:

* Container ID
* Container Name
* IPv4 Address
* MAC Address
* Endpoint Information

Docker's embedded DNS uses this registration data for name resolution.

---

## Exercise 8 – Network Types Exploration

### 1. When would you use `--network none` in a real scenario?

`--network none` is useful when complete network isolation is required, such as:

* Security testing
* Offline workloads
* Batch processing
* Compliance-sensitive applications
* Containers that should not communicate externally

### 2. What is the `host` network type used for?

Host networking allows a container to share the host machine's network namespace.

Benefits:

* No NAT overhead
* Maximum network performance
* Direct access to host interfaces

Common use cases include monitoring agents, network troubleshooting tools, and high-performance workloads.

---

## Exercise 9 – Container Stats and Processes

### 1. How does `docker stats` relate to running `top` on a Linux machine?

`docker stats` is the container equivalent of Linux monitoring tools such as `top` or `htop`.

It provides real-time visibility into:

* CPU utilization
* Memory consumption
* Network I/O
* Block I/O
* Process counts

for individual containers.

### 2. What is `containerd`? How does it relate to Docker?

Containerd is a high-performance container runtime responsible for:

* Pulling images
* Managing container lifecycle
* Managing storage
* Managing networking integration
* Executing containers

Docker uses containerd internally to perform runtime operations.

### 3. Kubernetes uses `containerd` directly — why does this mean Kubernetes does not need the full Docker daemon?

Kubernetes only requires a CRI-compatible container runtime. Containerd already provides container lifecycle management and image handling capabilities.

Since Kubernetes communicates directly with containerd, the Docker daemon becomes unnecessary, reducing complexity, overhead, and operational dependencies.
