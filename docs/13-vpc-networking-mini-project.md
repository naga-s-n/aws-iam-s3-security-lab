# AWS VPC Networking Mini-Project

## 1. Project Overview

This project demonstrates the fundamentals of AWS Virtual Private Cloud (VPC) networking through a hands-on implementation.

The objective was to build a VPC containing public and private subnets, deploy EC2 instances in both subnets, configure network access controls, and test communication between the instances.

The project also explores how route tables, Security Groups, Network ACLs (NACLs), and ephemeral ports work together to control network traffic.

### Learning Objectives

- Understand VPC and subnet architecture.
- Configure public and private subnets.
- Understand route tables and local routing.
- Configure Security Groups and Network ACLs.
- Understand stateful and stateless traffic filtering.
- Connect to a private EC2 instance through a public EC2 instance (bastion host).
- Test internal HTTP communication between EC2 instances.
- Troubleshoot network connectivity issues.

---

## 2. Network Architecture

### VPC Configuration

| Component | Configuration |
|---|---|
| VPC CIDR | `10.0.0.0/16` |
| Public subnet | `10.0.1.0/24` |
| Private subnet | `10.0.2.0/24` |
| AWS Region | Asia Pacific (Mumbai) - `ap-south-1` |

### EC2 Instances

| Instance | Subnet | Private IP | Purpose |
|---|---|---|---|
| `vpc-web-server-practice` | Public | `10.0.1.20` | Bastion host and HTTP client |
| `private-ec2-nat-test` | Private | `10.0.2.145` | Private application server |

### Architecture Overview

The VPC contains two subnets:

- **Public subnet:** Hosts an EC2 instance with internet connectivity through an Internet Gateway. It also acts as a bastion host for accessing the private EC2 instance.
- **Private subnet:** Hosts an EC2 instance without a public IP address. It communicates with resources inside the VPC using private IP addresses.

Both subnets communicate through the VPC's local route.

The private subnet does not require a NAT Gateway for communication with the public EC2 instance.

---

## 3. Route Table Configuration

Route tables determine where network traffic is directed.

### Public Route Table

The public subnet's route table contained:

| Destination | Target | Purpose |
|---|---|---|
| `10.0.0.0/16` | local | Allows communication within the VPC |
| `0.0.0.0/0` | Internet Gateway | Routes internet-bound traffic through the Internet Gateway |

### Private Route Table

The private subnet's route table contained:

| Destination | Target | Purpose |
|---|---|---|
| `10.0.0.0/16` | local | Allows communication within the VPC |

No NAT Gateway route was configured in the final setup.

### Key Learning

The local route allows resources in different subnets of the same VPC to communicate using private IP addresses.

A route table provides the network path, but it does not independently authorize traffic. Security Groups and Network ACLs determine whether traffic is permitted.

---

## 4. Security Group Configuration

Security Groups act as virtual firewalls attached to EC2 instances. They control inbound and outbound traffic.

### Public EC2 — `web-server-sg-practice`

The public EC2 instance used the following outbound rules:

| Direction | Protocol | Port | Destination | Purpose |
|---|---|---:|---|---|
| Outbound | TCP | 22 | `private-ec2-nat-test-sg` | SSH access to private EC2 |
| Outbound | TCP | 443 | `0.0.0.0/0` | HTTPS connectivity |
| Outbound | TCP | 8080 | `private-ec2-nat-test-sg` | HTTP application connectivity test |

### Private EC2 — `private-ec2-nat-test-sg`

| Direction | Protocol | Port | Source | Purpose |
|---|---|---:|---|---|
| Inbound | TCP | 22 | `web-server-sg-practice` | SSH access from public EC2 |
| Inbound | TCP | 8080 | `web-server-sg-practice` | HTTP access from public EC2 |

### Key Learning

- Security Groups are stateful.
- They automatically allow response traffic for an allowed connection.
- Inbound rules control incoming traffic.
- Outbound rules control outgoing traffic.
- Security Groups can reference other Security Groups as traffic sources or destinations.

Security Groups control network access; SSH key authentication separately verifies the identity of the connecting user.

---

## 5. Network ACL Configuration

Network ACLs act as firewalls at the subnet level. They control inbound and outbound traffic entering and leaving a subnet.

### Public Subnet — `custom-nacl-practice`

**Inbound rules**

