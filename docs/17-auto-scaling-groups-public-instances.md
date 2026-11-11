# Auto Scaling Groups with Public EC2 Instances (Without NAT Gateway)

## 1. Project Overview

This project demonstrates how to configure an AWS Auto Scaling Group (ASG) integrated with an Application Load Balancer (ALB), using public EC2 instances across two Availability Zones.

The objective was to understand how an Auto Scaling Group automatically:
- Launches EC2 instances.
- Maintains the desired number of instances.
- Scales out when CPU utilization increases.
- Scales in when CPU utilization decreases.
- Replaces terminated EC2 instances.
- Registers newly launched instances with an ALB target group.

This project uses public subnets and an Internet Gateway. No NAT Gateway was created.

**AWS Region:** Asia Pacific (Mumbai) — `ap-south-1`

## 2. Architecture

### Architecture Diagram

```text
                         Internet
                            |
                            |
                    Internet Gateway
                            |
                            |
                 Application Load Balancer
                      (Internet-facing)
                            |
                     Listener HTTP:80
                            |
                            |
                     Target Group
                       (HTTP:80)
                            |
                 Auto Scaling Group
                  Min: 2 | Max: 4
                            |
                 ---------------------
                 |                   |
                 |                   |
          Public Subnet A      Public Subnet B
          ap-south-1a           ap-south-1b
          10.6.1.0/24           10.6.2.0/24
                 |                   |
              EC2 #1              EC2 #2
              Apache              Apache
                 |                   |
                 ---------------------
                            |
                    Shared VPC Network

VPC CIDR: 10.6.0.0/16
NAT Gateway: Not used
```

The Auto Scaling Group manages the EC2 instances, while the ALB distributes incoming application traffic across healthy registered targets.

## 3. VPC and Networking Configuration

| Resource | Configuration |
|---|---|
| VPC Name | `asg-lab-vpc` |
| IPv4 CIDR | `10.6.0.0/16` |
| Public Subnet A | `asg-public-subnet-a` |
| Subnet A CIDR | `10.6.1.0/24` |
| Availability Zone A | `ap-south-1a` |
| Public Subnet B | `asg-public-subnet-b` |
| Subnet B CIDR | `10.6.2.0/24` |
| Availability Zone B | `ap-south-1b` |
| Internet Gateway | Created and attached |
| Route Table | Custom public route table |
| Default Route | `0.0.0.0/0` → Internet Gateway |
| Local Route | Automatically created VPC local route |
| Auto-assign Public IPv4 | Enabled on both subnets |
| NAT Gateway | Not created |

Both public subnets were associated with the custom route table.

## 4. Security Groups

### ALB Security Group — `asg-alb-sg`

Inbound rules:

| Type | Protocol | Port | Source |
|---|---|---|---|
| HTTP | TCP | 80 | `0.0.0.0/0` |

Outbound rules: Default allow-all outbound rule.

### EC2 Security Group — `asg-ec2-sg`

Inbound rules:

| Type | Protocol | Port | Source |
|---|---|---|---|
| HTTP | TCP | 80 | `asg-alb-sg` |

Outbound rules: Default allow-all outbound rule.

**Security principle:**

Although the EC2 instances are launched in public subnets and receive public IPv4 addresses, inbound HTTP access is permitted only from the ALB security group.

Direct public HTTP access to the EC2 instances is therefore restricted.

A temporary SSH inbound rule using the AWS-managed EC2 Instance Connect prefix list for Mumbai was added during CPU scaling tests. This rule was removed after testing.

## 5. Launch Template

| Setting | Configuration |
|---|---|
| Launch Template Name | `asg-lab-template` |
| Initial Version | Version 1 |
| Default Version | Updated version containing `asg-ec2-sg` |
| AMI | Amazon Linux 2023 |
| Instance Type | `t3.micro` |
| Key Pair | No key pair |
| Security Group | `asg-ec2-sg` |
| Storage | Default |
| VPC/Subnet Selection | Not specified in Launch Template |

