# Answers

## 1. Why did we create the NAT Gateway before the ECS service? What error do you see if you flip the order?

The ECS tasks are deployed into private subnets without public IP addresses. Although they don't require inbound internet connectivity, they do require outbound internet access during startup to pull container images from Amazon ECR, retrieve secrets from AWS Secrets Manager, and publish logs to Amazon CloudWatch.

The NAT Gateway provides this outbound connectivity. Creating it before the ECS service ensures that task initialization succeeds on the first deployment.

If the deployment order is reversed, ECS tasks typically fail during initialization with errors such as:

- `CannotPullContainerError`
- Timeout while pulling the image from Amazon ECR
- Failure to retrieve secrets from Secrets Manager
- Failure to publish logs to CloudWatch

The service repeatedly launches and stops tasks until the networking dependency is available.

---

## 2. Why does the target group use IP target type instead of Instance?

The application is deployed on **AWS ECS Fargate**, which is a serverless container runtime.

Unlike ECS running on EC2, Fargate tasks are not associated with EC2 instances. Each task receives its own Elastic Network Interface (ENI) and private IP address.

Therefore, the Application Load Balancer must register the individual task IP addresses as targets, making **IP** the correct target type.

The **Instance** target type is only appropriate when containers run on EC2 instances.

---

## 3. Why is the health check path `/login` and not `/`? What HTTP status code does `/` return for this app?

A health check endpoint should return a stable success response.

In this application:

- `/` returns **HTTP 302 (Redirect)** because unauthenticated users are redirected to `/login`.
- `/login` returns **HTTP 200 (OK)** consistently.

Using `/login` ensures the Application Load Balancer receives a successful response and does not incorrectly mark healthy targets as unhealthy due to redirects.

Health check endpoints should always avoid redirects and authentication dependencies whenever possible.

---

## 4. Why are ALB, ECS, and RDS using separate Security Groups instead of one large Security Group?

Each Security Group represents a different trust boundary.

- **ALB Security Group**
  - Accepts HTTP/HTTPS traffic from the Internet.

- **ECS Security Group**
  - Accepts application traffic only from the ALB Security Group.

- **RDS Security Group**
  - Accepts PostgreSQL traffic only from the ECS Security Group.

This design follows the **Principle of Least Privilege** by allowing only the minimum required communication between components.

Using one large Security Group would unnecessarily expose internal resources, increase the attack surface, and make rule management significantly more difficult.

Separating Security Groups also improves maintainability, auditing, and scalability.

---

## 5. The task definition uses `valueFrom` instead of `value` for `DB_LINK`. What's the security benefit? What permission does the execution role need?

Using `valueFrom` allows ECS to retrieve the database connection string directly from **AWS Secrets Manager** at runtime.

This provides several security advantages:

- Database credentials are never stored in plaintext inside the task definition.
- Secrets can be rotated without rebuilding the Docker image.
- Credentials remain encrypted at rest using AWS KMS.
- Access is centrally managed through IAM.

To retrieve the secret successfully, the ECS task requires permission for:

```
secretsmanager:GetSecretValue
```

on the referenced secret.

This approach aligns with cloud security best practices by separating application configuration from sensitive credentials.

---

## 6. Why does scale-in cooldown (300s) need to be longer than scale-out cooldown (60s)?

Scaling out should happen aggressively because the primary objective is maintaining application availability during sudden traffic increases.

Scaling in should be conservative because traffic spikes are often temporary.

If tasks are terminated too quickly, the application may immediately require scaling out again, causing unnecessary task creation and termination, commonly known as **thrashing**.

Using a shorter scale-out cooldown and a longer scale-in cooldown provides a more stable and cost-efficient scaling strategy.

---

## 7. If your NAT Gateway crashes, do running ECS tasks keep serving traffic? Why or why not? What about new tasks trying to start?

Yes.

Running ECS tasks continue serving requests because communication between the Application Load Balancer and ECS tasks occurs entirely within the VPC and does not depend on the NAT Gateway.

However, launching new tasks becomes problematic because task initialization requires outbound internet connectivity to:

- Pull container images from Amazon ECR
- Retrieve secrets from AWS Secrets Manager
- Publish logs to Amazon CloudWatch

Without a functioning NAT Gateway (or equivalent VPC endpoints), newly launched tasks will fail to initialize, eventually reducing service capacity if existing tasks terminate.

---

# Challenges Faced During Deployment

## Issue 1 – HTTP 500 Internal Server Error

The infrastructure deployment completed successfully:

- ECS Service: **2 Running Tasks**
- Target Group: **2 Healthy Targets**
- HTTPS configured successfully through ACM and ALB

However, user registration returned an HTTP 500 response.

CloudWatch logs identified the root cause:

```
psycopg2.errors.UndefinedTable: relation "user" does not exist
```

This confirmed that:

- Application connectivity to Amazon RDS was successful.
- Network, Security Groups, and database connectivity were functioning correctly.
- The required database schema had not been created before application startup.

The issue was isolated using CloudWatch Logs, demonstrating the importance of validating application initialization after infrastructure deployment.

---

## Issue 2 – ECS Exec Configuration

During troubleshooting, ECS Exec could not be enabled.

The console returned:

```
A valid taskRoleArn is required for enabling ECS Exec.
```

The root cause was that the ECS Task Definition did not have a **Task Role** associated with it.

Unlike the **Task Execution Role**, which ECS uses to pull images and retrieve secrets, the **Task Role** is assumed by the running container itself.

Creating an appropriate Task Role and associating it with the Task Definition is required before ECS Exec can establish secure Systems Manager sessions into running containers.