| Rule | Protocol | Port range | Source | Action |
|---:|---|---:|---|---|
| 100 | TCP | 80 | `0.0.0.0/0` | Allow |
| 110 | TCP | 22 | User public IP `/32` | Allow |
| 120 | TCP | 22 | `13.233.177.0/29` | Allow |
| 130 | TCP | 1024–65535 | `0.0.0.0/0` | Allow |
| * | All traffic | All | All | Deny |

**Outbound rules**

| Rule | Protocol | Port range | Destination | Action |
|---:|---|---:|---|---|
| 100 | TCP | 1024–65535 | `0.0.0.0/0` | Allow |
| 110 | TCP | 443 | `0.0.0.0/0` | Allow |
| 120 | TCP | 22 | `10.0.2.145/32` | Allow |
| 130 | TCP | 8080 | `10.0.2.145/32` | Allow |
| * | All traffic | All | All | Deny |

### Private Subnet

The private subnet used the default Network ACL, which allows all inbound and outbound traffic.

### Key Learning

- NACLs are stateless.
- Inbound and outbound traffic must be allowed separately.
- NACL rules are evaluated in ascending rule-number order.
- The first matching rule determines the action.
- The `*` rule is evaluated last and acts as the catch-all rule.

Unlike Security Groups, NACLs do not automatically allow response traffic.

---

## 6. Bastion Host — SSH Connectivity Test

A bastion host is an EC2 instance in a public subnet that provides a controlled entry point to resources in a private subnet.

### Architecture

The connection followed this path:

```text
Mac (Local Machine)
        |
        | SSH using public IP
        v
Public EC2 (Bastion Host)
        |
        | SSH using private IP
        v
Private EC2
```

### Connection Details

| Connection | Destination |
|---|---|
| Mac → Public EC2 | Public IP of the public EC2 |
| Public EC2 → Private EC2 | `10.0.2.145` |

### What Was Tested

1. Connected from the Mac to the public EC2 instance using SSH.
2. From the public EC2 instance, attempted SSH access to the private EC2.
3. Initially, the connection timed out because the public subnet's outbound NACL did not allow TCP port 22 to the private EC2.
4. Added an outbound NACL rule allowing TCP port 22 to `10.0.2.145/32`.
5. Retested the connection successfully.

### Key Learning

- A private EC2 instance can be accessed through a public EC2 bastion host without assigning it a public IP.
- The VPC local route provides the path between the subnets.
- Security Groups and NACLs must permit the required traffic.
- SSH agent forwarding was used to authenticate from the bastion host without copying the private key onto it.

---

## 7. Private HTTP Application Connectivity Test

The second practical test involved running a temporary HTTP server on the private EC2 instance and accessing it from the public EC2 instance.

### Objective

To verify that an EC2 instance in a public subnet can access an application running on an EC2 instance in a private subnet using its private IP address.

### HTTP Server Setup

A directory was created on the private EC2 instance:

```bash
mkdir -p ~/web-demo
```

A simple HTML file was created:

```bash
echo "Hello! This page is being served from the private EC2 instance." > ~/web-demo/index.html
```

The temporary Python HTTP server was started on port `8080`:

```bash
cd ~/web-demo && python3 -m http.server 8080 --bind 0.0.0.0
```

The server listened on port `8080` and accepted connections through the instance's network interfaces.

### Network Access Configuration

The following rules were configured:

| Component | Direction | Protocol | Port | Destination / Source |
|---|---|---|---:|---|
| Public EC2 Security Group | Outbound | TCP | 8080 | `private-ec2-nat-test-sg` |
| Private EC2 Security Group | Inbound | TCP | 8080 | `web-server-sg-practice` |
| Public subnet NACL | Outbound | TCP | 8080 | `10.0.2.145/32` |
| Public subnet NACL | Inbound | TCP | 1024–65535 | `0.0.0.0/0` |

The private subnet's default NACL already allowed inbound and outbound traffic.

### Connectivity Test

From the public EC2 instance, the following command was executed:

```bash
curl http://10.0.2.145:8080
```

The request successfully returned:

```text
Hello! This page is being served from the private EC2 instance.
```

This confirmed that the public EC2 instance could communicate with the HTTP server running on the private EC2 instance over the VPC's internal network.

The temporary Python HTTP server was stopped after the test using `Ctrl + C`.

### Understanding Request and Response Traffic

The HTTP request used:

