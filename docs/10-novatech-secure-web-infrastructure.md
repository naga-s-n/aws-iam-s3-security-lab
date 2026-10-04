# NovaTech Solutions — Secure Web Infrastructure on AWS

## Project Overview

NovaTech Solutions requires a small public web application to be hosted on AWS.

The infrastructure must allow public access to the website while keeping administrative access restricted. The EC2 server must also be able to retrieve application files from a private S3 bucket without storing AWS access keys on the server.

The infrastructure should provide persistent storage, backup capability, and a reusable server image for recovery or redeployment.

---

## Business Requirements

The infrastructure was designed to meet the following requirements:

- Host a publicly accessible web application.
- Allow secure administrative SSH access.
- Store application files in a private S3 bucket.
- Prevent public access to the S3 bucket.
- Allow the EC2 instance to retrieve required S3 objects.
- Avoid storing AWS credentials on the EC2 instance.
- Follow the principle of least privilege.
- Use persistent EBS storage.
- Encrypt the EBS root volume.
- Create a point-in-time backup of the server disk.
- Create a reusable server image for recovery or redeployment.
- Validate that the infrastructure can be recovered successfully.

---

## Architecture

```text
Internet
   |
   | HTTP :80
   v
Security Group
   |
   v
EC2 Instance + Nginx
   |
   +---- Encrypted EBS Root Volume
   |          |
   |          +---- EBS Snapshot
   |
   +---- IAM Role
             |
             | s3:GetObject
             v
       Private S3 Bucket
```

A custom AMI was also created from the configured EC2 instance and tested by launching a temporary recovery instance.

---

## AWS Resources

### EC2

Primary instance:

`novatech-web-server`

The instance runs Ubuntu and Nginx and hosts the NovaTech Solutions web page.

Instance type used:

`t3.micro`

A public IPv4 address allows the website to be reached over the internet.

---

## Security Group

Security Group:

`novatech-web-sg`

Inbound access was restricted to the required traffic.

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| HTTP | 80 | 0.0.0.0/0 | Public website access |
| SSH | 22 | My public IP /32 | Administrative access |

SSH was not exposed to the entire internet.

---

## Private S3 Storage

S3 bucket:

`novatech-web-assets`

Amazon S3 Block Public Access was enabled.

Example objects were stored under prefixes such as:

```text
config/
images/
```

The configuration object used during testing was:

```text
config/site-info.txt
```

Attempting to access the object anonymously through its S3 Object URL returned:

```text
AccessDenied
```

This confirmed that the object was not publicly accessible.

---

## IAM Role and Least Privilege

IAM role:

`NovaTech-EC2-WebRole`

The role is trusted by the EC2 service and is attached to the EC2 instance.

Customer-managed IAM policy:

`NovaTech-WebAssets-ReadOnly`

The policy grants only the permission required by the web server:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::novatech-web-assets/*"
    }
  ]
}
```

Permissions such as `s3:PutObject` and `s3:DeleteObject` were not granted.

No AWS access keys were configured or stored on the EC2 instance.

The EC2 instance obtains temporary credentials by assuming its IAM role.

---

## IAM and S3 Validation

The active AWS identity on the EC2 instance was verified using:

```bash
aws sts get-caller-identity
```

The output showed an assumed-role identity associated with:

`NovaTech-EC2-WebRole`

S3 read access was tested using:

```bash
aws s3 cp s3://novatech-web-assets/config/site-info.txt .
```

The object was downloaded successfully.

A write operation was then deliberately attempted:

```bash
aws s3 cp upload-test.txt s3://novatech-web-assets/config/upload-test.txt
```

The request returned:

```text
AccessDenied
```

This demonstrated that the EC2 instance could read the required objects but could not upload new objects.

---

## Persistent Storage

The EC2 instance uses an Amazon EBS root volume with the following configuration:

```text
Size:       30 GiB
Type:       gp3
Encryption: Enabled
```

EBS provides persistent block storage for the EC2 instance.

---

## EBS Snapshot and Recovery Test

A point-in-time snapshot of the root EBS volume was created.

Snapshot name:

`novatech-web-server-snapshot`

To validate the backup, a new EBS volume was created from the snapshot in the same Availability Zone as the EC2 instance.

The restored volume successfully reached the `Available` state and was attached to the EC2 instance.

The operating system detected both the original root disk and the restored 30 GiB disk.

After validating the restoration process, the temporary restored volume was detached and deleted.

This demonstrated the workflow:

```text
EBS Volume
    |
    v
EBS Snapshot
    |
    v
Restored EBS Volume
    |
    v
Attach to EC2
```

---

## Custom AMI

A reusable custom Amazon Machine Image was created from the configured web server.

AMI name:

`novatech-web-server-ami`

The AMI captures the configured server disk, including the operating system, Nginx installation, and the NovaTech web page.

Infrastructure settings such as the Security Group, IAM role, instance type, subnet, and public IP are not stored as part of the AMI and must be selected when launching a new instance.

---

## AMI Recovery and Redeployment Test

A temporary EC2 instance was launched from:

`novatech-web-server-ami`

Recovery instance:

`novatech-web-server-recovery-test`

The recovery instance was configured with:

- `t3.micro` instance type
- Existing EC2 key pair
- `novatech-web-sg`
- `NovaTech-EC2-WebRole`
- Public IPv4 address
- No User Data

No manual Nginx installation or website configuration was performed on the recovery instance.

After the instance passed its status checks, its public IPv4 address was opened in a browser.

The previously configured NovaTech Solutions web page appeared successfully.

This confirmed that the custom AMI could be used to recreate the configured web server.

The temporary recovery instance was terminated after successful validation.

---

## Security Decisions

The project followed several security practices:

- S3 Block Public Access enabled.
- S3 objects kept private.
- IAM role used instead of storing AWS access keys on EC2.
- IAM permissions limited to `s3:GetObject`.
- S3 write access deliberately denied.
- SSH restricted to a single public IP using `/32`.
- Only HTTP port 80 exposed publicly.
- EBS root volume encrypted.
- Private SSH key not stored in GitHub.
- No AWS credentials or secrets included in project documentation.

---

## Recovery Strategy

Two different recovery mechanisms were tested.

### EBS Snapshot

Used to create a point-in-time backup of the EC2 disk.

```text
EBS Volume -> Snapshot -> Restored EBS Volume
```

### Custom AMI

Used to recreate the configured EC2 server.

```text
Configured EC2 -> Custom AMI -> New EC2 Instance
```

The snapshot protects the disk data, while the AMI provides a reusable server image for redeployment.

---

## Project Outcome

The NovaTech Solutions infrastructure successfully demonstrated:

- Public web hosting using EC2 and Nginx.
- Restricted administrative access using Security Groups.
- Private S3 object storage.
- IAM role-based AWS access without hard-coded credentials.
- Least-privilege S3 permissions.
- Encrypted persistent EBS storage.
- EBS snapshot creation and restoration.
- Custom AMI creation.
- Successful EC2 recovery from the custom AMI.

The project provides a simple infrastructure foundation that can later be expanded with services such as Load Balancing, Auto Scaling, databases, HTTPS, monitoring, and Infrastructure as Code as those concepts are learned.

