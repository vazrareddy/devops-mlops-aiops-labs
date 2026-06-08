# Week 2 Day 4 - answers.md

## Part 1 - RDS Architecture Selection

| Scenario                                                   | Recommended Option | Reason                                                                                                         |
| ---------------------------------------------------------- | ------------------ | -------------------------------------------------------------------------------------------------------------- |
| Personal blog (200 visits/day)                             | Single-AZ          | Lowest cost solution. Downtime is acceptable for a personal application.                                       |
| Hospital OT scheduling app                                 | Multi-AZ Standby   | High availability is critical. Automatic failover minimizes downtime without requiring application changes.    |
| E-commerce site with Friday traffic spikes                 | Multi-AZ Cluster   | Read replicas handle heavy read traffic while maintaining high availability.                                   |
| Internal reporting tool used only 30 minutes per day       | Aurora Serverless  | Automatically scales down during idle periods, reducing infrastructure costs.                                  |
| Discord-like product requiring millions of small databases | Not RDS            | Requires large-scale distributed architecture and application-level sharding. Traditional RDS is not suitable. |

### Why are databases single-writer by default?

Most relational databases are designed around ACID guarantees (Atomicity, Consistency, Isolation, Durability). Allowing multiple writers across independent nodes creates consistency conflicts, transaction ordering issues, and data corruption risks.

For this reason, databases typically use a single primary writer while replicas handle read traffic. This model simplifies consistency and guarantees predictable transaction behavior.

### What problem does sharding solve?

Sharding distributes data across multiple database servers.

Example:

* Users 1–1,000,000 → Shard A
* Users 1,000,001–2,000,000 → Shard B

Benefits:

* Removes single database bottlenecks
* Increases horizontal scalability
* Improves performance
* Reduces storage pressure on a single database

Sharding is usually implemented at the application layer because the application understands business logic and user distribution patterns better than infrastructure services.

---

## Part 2.4 - Storage, IOPS and Encryption

### 1. Why does disk matter more than CPU for write-heavy databases?

For write-intensive workloads, storage performance is often the primary bottleneck.

Even if CPU and memory are sufficient, slow storage can delay transaction commits, increase query latency, and reduce throughput.

Databases continuously perform:

* INSERT operations
* UPDATE operations
* WAL (Write-Ahead Log) writes
* Index updates

If storage cannot process these operations fast enough, overall database performance degrades significantly.

### What is IOPS?

IOPS (Input/Output Operations Per Second) measures how many read or write operations a storage device can perform every second.

Examples:

* 3,000 IOPS = 3,000 operations/sec
* 30,000 IOPS = 30,000 operations/sec

Higher IOPS generally results in:

* Faster transactions
* Lower latency
* Better database performance

### Why does IOPS scale with disk size?

Larger storage volumes typically provide more storage blocks and greater parallelism.

As disk size increases, cloud providers can allocate additional throughput and IOPS capacity, enabling higher performance.

---

### 2. What is connection pooling?

Connection pooling reuses existing database connections instead of creating new connections for every request.

Without pooling:

* 1,000 connections
* ~10 MB memory per connection

Memory usage:

1000 × 10 MB = ~10 GB RAM

With a 50-connection pool:

50 × 10 MB = ~500 MB RAM

Benefits:

* Lower memory consumption
* Faster response times
* Better scalability
* Reduced database overhead

---

### 3. Encryption at Rest

#### AWS Managed KMS Key

Use Case:

* Standard enterprise applications
* Minimal operational effort

Advantages:

* Managed by AWS
* Automatic rotation
* Easy implementation

#### Customer Managed KMS Key

Use Case:

* Compliance-driven environments
* Fine-grained access control requirements

Advantages:

* Custom IAM permissions
* Custom rotation policies
* Detailed auditing

#### CloudHSM Imported Key

Use Case:

* Banking
* Government
* Military
* Highly regulated industries

Advantages:

* Full ownership of cryptographic material
* AWS never controls the key

---

## Part 6.1 - Why use a dedicated /health endpoint?

