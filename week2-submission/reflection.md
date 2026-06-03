# reflection.md

## 1. In Task 2.1 you opened SSH from 0.0.0.0/0. In a real company, what would you restrict it to instead, and how?

In a real company, I would not allow SSH access from 0.0.0.0/0 because it exposes the server to the entire internet. Instead, I would restrict access to a specific corporate VPN CIDR range or my public IP address using a /32 rule in the security group. This follows the principle of least privilege and reduces the attack surface. In many organizations, direct SSH access is avoided completely by using AWS Systems Manager Session Manager.

---

## 2. You copied a private key onto the bastion in Task 3.2. Name two reasons this is a bad practice and one alternative.

Copying a private key to a bastion host is risky because anyone who gains access to the bastion can potentially steal the key and access other systems. It also makes key management, rotation, and auditing more difficult because the same key may exist in multiple locations. A better alternative is AWS Systems Manager Session Manager, which allows secure access to private EC2 instances without storing or copying SSH keys.

---

## 3. When would you choose a Gateway Endpoint vs an Interface Endpoint vs just using the NAT Gateway?

I would use a Gateway Endpoint when private EC2 instances need access to Amazon S3 or DynamoDB because it is free and keeps traffic on the AWS network. I would use an Interface Endpoint when private access is required for services such as Systems Manager, Secrets Manager, ECR, or CloudWatch because these services do not support Gateway Endpoints. I would use a NAT Gateway when instances need general internet access, such as downloading operating system packages, accessing third-party APIs, or reaching public repositories.

---

## 4. VPC Peering vs Transit Gateway — when does Transit Gateway start to make sense?

VPC Peering works well when only a small number of VPCs need to communicate. As the number of VPCs increases, managing individual peering connections becomes complex because every VPC needs connections to multiple other VPCs. Transit Gateway provides a hub-and-spoke architecture that simplifies routing and connectivity management. It becomes the preferred option in large environments with multiple VPCs, AWS accounts, and regions.
