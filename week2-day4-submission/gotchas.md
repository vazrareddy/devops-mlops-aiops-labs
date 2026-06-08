# gotchas.md

# Week 2 Day 4 — AWS HA Application Deployment Gotchas

This document captures every significant issue encountered during the lab, the root cause, how it was diagnosed, and the final fix.

---

## 🚨 Issue 1: Could Not SSH From Bastion To Private Instance

### Symptoms

SSH from the bastion host to the application instance failed:

```bash
ssh ec2-user@10.0.3.126
```

Error:

```text
Permission denied (publickey,gssapi-keyex,gssapi-with-mic)
```

### Root Cause

The private instance was launched using the `bootcamp-key` key pair, but the corresponding private key file (`bootcamp-key.pem`) was not present on the bastion host.

The network path was working correctly:

* Ping succeeded
* Port 22 was reachable
* SSH handshake completed

Authentication failed because no valid private key was available.

### Investigation

Verified network connectivity:

```bash
ping 10.0.3.126
```

Verified SSH port reachability:

```bash
ssh -v ec2-user@10.0.3.126
```

Debug logs showed successful connection establishment but failed public key authentication.

### Fix

Copied the private key to the bastion host and connected using:

```bash
ssh -i ~/bootcamp-key.pem ec2-user@10.0.3.126
```

Connection succeeded immediately.

### Lesson Learned

When testing Launch Templates, always ensure the bastion host has access to the same key pair used by private instances.

---

## 🚨 Issue 2: User Data Log File Missing

### Symptoms

Expected user-data output file did not exist:

```bash
tail -100 /var/log/user-data.log
```

Error:

```text
No such file or directory
```

### Root Cause

Amazon Linux 2023 does not automatically create:

```text
/var/log/user-data.log
```

unless the user-data script explicitly redirects output there.

### Investigation

Checked cloud-init logs instead:

```bash
sudo tail -100 /var/log/cloud-init-output.log
```

### Fix

Used:

```bash
sudo tail -100 /var/log/cloud-init-output.log
```

to verify:

* Package installation
* Git clone
* Python dependency installation
* Gunicorn startup

### Lesson Learned

For Amazon Linux 2023, cloud-init logs are the primary troubleshooting source.

---

## 🚨 Issue 3: Launch Template Validation

### Symptoms

Needed to verify whether Launch Template user-data was actually working.

### Investigation

Connected to the private test instance and reviewed:

```bash
sudo tail -100 /var/log/cloud-init-output.log
```

Observed:

```text
git clone
pip install
gunicorn startup
```

and:

```text
Listening at: http://0.0.0.0:8000
```

### Fix

Confirmed Launch Template version 1 was functioning correctly.

### Lesson Learned

Never assume user-data executed successfully. Always verify via cloud-init logs.

---

## 🚨 Issue 4: Auto Scaling Group Health Validation

### Symptoms

Needed proof that instances launched by the ASG were healthy.

### Investigation

Connected to the ASG-created instance.

Verified application process:

```bash
ps -ef | grep gunicorn
```

Verified application endpoint:

```bash
curl -I http://localhost:8000/login
```

Response:

```text
HTTP/1.1 200 OK
```

### Fix

Confirmed application startup and readiness before attaching to the target group.

### Lesson Learned

Always validate application health locally before debugging ALB or Target Group issues.

---

## 🚨 Issue 5: ACM Certificate Stuck In Pending Validation

### Symptoms

Certificate remained in:

```text
Pending validation
```

for several minutes.

### Root Cause

Initial certificate was requested for:

```text
www.vazrareddy.com
```

while the application was accessed through:

```text
app.vazrareddy.com
```

### Investigation

Reviewed ACM validation records and Route53 DNS configuration.

### Fix

Requested a new certificate specifically for:

```text
app.vazrareddy.com
```

Used:

```text
Create records in Route53
```

Certificate moved to:

```text
Issued
```

within minutes.

### Lesson Learned

Certificate domain names must exactly match the hostname used by the application.

---

## 🚨 Issue 6: ALB Security Group Could Not Be Deleted

### Symptoms

Deletion failed:

```text
Security group cannot be deleted
```

### Root Cause

AWS still had ENIs attached to the ALB even after deletion was initiated.

### Investigation

Reviewed:

* Load Balancers
* Network Interfaces
* Security Group dependencies

### Fix

Waited for ALB deletion to fully complete.

AWS automatically cleaned up the associated ENIs.

Security group deletion then succeeded.

### Lesson Learned

ALB deletion is asynchronous. Security groups often remain attached until ENI cleanup finishes.

---

## 🚨 Issue 7: Elastic IP Could Not Be Released

### Symptoms

Attempting release produced:

```text
Cannot be released with association IDs
```

### Root Cause

Elastic IP was still attached to the NAT Gateway.

### Investigation

Reviewed Elastic IP details:

```text
Associated NAT Gateway:
bootcamp-natgw
```

### Fix

Deleted NAT Gateway first.

After deletion completed:

```text
Release Elastic IP
```

worked successfully.

### Lesson Learned

Elastic IPs attached to NAT Gateways must be released only after the NAT Gateway is deleted.

---

## 🚨 Issue 8: RDS Subnet Group Could Not Be Deleted

### Symptoms

Deletion failed:

```text
Subnet group is currently in use
```

### Root Cause

RDS instance deletion had been initiated but had not yet completed.

### Investigation

Checked RDS status:

```text
Deleting
```

### Fix

Waited until the database disappeared completely from RDS.

Then deleted:

```text
bootcamp-rds-subnets
```

successfully.

### Lesson Learned

AWS resource deletion is often asynchronous. Verify the dependent resource is fully removed before deleting supporting infrastructure.

---

## 🚨 Issue 9: Load Test Did Not Immediately Trigger Scale-Out

### Symptoms

After running:

```bash
hey -z 5m -c 50 https://app.vazrareddy.com/login
```

the ASG did not scale immediately.

### Root Cause

Target tracking policies require:

* CloudWatch metric collection
* Alarm evaluation
* Instance warmup period

Scaling decisions are not instantaneous.

### Investigation

Monitored:

* CPU utilization
* ASG Activity tab
* Scaling policy status

### Fix

Continued load test long enough for metrics to accumulate.

Observed:

```text
Desired Capacity: 1 → 2 → 3
```

in Activity History.

### Lesson Learned

Auto Scaling reacts to sustained load, not short traffic spikes.

---

# Key Takeaways

✅ Verify user-data using cloud-init logs

✅ Validate application locally before debugging ALB

✅ Match ACM certificates to exact DNS names

✅ Delete AWS resources in dependency order

✅ Expect delays in AWS cleanup operations

✅ Auto Scaling requires time for CloudWatch metrics and warm-up periods

✅ Always validate assumptions using logs and metrics before making infrastructure changes

