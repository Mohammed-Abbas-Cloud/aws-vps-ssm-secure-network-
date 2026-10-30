
# AWS VPC from Scratch: Public/Private + IGW + NAT + SSM Overview

This project demonstrates the design and implementation of a highly secure, custom AWS Virtual Private Cloud (VPC) environment from scratch. It focuses on isolating compute resources in a private subnet with outbound internet access via a NAT Gateway, while completely eliminating the need for SSH keys, bastion hosts, and open inbound ports by utilizing AWS Systems Manager (SSM) Session Manager for secure access.

---

## Architecture

```text
Internet
   │
 [IGW]
   │
┌──┴────── VPC 10.0.0.0/16 ────────┐
│ Public Subnet 10.0.1.0/24         │
│   └─ NAT Gateway (+ Elastic IP)   │
│                                   │
│ Private Subnet 10.0.2.0/24        │
│   └─ EC2 (No Public IP)           │
└───────────────────────────────────┘

```

> **Access server via SSM (No SSH)**
> The server resides in the Private Subnet, completely isolated from direct inbound internet traffic since there is no direct route from the internet to it. It accesses the internet (for package updates and patches) outwardly through the NAT Gateway. Administrative access is handled entirely via AWS SSM, eliminating the need for port 22, SSH key pairs, or Bastion hosts.

---

## Services Used

* **VPC (Virtual Private Cloud):** An isolated private network within AWS where you control the IP address range and network configuration.
* **Subnet:** A segment of the VPC's IP range tied to a single Availability Zone (AZ). A subnet is "Public" if its route table directs internet traffic to an IGW, and "Private" if it does not.
* **Internet Gateway (IGW):** The bi-directional gateway that connects the VPC to the internet.
* **NAT Gateway:** Enables instances in the Private Subnet to connect to the internet (outbound only) without exposing them to inbound internet connections. It requires a static Elastic IP.
* **Route Table:** A set of rules determining where network traffic is directed. This logically separates Public and Private subnets.
* **Security Group:** A stateful, instance-level firewall.
* **IAM Role:** Grants permissions to the EC2 instance. In this project, it allows the SSM Agent to communicate with the AWS Systems Manager.
* **SSM Session Manager:** Provides a secure, browser-based shell to the instance via HTTPS. Access is controlled by IAM, and sessions can be audited via CloudTrail.
* **VPC Endpoints (Optional):** Provides a private route from the VPC directly to AWS services (like SSM) without traversing the public internet.

---

## Implementation Steps

> **Note:** All resources were deployed in a single AWS Region.

1. **Create the VPC**
Created a custom VPC to establish the foundational network boundaries.
* **Name:** `lab-vpc`
* **IPv4 CIDR:** `10.0.0.0/16` (Provides 65,536 IPs for flexible subnetting)
* **Settings:** Enabled DNS hostnames (crucial for VPC Endpoints later).


2. **Configure Subnets**
Segmented the network into an exposed tier (for NAT/Load Balancers) and a protected tier (for servers/data) within the same AZ.
* **Public Subnet (`public-subnet-a`):** `10.0.1.0/24`
* **Private Subnet (`private-subnet-a`):** `10.0.2.0/24`


3. **Deploy the Internet Gateway (IGW)**
Created an IGW named `lab-igw` and attached it to `lab-vpc` to provide an internet pathway for the public subnet.
4. **Set Up Route Tables**
* `public-rt`: Added a route (`0.0.0.0/0`) targeting the IGW, then associated it with `public-subnet-a`.
* `private-rt`: Created and associated with `private-subnet-a` (left without internet routes initially).


5. **Provision the NAT Gateway**
Deployed a Zonal NAT Gateway in the public subnet to allow outbound traffic for private instances.
* **Name:** `lab-nat`
* **Subnet:** `public-subnet-a`
* **Elastic IP:** Allocated a new EIP.
* **Routing:** Once the NAT was Available, updated `private-rt` with a route (`0.0.0.0/0`) targeting the NAT Gateway.


