# answers.md

## 1. Why does 10.0.0.0/16 have 65,534 usable hosts and not 65,536?

**Answer:**
For /16 network has 65,536 total IP addresses but the reaming two addresses are reserved for the network ID and broadcast address,so the rest of all 65,534 usable host addresses.

---

## 2. What is the relationship cardinality between an IGW and a VPC? Why?

**Answer:**
An IGW has a one-to-one relationship with a VPC. A VPC can have only one attached IGW, and an IGW can be attached to only one VPC .The reason AWS designed it this way is that the Internet Gateway acts as the single entry and exit point between the VPC and the public internet. Having multiple IGWs attached to the same VPC would create routing confusion because AWS would not know which gateway should handle outbound internet traffic.

---

## 3. The route table already had one route before you added 0.0.0.0/0. What was it, what was its target, and why does it exist by default?

**Answer:**
The default route (10.0.0.0/16 ) target was local, it helps for the VPC's internal routing. AWS automatically creates this route so that all resources within the VPC can communicate with each other without requiring additional routing configuration. Without this route, instances in different subnets of the same VPC would not be able to exchange traffic.

---

## 4. (write 2–3 sentences): Open the role and look at the Trust relationships tab. What does the JSON say? Explain in your own words what "trust policy" means and how it differs from "permissions policy."

**Answer:**
When I opened the Trust Relationship tab, the JSON showed that the principal ec2.amazonaws.com is allowed to perform sts:AssumeRole. This means EC2 instances are trusted to assume and use this IAM role. A trust policy defines who can assume the role, whereas a permissions policy defines what actions the role can perform after it has been assumed. In this case, the trust policy allows EC2 to use the role, and the attached permissions policy (AmazonS3ReadOnlyAccess) determines what S3 operations the EC2 instance can perform.

---

## 5. Difference between AWS Managed Policy, Customer Managed Policy, and Inline Policy?

**Answer:**

**AWS Managed Policy:** Created and maintained by AWS . Use when you need standard AWS-defined permissions quickly.
For Example: AmazonS3ReadOnlyAccess

**Customer Managed Policy:** Created and managed by the customer. Use when implementing least-privilege access in production environments.
Example: Allow access only to one specific S3 bucket

**Inline Policy:** Embedded directly into a single user, group, or role. Use when permissions are unique to one identity and should not be reused elsewhere.

---

## 6. Difference between Gateway Endpoint and Interface Endpoint?

**Answer:**
A Gateway Endpoint is a route-table-based endpoint that provides private connectivity to AWS services without using the internet, NAT Gateway, or VPN. It is available only for Amazon S3 and DynamoDB and does not incur hourly charges. it supports:

* Amazon S3
* Amazon DynamoDB

An Interface Endpoint creates Elastic Network Interfaces (ENIs) inside subnets and provides private connectivity to AWS services through AWS PrivateLink. It supports most AWS services but incurs hourly and data processing charges. it supports:

* SSM
* Secrets Manager
* CloudWatch
* ECR
* SNS
* SQS

---

## 7. What three things must be true for SSM Session Manager to work on a private EC2?

**Answer:**

### 1. IAM Role

The EC2 instance must have an IAM role containing:

AmazonSSMManagedInstanceCore

This grants permissions to communicate with AWS Systems Manager.

### 2. Network Connectivity

The instance must be able to reach the SSM service, either through:

* Interface Endpoints
* NAT Gateway

Without network connectivity, the agent cannot register with SSM.

### 3. SSM Agent

The SSM Agent must be installed and running on the instance. The agent establishes and maintains communication with the Systems Manager service.