Using application endpoints such as /login for health checks is not ideal because business logic changes can accidentally break health checks.

A dedicated /health endpoint should:

* Return HTTP 200
* Execute lightweight checks
* Avoid expensive operations
* Minimize database dependencies

Example:

A /health endpoint can validate application readiness without requiring user authentication or complex workflows.

---

## Part 7.2 - Route 53 Routing Policies

### Simple Routing

Use Case:

Single application hosted behind one load balancer.

### Weighted Routing

Use Case:

Canary deployments.

Example:

* Version A = 90%
* Version B = 10%

### Latency Routing

Use Case:

Global applications serving users from the nearest AWS region.

### Failover Routing

Use Case:

Disaster recovery architecture.

Primary region failure automatically redirects traffic to the secondary region.

### Geolocation Routing

Use Case:

Country-specific content delivery.

Example:

* India → Indian website
* USA → US website

### Geoproximity Routing

Use Case:

Traffic routed according to physical proximity to AWS resources.

### Multivalue Routing

Use Case:

Returns multiple healthy endpoints for basic load balancing and redundancy.

### IP-Based Routing

Use Case:

Enterprise applications routing users based on source IP ranges.

---

## Part 8.3 - Scaling Concepts

### Why is CPU-based scaling often insufficient?

CPU utilization is only one indicator of system load.

#### Uber Pattern

Ride demand spikes before CPU usage increases.

Scaling should be based on business metrics such as:

* Active riders
* Active drivers
* Ride requests

#### Medium Pattern

Traffic spikes after a viral article.

Scaling should consider:

* Requests per second
* ALB request count

instead of CPU alone.

#### Black Friday Pattern

Traffic surge is predictable.

Scheduled scaling is more effective than reactive CPU scaling.

---

### What is Scheduled Scaling?

Scheduled scaling adjusts capacity at predefined times.

Example:

Every Friday at 6 PM:

* Minimum Capacity = 5
* Desired Capacity = 10

Advantages:

* Capacity is available before traffic arrives.
* Eliminates cold-start delays.

---

### What are Warmup and Cooldown periods?

#### Warmup

Warmup gives new instances time to initialize.

During warmup, instances may:

* Install packages
* Execute user-data scripts
* Start application services

Without warmup, Auto Scaling may make incorrect scaling decisions.

#### Cooldown

Cooldown prevents rapid scaling fluctuations.

Benefits:

* Prevents scaling loops
* Stabilizes infrastructure
* Reduces unnecessary instance launches

A 100-second warmup is important because user-data execution, dependency installation, and application startup require time before an instance can serve production traffic.

---

# Reflection Answers

## Why learn AWS Console before Terraform?

Learning through the AWS Console helps engineers understand how resources interact behind the scenes.

Terraform automates infrastructure provisioning, but it does not eliminate the need to understand:

* VPCs
* Route tables
* Security groups
* Load balancers
* Auto Scaling Groups

Understanding the manual process improves troubleshooting and architectural decision-making.

---

## EC2 Health Check vs ELB Health Check

### EC2 Health Check

Validates:

* Instance availability
* Hypervisor health
* Operating system responsiveness

Does not verify application health.

### ELB Health Check

Validates:

* Application availability
* Endpoint response codes
* Service readiness

If Gunicorn crashes:

* EC2 Health Check = Healthy
* ELB Health Check = Unhealthy

ELB health checks would trigger instance replacement.

---

## Why allow port 8000 only from the ALB Security Group?

This design enforces layered security.

Traffic flow:

User → ALB → EC2 Instance

Benefits:

* Prevents direct internet access to application instances
* Reduces attack surface
* Centralizes TLS termination
* Ensures all traffic passes through the load balancer

---

## Why use dedicated RDS subnets?

Dedicated RDS subnets improve isolation between application and database layers.

Benefits:

* Separate network boundaries
* Dedicated NACL management
* Improved security posture
* Reduced blast radius
* Better compliance and auditing controls

This follows enterprise-grade cloud architecture best practices.

