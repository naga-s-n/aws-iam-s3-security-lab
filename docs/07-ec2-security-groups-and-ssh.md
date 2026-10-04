# EC2 Security Groups and SSH Lab

## Objective

Understand how Security Groups control network access to an EC2 instance and securely connect to the instance using SSH and an EC2 key pair.

This lab also demonstrates the difference between:

- Network-level access controlled by a Security Group.
- Application-level availability determined by whether a service is running and listening on a port.

---

## Architecture

```text
My Computer
    |
    | SSH : 22
    | Source: My Public IP /32
    |
    v
Security Group
    |
    v
EC2 Instance
    |
    +---- SSH service
    |
    +---- Nginx : 80
              ^
              |
Internet -----+
    HTTP : 80
```

The Security Group acts as a virtual firewall controlling traffic allowed to reach the EC2 instance.

---

## Security Group Configuration

The Security Group used in this lab was:

`ec2-nginx-learning-sg`

The inbound rules were:

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| TCP | 22 | My public IP /32 | SSH administration |
| TCP | 80 | 0.0.0.0/0 | Public HTTP access |

### SSH Rule

SSH uses:

```text
TCP port 22
```

Instead of allowing SSH from:

```text
0.0.0.0/0
```

the rule was restricted to the public IP used for administration:

```text
<My-Public-IP>/32
```

A `/32` IPv4 CIDR represents a single IPv4 address.

This reduces unnecessary exposure of the SSH port.

### HTTP Rule

HTTP uses:

```text
TCP port 80
```

For this learning web server, HTTP traffic was allowed from:

```text
0.0.0.0/0
```

This means IPv4 clients on the internet can attempt to reach the instance on TCP port 80, subject to the rest of the network path and the application being available.

---

## Security Groups Are Stateful

AWS Security Groups are stateful.

If inbound traffic is allowed and a connection is established, the response traffic for that connection is automatically allowed back through the Security Group.

This behavior does not require a separate inbound rule for the response traffic.

---

## EC2 Key Pair

The EC2 instance was launched using the key pair:

`ec2-nginx-learning-key`

The private key file was stored locally as:

```text
ec2-nginx-learning-key.pem
```

The private key must remain secret and must never be committed to GitHub.

The instance receives the corresponding public key during launch, allowing SSH authentication using the matching private key.

---

## Protecting the Private Key

Before using the private key for SSH, its local file permissions were restricted:

```bash
chmod 400 ec2-nginx-learning-key.pem
```

This allows the file owner to read the private key while preventing broader file access.

---

## Connecting with SSH

The EC2 instance was accessed using:

```bash
ssh -i ec2-nginx-learning-key.pem ubuntu@<PUBLIC-IP>
```

Where:

- `ssh` starts the SSH client.
- `-i` specifies the identity/private key file.
- `ubuntu` is the login user for the Ubuntu EC2 instance used in this lab.
- `<PUBLIC-IP>` represents the instance's current public IPv4 address.

Successful SSH access required both:

1. The correct private key.
2. Network access permitted by the Security Group.

Having only one of these is not sufficient.

---

## Understanding the `/32` Source Rule

The SSH Security Group rule was restricted to:

```text
My Public IP /32
```

This refers to the public source IPv4 address from which the SSH connection reaches AWS.

It does not uniquely identify a physical laptop.

For example, multiple devices using the same home network may appear to the internet through the same public IPv4 address because of NAT.

A device would still need the appropriate authentication credentials, such as the correct private key, to successfully authenticate using SSH.

---

## Verifying Nginx

After connecting to the EC2 instance, the Nginx service was checked with:

```bash
sudo systemctl status nginx
```

The service showed:

```text
active (running)
```

This confirmed that the Nginx service was running.

The listening network sockets were then inspected using:

```bash
sudo ss -tulpn | grep nginx
```

Nginx was observed listening on TCP port 80.

---

## Security Group vs Application

An important troubleshooting experiment was performed by stopping Nginx while leaving the Security Group HTTP rule unchanged.

When Nginx was stopped:

```text
Security Group allows TCP 80
            +
No application listening on TCP 80
            =
Website unavailable
```

After Nginx was started again:

```text
Security Group allows TCP 80
            +
Nginx listening on TCP 80
            =
Website reachable
```

This demonstrates that a Security Group does not start or manage an application.

It only controls whether matching network traffic is permitted.

---

## Troubleshooting Model

When an application running on EC2 cannot be reached, both the network layer and application layer should be investigated.

```text
Client
   |
   v
Network path
   |
   v
Security Group
   |
   v
EC2 operating system
   |
   v
Application/service
   |
   v
Listening port
```

For the Nginx lab, successful HTTP access required:

- A reachable EC2 instance.
- Appropriate network routing.
- A Security Group allowing TCP port 80.
- Nginx running.
- Nginx listening on TCP port 80.

---

## Key Learnings

- Security Groups act as stateful virtual firewalls for AWS resources such as EC2 instances.
- Inbound rules determine which incoming traffic is allowed.
- SSH commonly uses TCP port 22.
- HTTP commonly uses TCP port 80.
- `0.0.0.0/0` represents all IPv4 addresses and should only be used when appropriate.
- `/32` represents one specific IPv4 address.
- Restricting SSH to the required source IP reduces unnecessary exposure.
- A Security Group can be associated with multiple EC2 instances.
- Security Group permission alone does not guarantee that an application is available.
- An application must also be running and listening on the expected port.
- SSH authentication requires the correct credentials in addition to network access.
- The private key must remain secret.

---

## Security Notes

- The `.pem` private key is not included in this repository.
- No private key contents are documented.
- No AWS access keys or secret access keys are included.
- The actual public IPv4 address is represented using placeholders.
- SSH was not exposed to `0.0.0.0/0`.
- HTTP access to `0.0.0.0/0` was intentional for the public Nginx learning exercise.

