# Docker Exercises – Day 2 Answers

## Exercise 1 – The Data Problem (Why Volumes Exist)

### Why did the file disappear when you created a new container?

Containers use ephemeral storage. Any data written inside a container exists only in that container's writable layer. When the container is removed, the writable layer is deleted along with all data stored inside it.

### In production, what kind of data would you lose if you relied on container storage alone?

* Application logs
* User uploads
* Database files
* Session data
* Generated reports
* Temporary business data

---

## Exercise 2 – Host Path Mount

### When would you use a read-only mount in production?

Read-only mounts are used when containers need access to files but should not be allowed to modify them.

Examples:

* Nginx configuration files
* TLS certificates
* Application configuration files
* Static content

### What is the difference between `-v ~/datapoints:/data` and `-v ~/datapoints:/data:ro`?

`-v ~/datapoints:/data`

* Read and write access

`-v ~/datapoints:/data:ro`

* Read-only access
* Container cannot modify files

---

## Exercise 3 – Sharing Data Across Multiple Containers

### What problem could occur if two containers write to the same file at exactly the same time?

Simultaneous writes can cause:

* Race conditions
* Data corruption
* Partial writes
* Inconsistent application state

### In production on AWS, which service would you use to share storage across multiple containers — EBS or EFS? Why?

EFS is preferred because it provides shared network storage that can be accessed simultaneously by multiple containers and instances. EBS is typically attached to a single EC2 instance and is not designed for multi-writer access.

---

## Exercise 4 – Named Docker Volumes

### What is the advantage of `--mount` over `-v` in terms of readability?

`--mount` is more explicit and easier to understand because it clearly defines the source, target, and mount type.

Example:

```bash
--mount type=volume,source=april-batch,target=/data
```

### When would you use `type=bind` instead of `type=volume`?

Use `type=bind` when the container must access a specific directory on the host machine, such as application code, configuration files, or local development files.

### Who manages the storage path when you use a named volume — you or Docker engine?

Docker Engine manages the storage location automatically.

---

## Exercise 5 – Write Your First Dockerfile

### Why do we `COPY requirements.txt` separately before `COPY . .`?

This optimizes Docker layer caching. Dependency installation is only re-executed when requirements.txt changes, reducing rebuild times.

### What does `WORKDIR /app` do?

It sets `/app` as the default working directory for all subsequent Dockerfile instructions.

### Why is `gunicorn` used instead of running `python app.py` directly?

Gunicorn is a production-grade WSGI server that supports multiple workers, concurrency, and better request handling than Flask's built-in development server.

---

## Exercise 6 – Docker Layer Caching in Action

### Why does changing `app.py` NOT re-run `pip install` in the correct order?

Docker reuses cached layers. Since requirements.txt did not change, the dependency installation layer remains valid and is not rebuilt.

### What is the rule of thumb for ordering Dockerfile instructions to maximise caching?

Place infrequently changing instructions first and frequently changing instructions later.

Typical order:

1. FROM
2. WORKDIR
3. COPY requirements.txt
4. RUN pip install
5. COPY application code
6. CMD

---

## Exercise 7 – Compare Image Sizes

### What is the size difference between the full and slim Python image?

The full Python image is significantly larger because it includes additional packages and development tools. The slim image removes unnecessary components, resulting in a much smaller footprint.

### What is the tradeoff of using a smaller base image?

Advantages:

* Faster deployments
* Smaller attack surface
* Reduced storage usage

Disadvantages:

* Fewer debugging tools
* Additional packages may need manual installation

### A developer hands you a Dockerfile using `ubuntu` as the base for a Python app. What would you suggest and why?

I would recommend using `python:slim` because it includes Python while remaining significantly smaller and more secure than a full Ubuntu image.

---

## Exercise 8 – Run Your Container with Port Forwarding

### Why does the app work from inside the container but not from outside?

Without port mapping, the application is only accessible within the container network namespace. The host system has no route to the container port.

### In production on AWS ECS or EKS, do you use port forwarding like this? What handles external traffic instead?

No. Production environments typically use:

* Application Load Balancers (ALB)
* Network Load Balancers (NLB)
* Kubernetes Ingress Controllers
* Service Mesh solutions

These components handle external traffic routing.

---

## Exercise 9 – Override Startup with ENTRYPOINT

### When would `--entrypoint` be useful in a real ECS or Kubernetes setup?

Common use cases include:

* Database migrations
* Debugging containers
* Running maintenance scripts
* One-time administrative tasks

### What is the difference between `CMD` and `ENTRYPOINT` in a Dockerfile?

ENTRYPOINT defines the primary executable.

CMD provides default arguments for that executable.

### Should you define `ENTRYPOINT` inside the Dockerfile? Why or why not?

Yes, when the container has a single intended purpose. It ensures consistent behavior while still allowing runtime customization through CMD.

---

## Exercise 10 – Tag and Push to Docker Hub

### Why do images built on Mac M1 sometimes fail to run on EC2?

Mac M1/M2 systems use ARM64 architecture, while most EC2 instances use AMD64 architecture. Images built for one architecture may not run on another unless multi-platform builds are used.

### In a company using AWS, which private registry would you use instead of Docker Hub? What are the benefits?

Amazon Elastic Container Registry (ECR).

Benefits:

* IAM integration
* Private repositories
* Vulnerability scanning
* Native AWS integration
* Better security and governance

---

## Exercise 11 – Docker Compose: Two-Tier App

### Why does the app use `postgres` as the database hostname instead of `localhost`?

Docker Compose automatically creates internal DNS entries using service names. The app container can resolve `postgres` directly through Docker's embedded DNS.

### What does the `volumes` block at the bottom of the file do — and why do you also reference the volume inside the postgres service?

The top-level volumes block creates and manages the volume.

The service-level reference mounts that volume into the PostgreSQL container.

### Why is `restart: on-failure:3` set to 3 and not unlimited?

Limiting retries prevents infinite restart loops and avoids wasting system resources when a persistent failure occurs.

### Why is Docker Compose not recommended for production?

Docker Compose lacks:

* Auto-scaling
* Self-healing
* Rolling updates
* Advanced scheduling
* High availability features

Production environments typically use ECS or Kubernetes.

---

## Exercise 12 – Simulate the Startup Race Condition

### Why did the application fail when the healthcheck was removed?

The application attempted to connect before PostgreSQL was fully initialized and ready to accept connections.

### What does `pg_isready` actually check inside the postgres container?

It verifies that PostgreSQL is running and capable of accepting client connections.

### What is `start_period` in the healthcheck configuration used for?

It provides an initial grace period before healthcheck failures are counted, allowing services enough time to start successfully.

---

## Key Takeaways

* Containers provide process isolation but not persistent storage.
* Volumes are required for durable application data.
* Docker layer caching significantly improves build performance.
* Smaller images improve security and deployment speed.
* Docker Compose simplifies local multi-container development.
* Health checks help prevent startup race conditions.
* Amazon ECR is preferred over Docker Hub in enterprise AWS environments.
* Kubernetes and ECS provide production-grade orchestration beyond Docker Compose.
