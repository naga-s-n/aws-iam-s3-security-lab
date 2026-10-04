B# EC2 Nginx Launch Lab

## Objective

Launch an Amazon EC2 instance, install and run Nginx using User Data, configure network access with a Security Group, and verify that the web server is reachable from the internet.

---

## Architecture

```text
Internet
   |
   | HTTP : 80
   v
Security Group
   |
   v
EC2 Instance
   |
   v
Ubuntu + Nginx
```

---

## EC2 Configuration

The instance was launched with the following configuration:

- AMI: Ubuntu Server 26.04 LTS
- Instance type: t3.micro
- VPC: Default VPC
- Subnet: Default VPC subnet
- Auto-assign public IPv4 address: Enabled
- Root volume: 30 GiB gp3
- Root volume IOPS: 3000
- Root volume encryption: Disabled
- Key pair authentication: Enabled

The EC2 instance was named:

`ec2-nginx-learning`

---

## Security Group

A Security Group named:

`ec2-nginx-learning-sg`

was configured with the following inbound rules:

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| TCP | 22 | My public IP /32 | SSH administration |
| TCP | 80 | 0.0.0.0/0 | Public HTTP access |

SSH access was restricted to a single public IP using a `/32` CIDR rather than being exposed to the entire internet.

HTTP port 80 was intentionally open to the internet so the Nginx web page could be accessed through the EC2 instance's public IPv4 address.

---

## User Data

The following User Data script was supplied during instance launch:

```bash
#!/bin/bash
apt update
apt install nginx -y
systemctl enable nginx
systemctl start nginx
```

The script performs four actions:

1. Refreshes the package index.
2. Installs Nginx.
3. Enables Nginx to start automatically on future boots.
4. Starts the Nginx service.

This allowed the web server to be automatically configured during the initial instance launch.

---

## Verification

After the EC2 instance passed its status checks, the public IPv4 address was opened in a browser using:

```text
http://<PUBLIC-IP>
```

The default:

`Welcome to nginx!`

page was successfully displayed.

This confirmed that:

- The EC2 instance was running.
- Nginx had been installed successfully.
- Nginx was listening for HTTP traffic.
- The Security Group allowed inbound TCP port 80.
- The instance was reachable through its public IPv4 address.

---

## SSH Access

The private key file permissions were restricted before connecting:

```bash
chmod 400 ec2-nginx-learning-key.pem
```

The instance was then accessed using SSH:

```bash
ssh -i ec2-nginx-learning-key.pem ubuntu@<PUBLIC-IP>
```

The Nginx service status was verified with:

```bash
sudo systemctl status nginx
```

Nginx was shown as:

```text
active (running)
```

The listening network port was also checked using:

```bash
sudo ss -tulpn | grep nginx
```

Nginx was listening on TCP port 80.

---

## Application vs Network Troubleshooting Test

To understand the difference between application availability and network access, Nginx was manually stopped.

When Nginx was stopped, the website became unreachable even though the Security Group still allowed inbound HTTP traffic on port 80.

After Nginx was started again, the website became reachable.

This demonstrated an important troubleshooting principle:

> A Security Group can allow traffic to a port, but an application must also be running and listening on that port for the request to succeed.

---

## Key Learnings

- EC2 provides virtual compute instances in AWS.
- An AMI provides the starting machine image for an EC2 instance.
- Instance types define compute characteristics such as CPU and memory.
- Security Groups control allowed network traffic to and from an instance.
- SSH port 22 should not be unnecessarily exposed to the entire internet.
- User Data can automate initial instance configuration.
- `systemctl start` starts a service immediately.
- `systemctl enable` configures a service to start automatically on future boots.
- A Security Group allowing a port does not guarantee that an application is listening on that port.
- Public IPv4 addresses allow internet communication when routing and security rules also permit it.

---

## Security Notes

- The private `.pem` key is not stored in this repository.
- No AWS access keys or secret credentials are included.
- SSH access was restricted to the required source IP.
- Public HTTP access was enabled only for the web-server learning exercise.

