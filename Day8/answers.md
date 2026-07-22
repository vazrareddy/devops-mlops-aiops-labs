# Answers

## 1. Why did we create the NAT Gateway before the ECS service? What error do you see if you flip the order?

The ECS tasks run inside private subnets and require outbound internet access to:
- Pull Docker images from Amazon ECR
- Retrieve secrets from AWS Secrets Manager
- Send logs to Amazon CloudWatch

The NAT Gateway provides this internet access.

If the ECS service is created before the NAT Gateway, the tasks cannot access these AWS services and fail to start. Common errors include:
- `CannotPullContainerError`
- Image pull timeout
- Secrets Manager access failure
- CloudWatch log delivery failure

---

## 2. Why does the target group use IP target type instead of Instance?

The application is deployed on AWS ECS Fargate, which does not use EC2 instances.

Each Fargate task receives its own Elastic Network Interface (ENI) and private IP address. Therefore, the Application Load Balancer routes traffic directly to the task IP address, requiring the target type to be **IP** instead of **Instance**.

---

## 3. Why is the health check path `/login` and not `/`? What HTTP status code does `/` return for this app?

The application redirects users from `/` to `/login`.

- `/` returns **HTTP 302 (Redirect)**
- `/login` returns **HTTP 200 (OK)**

Since the Application Load Balancer expects a successful response for health checks, `/login` is used as the health check endpoint.

---

## 4. Why are ALB Security Group, ECS Security Group, and RDS Security Group three separate groups instead of one big Security Group?

Separate Security Groups follow the **Principle of Least Privilege**.

- **ALB Security Group**
  - Allows HTTP/HTTPS traffic from the internet.

- **ECS Security Group**
  - Allows application traffic only from the ALB Security Group.

- **RDS Security Group**
  - Allows PostgreSQL traffic only from the ECS Security Group.

Using separate Security Groups improves security, reduces unnecessary access, and makes rule management easier.

---

## 5. The task definition uses `valueFrom` instead of `value` for `DB_LINK`. What's the security benefit? What permission does the execution role need?

The database connection string is stored securely in **AWS Secrets Manager** instead of hardcoding it inside the task definition.

Using `valueFrom` prevents exposing database credentials in ECS configurations or source code.

The ECS task requires permission to retrieve the secret using:

- `secretsmanager:GetSecretValue`

This ensures sensitive information remains secure.

---

## 6. Why does scale-in cooldown (300s) need to be longer than scale-out cooldown (60s)?

Scale-out should happen quickly when application traffic increases so that additional tasks can handle the load.

Scale-in should happen more slowly to ensure traffic has actually decreased before terminating running tasks.

This prevents unnecessary task creation and termination (thrashing) and provides a more stable application.

---

## 7. If your NAT Gateway crashes, do running ECS tasks keep serving traffic? Why or why not? What about new tasks trying to start?

Yes.

Running ECS tasks continue serving traffic because communication between the Application Load Balancer and ECS tasks happens entirely within the VPC and does not require the NAT Gateway.

However, new ECS tasks may fail to start because they need outbound internet access through the NAT Gateway to:
- Pull container images from Amazon ECR
- Retrieve secrets from AWS Secrets Manager
- Send logs to Amazon CloudWatch

Without the NAT Gateway, these startup operations cannot be completed.

---

# Challenges Faced During Deployment

### Issue 1: HTTP 500 Internal Server Error during User Registration

After deployment:
- ECS Service showed **2 Running Tasks**
- Target Group showed **2 Healthy Targets**
- Application was accessible over HTTPS

However, submitting the registration form returned:

```
500 Internal Server Error
```

CloudWatch logs showed:

```
psycopg2.errors.UndefinedTable: relation "user" does not exist
```

This confirmed that:
- ECS deployment was successful.
- The application could connect to Amazon RDS.
- The required database table had not been created.

---

### Issue 2: ECS Exec could not be enabled

While enabling ECS Exec, the following error was displayed:

```
A valid taskRoleArn is required for enabling ECS Exec.
```

The Task Definition did not have a Task Role configured.

The solution is to create an ECS Task Role, attach the required IAM permissions, associate it with the Task Definition, and then enable ECS Exec.