| Field | Value |
|---|---|
| Source | Public EC2 |
| Source port | `48304` (ephemeral port example) |
| Destination | Private EC2 |
| Destination port | `8080` |

The response used the reverse port arrangement:

| Field | Value |
|---|---|
| Source | Private EC2 |
| Source port | `8080` |
| Destination | Public EC2 |
| Destination port | `48304` |

The response uses the ephemeral port originally selected by the client.

### Key Learning

- Port `8080` was the port on which the temporary Python HTTP server was listening.
- The public EC2 instance acted as an HTTP client.
- The private EC2 instance acted as an application server.
- The private EC2 instance did not need a public IP address or NAT Gateway for internal VPC communication.
- NACLs needed to permit both request and response traffic because they are stateless.
- Security Groups automatically allowed response traffic for the permitted connection because they are stateful.

---

## 8. Network Troubleshooting Observations

During the practical exercises, a few connectivity issues were encountered and resolved.

### SSH Connectivity Timeout

**Problem:** The public EC2 instance could not initially establish an SSH connection to the private EC2 instance.

**Cause:** The public subnet's outbound NACL did not have an allow rule for TCP port `22` to the private EC2 instance.

**Resolution:** Added an outbound NACL rule allowing TCP port `22` to `10.0.2.145/32`.

**Learning:** Even when Security Groups permit communication and a route exists, a restrictive NACL can still block traffic.

### HTTP Connectivity

**Problem:** The private HTTP application needed to be accessed from the public EC2 instance.

**Resolution:** Configured the relevant Security Group and NACL rules for TCP port `8080`, then tested connectivity using `curl`.

**Learning:** Successful application connectivity requires the route, Security Groups, NACLs, and application server to work together.

### Understanding Ephemeral Ports

The HTTP response was directed to the client's temporary source port rather than back to the server's listening port.

**Learning:** For stateless NACLs, the return traffic must be explicitly permitted. Allowing only the destination service port is not sufficient for every request-response flow.

---

## 9. Important Concepts Learned

| Concept | What I Learned |
|---|---|
| VPC | A logically isolated virtual network in AWS |
| Public subnet | A subnet whose route table provides a route to an Internet Gateway |
| Private subnet | A subnet without a direct route to an Internet Gateway |
| Local route | Enables communication between subnets within the same VPC |
| Route table | Determines where network traffic is directed |
| Security Group | Stateful virtual firewall associated with network interfaces |
| Network ACL | Stateless firewall associated with a subnet |
| Bastion host | A controlled entry point for accessing private resources |
| Ephemeral port | A temporary client-side port used for network communication |
| NAT Gateway | Enables private subnet resources to initiate outbound internet connections without assigning public IPs |

### Security Group vs Network ACL

| Feature | Security Group | Network ACL |
|---|---|---|
| Level | Instance / network interface | Subnet |
| Traffic filtering | Stateful | Stateless |
| Rules | Allow rules | Allow and deny rules |
| Rule evaluation | All applicable rules are considered | Rules evaluated in ascending order |
| Return traffic | Automatically allowed for permitted connections | Must be explicitly permitted |

---

## 10. Resource Cleanup

After completing the practical exercises, temporary resources were cleaned up to reduce unnecessary AWS costs.

### Cleanup Performed

- Deleted the temporary NAT Gateway.
- Released the Elastic IP associated with the NAT Gateway.
- Deleted the temporary VPC Flow Log.
- Deleted the associated CloudWatch log group.
- Stopped the temporary Python HTTP server using `Ctrl + C`.
- Stopped the private EC2 instance `private-ec2-nat-test`.
- Stopped the public EC2 instance `vpc-web-server-practice`.

### Important Cost Considerations

Stopping an EC2 instance stops its compute charges, but attached EBS volumes can continue to incur storage charges.

The VPC, subnets, route tables, Security Groups, and NACLs were retained for learning and portfolio documentation.

---

## 11. Conclusion

This mini-project provided hands-on experience with AWS VPC networking and demonstrated how different networking components work together.

The practical exercises covered communication between public and private subnets, SSH access through a bastion host, HTTP connectivity between EC2 instances, and troubleshooting using Security Groups and Network ACLs.

The project helped reinforce the distinction between network routing and traffic authorization, as well as the difference between stateful and stateless traffic filtering.

These concepts form an important foundation for understanding AWS networking, cloud infrastructure, and future Cloud/DevOps projects.