The Auto Scaling Group determines the subnets in which the instances are launched.

### User Data Script

The following User Data script installs Apache and generates a webpage displaying the instance ID and Availability Zone.

```bash
#!/bin/bash
dnf install -y httpd
systemctl enable httpd
systemctl start httpd

TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

AZ=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)

INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)

cat <<EOF > /var/www/html/index.html
<html>
<body>
<h1>Hello from Auto Scaling Group!</h1>
<p>Availability Zone: $AZ</p>
<p>Instance ID: $INSTANCE_ID</p>
</body>
</html>
EOF
```

IMDSv2 token-based requests are used to retrieve instance metadata.

The webpage helps identify which EC2 instance handled an incoming ALB request.

## 6. Target Group

| Setting | Configuration |
|---|---|
| Target Group Name | `asg-lab-tg` |
| Target Type | Instances |
| Protocol | HTTP |
| Port | 80 |
| VPC | `asg-lab-vpc` |
| Health Check Protocol | HTTP |
| Health Check Path | `/` |
| Success Codes | `200` |

The Auto Scaling Group automatically registers newly launched instances with this target group and deregisters instances that are terminated.

## 7. Application Load Balancer

| Setting | Configuration |
|---|---|
| ALB Name | `asg-lab-alb` |
| Scheme | Internet-facing |
| IP Address Type | IPv4 |
| VPC | `asg-lab-vpc` |
| Availability Zones | `ap-south-1a`, `ap-south-1b` |
| Subnets | Both public subnets |
| Security Group | `asg-alb-sg` |
| Listener | HTTP:80 |
| Default Action | Forward to `asg-lab-tg` |

The ALB DNS name was tested in a browser.

The webpage successfully displayed the Availability Zone and instance ID returned by the EC2 instance.

## 8. Auto Scaling Group Configuration

| Setting | Configuration |
|---|---|
| ASG Name | `asg-lab-group` |
| Launch Template | `asg-lab-template` |
| VPC | `asg-lab-vpc` |
| Subnets | Both public subnets |
| Minimum Capacity | 2 |
| Desired Capacity | 2 |
| Maximum Capacity | 4 |
| Target Group | `asg-lab-tg` |
| EC2 Health Checks | Enabled |
| ELB Health Checks | Enabled |
| Health Check Grace Period | 180 seconds |

### Target Tracking Scaling Policy

| Setting | Configuration |
|---|---|
| Policy Name | `asg-cpu-target-tracking` |
| Policy Type | Target Tracking |
| Metric | Average CPU Utilization |
| Target Value | 50% |
| Instance Warmup | 180 seconds, if configured |

The target tracking policy monitors average CPU utilization across the Auto Scaling Group.

It does not enforce a separate 50% CPU limit on every individual EC2 instance.

When average CPU utilization remains above the target, the policy can increase capacity. When it decreases below the target, the policy can reduce capacity, subject to scaling behavior and capacity limits.

## 9. Testing and Verification

### Test 1 — Initial Instance Launch

Created the Auto Scaling Group with a desired capacity of two.

**Result:**
- Two EC2 instances launched automatically.
- Both instances were running.
- Instances were distributed across the two Availability Zones.
- Both instances registered with the ALB target group.
- Both targets eventually became Healthy.

**Status:** PASSED

### Test 2 — ALB Traffic Distribution

Opened the ALB DNS name in a browser.

Verified that the webpage displayed:
- Auto Scaling Group greeting.
- Availability Zone.
- Instance ID.

Repeated browser requests to verify that the application was reachable through the ALB.

**Result:** Application loaded successfully and displayed instance metadata.

**Status:** PASSED

### Test 3 — Automatic Scale-Out

Configured a target tracking scaling policy with a 50% average CPU utilization target.

Temporarily added an SSH inbound rule using the AWS-managed EC2 Instance Connect prefix list.

Connected to both original EC2 instances using EC2 Instance Connect.

Executed the following commands on each instance:

```bash
yes > /dev/null &
yes > /dev/null &
```