6. **Create the IAM Role for SSM**
Created an IAM Role (`lab-ssm-role`) with the `AmazonSSMManagedInstanceCore` policy attached, allowing the EC2 instance's SSM Agent to register and receive commands.
7. **Configure Security Groups**
Created `private-ec2-sg` in `lab-vpc`.
* **Inbound Rules:** None (Empty).
* **Outbound Rules:** All traffic allowed.


8. **Launch the EC2 Instance**
Deployed the workload into the secure private subnet.
* **Name:** `private-ec2`
* **AMI:** Amazon Linux 2023 (SSM Agent pre-installed)
* **Instance Type:** `t3.micro`
* **Key Pair:** Proceed without a key pair
* **Network:** `lab-vpc`, `private-subnet-a`, Auto-assign public IP disabled, `private-ec2-sg`.
* **IAM Profile:** Attached `lab-ssm-role`.



---

## Security Decisions

* **Why a Private Subnet?**
Keeps compute resources completely isolated from direct internet scans, attacks, or unauthorized access.
* **Why SSM instead of SSH?**
Removes the need to manage, share, or rotate SSH keys. Access is entirely governed by centralized IAM policies, and traffic flows over secure HTTPS rather than exposing port 22.
* **Why an empty Inbound Security Group?**
SSM operates purely via outbound connections (the agent reaches out to the SSM service). Therefore, zero inbound ports need to be opened, providing the strongest possible network-level security posture.

---

## Testing & Results

### Connection Test

After 2-5 minutes, successfully connected to the instance via **EC2 Console → Connect → Session Manager**. Verified the node was **Online** in Systems Manager Fleet Manager.

### Outbound Traffic & NAT Verification

Ran the following commands inside the SSM shell:

```bash
curl -s [https://checkip.amazonaws.com](https://checkip.amazonaws.com)
sudo dnf update -y

```

* **Result:** The IP returned matched the Elastic IP of the NAT Gateway, proving the private instance has no public IP of its own and successfully routes outwardly through the NAT. The `dnf update` executed successfully.

### Route Deletion Test

Deleted the `0.0.0.0/0` route targeting the NAT Gateway in `private-rt` and re-ran the update.

* **Result:** The update failed and timed out, successfully demonstrating the NAT Gateway's critical role in outbound connectivity.

---

## Cost & Cleanup

> ⚠️ **Important Note:** NAT Gateways incur an hourly charge as well as data processing fees. To prevent ongoing charges, the NAT Gateway and its associated Elastic IP were deleted immediately after lab completion.

---

## Lessons Learned

This project provided hands-on experience with the absolute core principles of AWS networking and zero-trust access. It highlighted that a subnet's "Public" or "Private" status is dictated entirely by its Route Table, not its configuration at creation. It also demonstrated the power of the SSM agent, proving that traditional Bastion hosts and SSH keys are largely obsolete for modern, secure AWS administration.

---

## Future Improvements

* **VPC Endpoints (PrivateLink):** Create interface endpoints for `ssm`, `ssmmessages`, and `ec2messages` to route SSM traffic entirely within the AWS backbone, bypassing the NAT Gateway and internet entirely for administrative access.
* **Multi-AZ High Availability:** Expand the architecture to span at least two Availability Zones with duplicate public/private subnets and load balancing.
* **Regional NAT Gateway:** Explore the Regional NAT Gateway feature, which automatically scales across AZs without requiring a dedicated public subnet to host it, optimizing cost and HA architecture.
* **Infrastructure as Code (IaC):** Replicate this exact setup using Terraform or AWS CloudFormation for automated, repeatable deployments.
* **Auditing:** Configure SSM to log all terminal session inputs and outputs securely into an encrypted S3 Bucket or CloudWatch Logs for compliance.
