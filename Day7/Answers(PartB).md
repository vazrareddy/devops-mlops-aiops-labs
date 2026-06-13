# Part B Reflection Questions

## 1. Why did we create the log group BEFORE the task instead of letting ECS create it?

Creating the CloudWatch Log Group before launching the ECS task ensures that logging infrastructure is available when the container starts. During task startup, the ECS agent immediately attempts to send container logs to CloudWatch. If the log group does not exist and the execution role lacks permissions to create it, task startup can fail or logs may be lost.

Pre-creating the log group also allows engineers to configure retention policies, access controls, naming conventions, and monitoring standards in advance. In production environments, logging resources are typically provisioned through Infrastructure as Code (Terraform or CloudFormation) rather than being created automatically at runtime.

In short, creating the log group beforehand improves reliability, security, and operational consistency.

---

## 2. What's the difference between Task Role and Task Execution Role?

Although both are IAM roles, they serve different purposes.

### Task Execution Role

The Task Execution Role is used by the ECS platform itself.

It allows ECS to:

* Pull container images from ECR
* Push logs to CloudWatch
* Retrieve secrets from AWS Secrets Manager
* Access Parameter Store values

Example:

When the nginx task starts, ECS uses the execution role to pull the container image and send logs to CloudWatch.

### Task Role

The Task Role is used by the application running inside the container.

It allows the application to interact with AWS services.

Example:

A Python application inside a container may need to:

* Read files from S3
* Publish messages to SQS
* Write records to DynamoDB

Those permissions belong to the Task Role.

### Summary

Task Execution Role:

```text
ECS Platform → AWS Services
```

Task Role:

```text
Application Container → AWS Services
```

A common interview mistake is assuming they are interchangeable. They are not.

---

## 3. Why doesn't a stopped task auto-restart? What would change that behavior?

An ECS Task is a one-time execution unit. ECS launches the task, runs it, and considers its job complete when it stops.

ECS does not automatically restart standalone tasks because there is no desired state defined.

To enable automatic recovery, the task must be managed by an ECS Service.

### Task

```text
Desired Count = Not Defined
```

If it stops:

```text
No replacement created
```

### Service

```text
Desired Count = 1
```

If the task crashes:

```text
ECS launches a replacement task automatically
```

This is called self-healing.

Production workloads are almost always deployed through ECS Services rather than standalone Tasks because services maintain desired state and high availability.

---

## 4. If you needed to run the same nginx container on your own EC2 instances instead of Fargate, what would change in the task definition?

The container definition would remain largely unchanged because nginx is still the workload being executed. However, the infrastructure model changes significantly.

### Fargate

AWS manages:

* EC2 instances
* Operating system patching
* Capacity provisioning
* Cluster infrastructure

The task definition only specifies:

* CPU
* Memory
* Image
* Networking
* Logging

### ECS on EC2

The organization must manage:

* EC2 instances
* ECS container agents
* Capacity planning
* OS patching
* Auto Scaling Groups

The task definition may also require host-level considerations such as port conflicts, placement constraints, and resource allocation based on available EC2 capacity.

In summary, the application remains the same, but operational responsibility shifts from AWS to the engineering team.

---

## 5. The container port is 80. There's no host port mapping like in Docker Compose. Why?

In Docker Compose, containers share the host machine's network namespace through port mappings.

Example:

```text
Host Port 8080 → Container Port 80
```

This is necessary because multiple containers run on the same server.

In ECS Fargate, each task receives its own Elastic Network Interface (ENI) and private IP address. The container is directly reachable through its network interface without sharing a host's network stack.

Instead of:

```text
Host:8080 → Container:80
```

we have:

```text
Task IP → Container:80
```

Because every task has its own network identity, host port mappings are usually unnecessary.

This networking model improves isolation, simplifies service discovery, and eliminates many port collision problems commonly seen in traditional container deployments.
