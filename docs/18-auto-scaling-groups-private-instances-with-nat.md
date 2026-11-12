# Auto Scaling Group with Private EC2 Instances and Two NAT Gateways

## 1. Project Overview

This project demonstrates how to deploy an AWS Auto Scaling Group (ASG) using private EC2 instances across two Availability Zones.

Unlike the previous project, where EC2 instances were deployed in public subnets without NAT Gateways, this architecture uses:

- Two public subnets.
- Two private subnets.
- Two NAT Gateways (one per Availability Zone).
- An internet-facing Application Load Balancer.
- A private Auto Scaling Group.
- AWS Systems Manager Session Manager for instance access.
- CPU-based target tracking scaling.

The objective was to understand how private EC2 instances can access the internet for outbound communication without receiving public IPv4 addresses, while remaining accessible through an Application Load Balancer.

## 2. Architecture

### Region

Mumbai (`ap-south-1`)

### VPC

- Name: `asg-private-lab-vpc`
- CIDR: `10.7.0.0/16`

### Subnets

| Subnet | CIDR | Availability Zone | Type |
|---|---|---|---|
| Public Subnet A | `10.7.1.0/24` | ap-south-1a | Public |
| Public Subnet B | `10.7.2.0/24` | ap-south-1b | Public |
| Private Subnet A | `10.7.11.0/24` | ap-south-1a | Private |
| Private Subnet B | `10.7.12.0/24` | ap-south-1b | Private |

### Network Components

- Internet Gateway attached to the VPC.
- Two NAT Gateways, each deployed in a separate public subnet.
- Two private route tables, each associated with its corresponding private subnet.
- Public route table with a default route to the Internet Gateway.

### Routing

| Route Table | Associated Subnet | Default Route |
|---|---|---|
| Public Route Table | Public A and B | `0.0.0.0/0` → Internet Gateway |
| Private Route Table A | Private A | `0.0.0.0/0` → NAT Gateway A |
| Private Route Table B | Private B | `0.0.0.0/0` → NAT Gateway B |

Each private subnet uses the NAT Gateway in the same Availability Zone.

This design avoids depending on a NAT Gateway in another Availability Zone and provides better Availability Zone-level resilience.

## 3. Security Groups

### Application Load Balancer Security Group

Name: `asg-private-alb-sg`

Inbound:
- HTTP (TCP 80) from `0.0.0.0/0`

Outbound:
- All traffic allowed.

### EC2 Security Group

Name: `asg-private-ec2-sg`

Inbound:
- HTTP (TCP 80) with the source restricted to `asg-private-alb-sg`.
- No public SSH access.

Outbound:
- All traffic allowed.

This ensures that application instances accept HTTP traffic only from the Application Load Balancer.

## 4. Target Group

Name: `asg-private-tg`

Configuration:
- Target type: Instances
- Protocol: HTTP
- Port: 80
- Health check path: `/`
- Success codes: `200`

The target group automatically registers and deregisters instances launched and terminated by the Auto Scaling Group.

## 5. Application Load Balancer

Name: `asg-private-alb`

Configuration:
- Scheme: Internet-facing
- IP address type: IPv4
- VPC: `asg-private-lab-vpc`
- Subnets: Both public subnets
- Security group: `asg-private-alb-sg`
- Listener: HTTP (80)
- Forwarding target: `asg-private-tg`

The ALB provides public access to the application while the actual EC2 instances remain in private subnets.

## 6. Launch Template

Name: `asg-private-template`

Configuration:
- AMI: Amazon Linux 2023
- Instance type: `t3.micro`
- Key pair: None
- Security group: `asg-private-ec2-sg`
- IAM instance profile: `asg-private-ssm-role`
- Public IPv4 address: Not assigned through the private subnet configuration
- IMDSv2: Used for metadata access

### User Data

