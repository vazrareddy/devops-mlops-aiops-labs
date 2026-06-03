# gotchas.md

## 1. Incorrect Subnet CIDR Planning

Initially, I had to carefully verify that each subnet used a unique CIDR block. I double-checked all subnet ranges to ensure there were no overlaps and that each subnet fit within the VPC CIDR range.

**Fix:** Verified subnet calculations before creating resources.

---

## 2. Bastion SSH Access

I had to move the downloaded key pair to the ~/.ssh directory and set the correct permissions before SSH would work.

**Fix:** Used:

chmod 400 ~/.ssh/bootcamp-key.pem

and retried the connection.

---

## 3. Private EC2 Could Not Reach the Internet

Before creating the NAT Gateway, the private EC2 instance could not run yum update because there was no outbound internet path.

**Fix:** Created a NAT Gateway in the public subnet and added a default route (0.0.0.0/0) in the private route table.

---

## 4. Yum Update Process Was Stuck

While testing internet connectivity, a previous yum update process was still running and blocked new update commands.

**Fix:** Identified the process using:

ps -fp <PID>

and terminated the old process before rerunning the command.

---

## 5. Confusion Between Bastion and Private EC2

At one point I attempted to SSH into the private instance while I was already logged into it, which caused authentication errors and confusion.

**Fix:** Verified my current host using the shell prompt and private IP address before opening a new SSH session.

---

## 6. S3 Access Changed After Custom Policy

After replacing AmazonS3ReadOnlyAccess with a custom policy, the command aws s3 ls returned AccessDenied.

**Fix:** Confirmed that the custom policy intentionally allowed access only to the specific bucket and not to list all buckets in the account.

---

## 7. Interface Endpoint Creation Failed

While creating the SSM Interface Endpoints, AWS returned an error indicating that DNS hostnames and DNS resolution were not enabled for the VPC.

**Fix:** Enabled DNS Resolution and DNS Hostnames in the VPC settings and recreated the endpoint.

---

## 8. Security Group Rule Added in Wrong Section

While creating vpce-sg, I initially added the HTTPS rule in the outbound section instead of the inbound section.

**Fix:** Added an inbound HTTPS (443) rule from private-app-sg and kept the default outbound rule.

---

## 9. Understanding Route Tables

I initially focused only on internet routes and overlooked the default local route that AWS automatically creates.

**Fix:** Reviewed the route table and understood that the local route enables communication between resources inside the VPC.

---

## 10. Understanding Gateway Endpoint Behavior

After deleting the NAT Gateway, S3 access stopped working until the Gateway Endpoint was configured.

**Fix:** Created the S3 Gateway Endpoint and associated it with the private route table, which restored private connectivity to S3.
