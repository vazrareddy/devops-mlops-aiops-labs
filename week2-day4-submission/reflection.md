# Week 2 Day 4 — Reflection

## 1. Why does the bootcamp deliberately do this with the console before Terraform? What would change if you did it with Terraform from day one?

The bootcamp intentionally starts with the AWS Console because Infrastructure as Code (IaC) is only effective when the engineer understands the underlying resources being automated.

When creating infrastructure manually, I am forced to understand:

* Which resources are required
* How resources depend on each other
* What networking components are created
* Which security groups allow specific traffic flows
* How Route Tables, NAT Gateways, ALBs, Target Groups, and Auto Scaling Groups interact

For example, when building the ALB manually, I could directly observe:

```text
Internet
    ↓
ALB
    ↓
Target Group
    ↓
Auto Scaling Group
    ↓
EC2 Instances
```

Likewise, creating RDS manually made it clear why a DB subnet group requires subnets in multiple Availability Zones and why the database should not be internet accessible.

If Terraform had been introduced on day one, infrastructure creation would have been much faster, but much of the learning would have become abstract. I could deploy dozens of resources without truly understanding their purpose or dependencies. In production, this creates engineers who can run Terraform but struggle to troubleshoot when something breaks.

The console-first approach builds a mental model of AWS. Terraform then becomes a tool for expressing that model in code.

A senior engineer should be able to answer both:

1. "What does this Terraform create?"
2. "What AWS resources does this architecture actually contain?"

The console teaches the second. Terraform teaches the first.

---

## 2. In your own words, what is the difference between the EC2 health check and the ELB health check on an ASG? Which one would have replaced an instance with a hung gunicorn process?

Although both health checks can trigger instance replacement, they measure very different things.

### EC2 Health Check

The EC2 health check asks:

> "Is the virtual machine itself healthy?"

It verifies:

* Hypervisor health
* Network availability
* Instance status checks
* Operating system responsiveness

An EC2 instance can pass EC2 health checks even when the application running on it is completely broken.

Example:

```text
EC2 Running
Linux Running
CPU Normal

Gunicorn Crashed
Application Down
```

EC2 health checks would still pass.

---

### ELB Health Check

The ELB health check asks:

> "Can the application actually serve requests?"

For this lab:

```text
Path: /login
Port: 8000
Expected: HTTP 200
```

The load balancer continuously sends requests to the application.

If Gunicorn crashes, hangs, or becomes unable to respond:

```text
GET /login
Timeout
```

the target becomes unhealthy.

The Auto Scaling Group then replaces the instance.

---

### Which health check would replace a hung Gunicorn process?

The ELB health check.

A hung Gunicorn process is an application failure, not an infrastructure failure.

The EC2 instance would still be:

```text
Running
Reachable
Passing EC2 checks
```

but the ALB would mark it unhealthy because the application endpoint no longer responds correctly.

This distinction is important in production because most outages are application-level failures, not VM failures.

---

## 3. The ALB security group allows 0.0.0.0/0 on 443, but the ASG security group only allows 8000 from the ALB security group. Walk through what this restriction actually protects against.

This design implements a security principle called **tier isolation**.

### ALB Security Group

```text
Inbound:
443 from 0.0.0.0/0
```

This allows anyone on the internet to access the load balancer.

That is expected because the ALB is the public entry point into the application.

---

### ASG Security Group

```text
Inbound:
8000 from ALB Security Group only
```

Notice that it does not allow:

```text
8000 from 0.0.0.0/0
```

or

```text
8000 from the VPC CIDR
```

Only the ALB is trusted to communicate with the application servers.

---

### What does this protect against?

Without this restriction:

```text
Internet
   ↓
EC2 Instance Port 8000
```

an attacker could bypass:

* ALB health checks
* TLS termination
* WAF (if attached later)
* Request filtering
* Centralized logging

and communicate directly with the application.

With the restriction:

```text
Internet
   ↓
ALB (443)
   ↓
EC2 (8000)
```

the application instances are effectively hidden behind the load balancer.

Even if someone discovers the private IP of an instance:

```text
10.0.4.6
```

they still cannot connect because the security group blocks the traffic.

This architecture reduces the attack surface and enforces a single controlled entry point into the application.

---

## 4. A teammate proposes putting the database in a private subnet of the same route table as the app tier (skipping the dedicated rds-1/rds-2 subnets). What's lost? Reference NACLs in your answer.

Technically, the database would still function.

However, significant security and operational isolation would be lost.

---

### Current Design

```text
App Subnets
10.0.3.0/24
10.0.4.0/24

RDS Subnets
Separate subnet group
Separate routing boundary
Separate NACL opportunity
```

This creates a dedicated database network layer.

---

### If App and DB Share Subnets

```text
Private Subnet
├── EC2 Application
└── RDS Database
```

Both workloads now share:

* Route tables
* Network ACLs
* Network boundaries

---

### Why NACL Separation Matters

Security Groups are stateful and operate at the instance level.

NACLs are stateless and operate at the subnet level.

With dedicated RDS subnets I can create a database-specific NACL such as:

```text
Allow:
5432 from App Subnets

Deny:
Everything Else
```

This provides an additional network security layer.

If application and database resources share the same subnet, NACLs can no longer distinguish between them.

Any NACL rule affects both tiers simultaneously.

This removes the ability to enforce subnet-level segmentation.

---

### Operational Impact

Dedicated RDS subnets provide:

* Better network isolation
* Easier compliance audits
* Cleaner security boundaries
* Independent NACL policies
* Clear separation of application and data tiers

Shared subnets create tighter coupling between resources and reduce defense-in-depth.

In small environments this may appear simpler, but in production architectures the separation of application and database tiers is considered a best practice because it improves both security and operational flexibility.

---

# Final Takeaway

The primary lesson from this lab is that high availability is not achieved by a single AWS service. It emerges from multiple layers working together:

* Multi-AZ networking
* Security group segmentation
* Dedicated database subnets
* Application-aware health checks
* Load balancing
* Auto Scaling
* DNS and certificate management

Each layer protects against a different failure mode. The architecture becomes resilient because no single component is trusted to provide availability by itself.