The following script installs Apache and creates a webpage displaying the instance's Availability Zone and instance ID.

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
<h1>Hello from Private Auto Scaling Group!</h1>
<p>Availability Zone: $AZ</p>
<p>Instance ID: $INSTANCE_ID</p>
</body>
</html>
EOF
```

The instances use their respective NAT Gateways to reach external package repositories during initialization.

## 7. IAM Role and Systems Manager Session Manager

Name: `asg-private-ssm-role`

Trusted entity:
- EC2

Attached AWS managed policy:
- `AmazonSSMManagedInstanceCore`

The IAM role was attached to the running instances and configured in the default version of the Launch Template for future instances.

Systems Manager Session Manager was used to connect to private EC2 instances without requiring:

- Public IPv4 addresses.
- SSH access.
- An inbound port 22 rule.
- A bastion host.

Both instances eventually appeared under Systems Manager Managed Nodes with ping status `Online`.

Session Manager connectivity was verified successfully.

## 8. Auto Scaling Group

Name: `asg-private-group`

Configuration:

| Setting | Value |
|---|---|
| Launch Template | `asg-private-template` |
| VPC | `asg-private-lab-vpc` |
| Subnets | Both private subnets |
| Minimum capacity | 2 |
| Desired capacity | 2 |
| Maximum capacity | 4 |
| Health checks | EC2 and ELB |
| Health check grace period | 180 seconds |
| Target Group | `asg-private-tg` |

The Auto Scaling Group launches instances exclusively in the private subnets.

### Scaling Policy

Policy name: `asg-private-cpu-target-tracking`

Configuration:
- Policy type: Target tracking scaling.
- Metric: Average CPU utilization.
- Target value: 50%.
- Instance warmup: 180 seconds.

The policy automatically adjusts the number of instances based on average CPU utilization.

## 9. Testing and Verification

### Test 1: Initial Instance Launch

- ASG launched two EC2 instances.
- Both instances were placed in private subnets.
- Neither instance received a public IPv4 address.
- Both instances registered with the target group.
- Both target health checks became `Healthy`.
- Application was accessible through the ALB DNS name.

### Test 2: Private Instance Connectivity

- Connected to an instance using Systems Manager Session Manager.
- Verified the SSM Agent was active and running.
- Successfully generated an IMDSv2 token.
- Retrieved instance metadata.
- Confirmed the Availability Zone was `ap-south-1b` for the tested instance.

### Test 3: Scale-Out (2 → 4)

Generated CPU load using the Linux `yes` command:

```bash
yes > /dev/null &
```

Observed CPU utilization increase and the target tracking policy trigger scale-out.

Results:
- ASG increased desired capacity from 2 to 4.
- Two additional EC2 instances were launched.
- All four instances registered with the target group.
- All four targets reported `Healthy`.

### Test 4: Scale-In (4 → 2)

Stopped the CPU stress process:

```bash
pkill -x yes
```

Verified that the process had stopped:

```bash
pgrep -a yes
```

Results:
- CPU utilization decreased.
- ASG scaled in from 4 to 2 instances.
- Two instances remained.
- Both registered targets reported `Healthy`.

### Test 5: Automatic Instance Replacement

Manually terminated one EC2 instance belonging to the Auto Scaling Group.

Results:
- ASG detected that the number of running instances had dropped below desired capacity.
- A replacement EC2 instance was launched.
- The replacement registered with the target group.
- The replacement target health status became `Healthy`.
- ASG returned to its desired capacity of two instances.

## 10. Final Results

All planned tests were successfully completed.

| Test | Result |
|---|---|
| Private EC2 deployment | Successful |
| No public IPv4 on application instances | Verified |
| Two NAT Gateways across two AZs | Configured |
| Outbound connectivity through NAT | Verified through package installation |
| SSM Session Manager connectivity | Successful |
| CPU-based scale-out (2 → 4) | Successful |
| Target group health checks | Healthy |
| CPU-based scale-in (4 → 2) | Successful |
| Automatic instance replacement | Successful |

## 11. Key Learnings

1. Private EC2 instances can access the internet for outbound communication through NAT Gateways without having public IPv4 addresses.
2. Deploying one NAT Gateway per Availability Zone avoids cross-AZ NAT dependencies.
3. Private EC2 instances can be managed securely using Systems Manager Session Manager without SSH access.
4. Auto Scaling Groups automatically launch replacement instances when instances are terminated unexpectedly.
5. Target tracking policies can automatically scale an application based on CPU utilization.
6. Application Load Balancers distribute traffic across healthy instances registered in their target groups.
7. Health checks help the ALB route traffic only to healthy application instances.

## 12. Cleanup

All resources created for this lab should be deleted after documentation and verification.

The cleanup sequence will be documented and performed separately to avoid leaving chargeable resources running.

Particular attention should be given to both NAT Gateways and their Elastic IP addresses, since they incur ongoing charges.

---

**Project status: Completed and verified.**