These commands generated CPU-intensive background processes.

Observed CPU utilization:

| Instance | CPU Utilization |
|---|---:|
| Original EC2 Instance 1 | 94.63% |
| Original EC2 Instance 2 | 100% |

The Auto Scaling Group automatically launched the third and fourth instances.

The group reached its configured maximum capacity of four.

**Result:** Capacity increased from two to four instances automatically.

**Status:** PASSED

### Test 4 — Automatic Scale-In

Stopped the CPU-intensive background processes on both original instances.

Executed:

```bash
pkill -x yes
```

Monitored the Auto Scaling Group and observed that desired capacity and running instance count eventually returned to two.

**Result:** Capacity decreased from four to two after CPU demand reduced.

**Status:** PASSED

### Test 5 — Automatic Instance Replacement

Manually terminated one EC2 instance belonging to the Auto Scaling Group.

The ASG detected that the number of running instances had fallen below its desired capacity.

It automatically launched a replacement EC2 instance.

Verified that:
- The replacement instance was running.
- It registered with the target group.
- It eventually became Healthy.
- The application was accessible through the ALB DNS name.

**Result:** The Auto Scaling Group successfully replaced a terminated instance.

**Status:** PASSED

### Test 6 — Security Group Cleanup

Removed the temporary SSH inbound rule from `asg-ec2-sg`.

Verified that the intended HTTP-only inbound rule from `asg-alb-sg` remained.

**Status:** PASSED

## 10. Key Concepts Learned

1. A Launch Template defines the configuration used to launch EC2 instances.
2. An Auto Scaling Group manages EC2 instance capacity automatically.
3. Minimum, desired and maximum capacities control the permitted group size.
4. The ASG can launch instances across multiple Availability Zones.
5. The ASG can integrate with an ALB through a target group.
6. Newly launched instances are automatically registered with the target group.
7. Target tracking scaling policies use CloudWatch metrics to adjust capacity.
8. Average CPU utilization is calculated across the group rather than enforced individually per instance.
9. Scale-out increases capacity when demand rises.
10. Scale-in reduces capacity when demand decreases.
11. Maximum capacity prevents normal scaling activities from exceeding the configured upper limit.
12. The ASG can automatically replace terminated instances.
13. ELB health checks help the ASG identify instances that are not passing load balancer health checks.
14. Public EC2 instances can still restrict direct application access through security group references.
15. An ALB can distribute traffic across healthy EC2 instances managed by an ASG.

## 11. Project Outcome

Successfully implemented an Auto Scaling Group integrated with an internet-facing Application Load Balancer using public EC2 instances across two Availability Zones.

Demonstrated:
- Initial EC2 instance provisioning.
- ALB integration.
- CPU-based automatic scale-out.
- Automatic scale-in.
- Automatic EC2 instance replacement.
- Health check integration.
- Restricted EC2 inbound HTTP access.

No NAT Gateway was required for this architecture.

## 12. Resource Cleanup

All resources created specifically for this lab must be deleted after testing to avoid unnecessary AWS charges.

Cleanup sequence:

1. Delete the Auto Scaling Group (`asg-lab-group`), selecting the option to delete its EC2 instances.
2. Verify that the ASG-managed EC2 instances are terminated.
3. Delete the Application Load Balancer (`asg-lab-alb`).
4. Delete the target group (`asg-lab-tg`).
5. Delete the custom route table after disassociating both subnets.
6. Delete the security groups (`asg-ec2-sg` and `asg-alb-sg`).
7. Detach and delete the Internet Gateway.
8. Delete both public subnets.
9. Delete the VPC (`asg-lab-vpc`).
10. Verify that no leftover lab-related network interfaces, EBS volumes or snapshots remain.

No NAT Gateway or Elastic IP was created for this project.

**Cleanup status:** Pending at the time of documentation.

---

**Project:** Auto Scaling Groups with Public EC2 Instances (Without NAT Gateway)  
**Region:** `ap-south-1`  
**Status:** Implementation and testing completed; resource cleanup pending.
