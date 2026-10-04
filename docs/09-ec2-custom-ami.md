# EC2 Custom AMI Lab

## Objective

Create a custom Amazon Machine Image (AMI) from an already configured EC2 instance and verify that the AMI can be used to launch a new independent EC2 instance with the existing software and disk configuration already present.

This lab demonstrates:

- Creating a custom AMI
- The relationship between AMIs and EBS snapshots
- AMI block-device mappings
- Launching a new EC2 instance from a custom AMI
- Verifying that installed software is preserved
- Understanding what an AMI does and does not copy
- Cleaning up a temporary test instance

---

## Source EC2 Instance

The custom AMI was created from the existing EC2 instance:

`ec2-nginx-learning`

The instance already contained:

- Ubuntu Server 26.04 LTS
- Nginx installed
- Nginx configured to start automatically
- A 30 GiB gp3 root EBS volume
- 3000 IOPS

The source EC2 instance was stopped when the custom AMI was created.

---

## Creating the Custom AMI

A custom AMI was created from the configured EC2 instance.

The image was named:

`ec2-nginx-learning-ami`

Description:

```text
Custom AMI of Ubuntu Nginx learning EC2 instance
```

The default reboot option was retained during image creation.

Because the source EC2 instance was already stopped, there were no active application writes occurring on the instance during the image creation process.

---

## AMI and EBS Snapshot Relationship

During AMI creation, Amazon EC2 created a snapshot of the instance's included EBS volume.

The source instance had:

```text
Root volume
Device: /dev/sda1
Size: 30 GiB
Type: gp3
IOPS: 3000
```

Conceptually:

```text
Configured EC2 Instance
        |
        | Create Image
        v
     Custom AMI
        |
        +---- Block-device mapping
        |
        +---- EBS snapshot
                  |
                  v
        Captured disk data
```

For an EBS-backed AMI, the AMI references EBS snapshot(s) containing the disk data required to create the instance volumes.

If multiple EBS volumes are included in the image, snapshots are created for the included volumes.

---

## AMI Block-Device Mapping

The custom AMI included a block-device mapping describing how storage should be created when a new EC2 instance is launched.

For this lab, the root volume configuration included:

```text
Size: 30 GiB
Type: gp3
IOPS: 3000
Delete on termination: Enabled
```

This explains why launching a new instance from the custom AMI automatically proposed a 30 GiB gp3 root volume.

The new instance does not share the original EC2 instance's EBS volume.

Instead, AWS creates a new EBS volume for the new instance using the snapshot associated with the AMI.

---

## Snapshot Verification

After the custom AMI became available, the EBS Snapshots page contained both:

```text
ec2-nginx-learning-snapshot
```

which was the manually created snapshot from the earlier EBS lab, and:

```text
ec2-nginx-learning-ami
```

which represented the snapshot created as part of the custom AMI process.

This demonstrated that manually created EBS snapshots and AMI-associated snapshots are separate resources created for different purposes.

---

## Launching a Test Instance from the Custom AMI

To verify the custom AMI, a temporary second EC2 instance was launched from:

`ec2-nginx-learning-ami`

The temporary instance was named:

`ec2-nginx-from-custom-ami`

The existing key pair was reused:

`ec2-nginx-learning-key`

The existing Security Group was also reused:

`ec2-nginx-learning-sg`

The new instance received its own public IPv4 address and its own EBS root volume.

---

## No Nginx Installation User Data

An important part of the experiment was that no User Data script was supplied to install Nginx.

The User Data field was deliberately left empty.

Therefore, the new instance was not given commands such as:

```bash
apt update
apt install nginx -y
systemctl enable nginx
systemctl start nginx
```

This was intentional.

The goal was to determine whether Nginx was already present in the disk state captured by the custom AMI.

---

## Verification

After the temporary EC2 instance entered the running state and passed its status checks, its new public IPv4 address was opened in a browser:

```text
http://<NEW-PUBLIC-IP>
```

The browser successfully displayed:

```text
Welcome to nginx!
```

This occurred even though no Nginx installation User Data was supplied to the new instance.

The result demonstrated that the custom AMI contained the disk state from the configured source instance, including the installed Nginx software and its configuration.

---

## Independence from the Source Instance

During the test, the original EC2 instance:

`ec2-nginx-learning`

remained in the:

```text
Stopped
```

state.

Despite the original instance being stopped, the new instance launched from the custom AMI successfully served the Nginx web page.

This demonstrated that the newly launched instance was not dependent on the original EC2 instance remaining online.

Conceptually:

```text
Original EC2
     |
     | Create custom AMI
     v
Custom AMI
     |
     +---- EBS snapshot(s)
     |
     | Launch
     v
New EC2 Instance
     |
     +---- New instance ID
     +---- New public IPv4 address
     +---- New root EBS volume
     +---- Nginx already installed
```

Once an AMI has been successfully created and remains registered with its required backing snapshots available, it can be used independently to launch new EC2 instances.

---

## What an AMI Captures

The custom AMI captures the machine image and disk state needed to reproduce the configured system.

In this lab, that included the disk contents containing:

- Ubuntu
- Nginx
- Nginx configuration
- Other files stored on the included EBS root volume

However, a new instance launched from an AMI is still a separate EC2 resource.

Properties such as the following are determined for the newly launched instance rather than simply cloning the source instance:

- Instance ID
- Public IPv4 address
- Instance type
- Subnet selection
- Security Group association
- Key pair selection

Therefore:

```text
AMI = reusable machine image

AMI != live clone of the original EC2 resource
```

---

## Launching Multiple Instances

A single custom AMI can be used to launch multiple EC2 instances.

For example:

```text
Custom AMI
    |
    +---- EC2 #1
    |
    +---- EC2 #2
    |
    +---- EC2 #3
    |
    +---- EC2 #4
    |
    +---- EC2 #5
```

Each instance receives its own EC2 identity and storage.

They do not all share one root EBS volume.

Because the required software is already captured in the AMI's disk image, each instance can start with that software already present.

---

## Temporary Instance Cleanup

After successfully verifying the AMI, the temporary instance:

`ec2-nginx-from-custom-ami`

was terminated.

Its root volume had:

```text
Delete on termination: Enabled
```

After termination, the EBS Volumes page was checked.

Only the original 30 GiB root volume belonging to the primary `ec2-nginx-learning` instance remained.

This verified that the temporary instance's root EBS volume was automatically deleted.

Conceptually:

```text
Temporary EC2
     |
     | Terminate
     v
Root EBS Volume
     |
     | Delete on termination = Yes
     v
Automatically deleted
```

---

## AMI Snapshot vs Instance Root Volume

The deletion of the temporary instance's root EBS volume did not delete the snapshot backing the custom AMI.

These are different resources.

```text
AMI backing snapshot
        |
        | used to create
        v
New root EBS volume
        |
        | attached to
        v
New EC2 instance
```

Terminating the new EC2 can delete its own root volume when `Delete on termination` is enabled, while the AMI and its required backing snapshot remain available.

---

## EBS Snapshot vs AMI

The difference can be summarized as:

```text
EBS Snapshot
    |
    +---- Point-in-time backup of EBS volume data
    |
    +---- Can be used to create new EBS volumes


AMI
    |
    +---- Reusable machine image/template
    |
    +---- Used to launch EC2 instances
    |
    +---- EBS-backed AMIs reference EBS snapshot(s)
```

An EBS snapshot focuses on backing up volume data.

An AMI provides the information required to launch EC2 instances from a reusable machine image.

---

## Key Learnings

- An AMI is a reusable machine image used to launch EC2 instances.
- A custom AMI can be created from an already configured EC2 instance.
- EBS-backed AMIs use EBS snapshots to preserve included disk data.
- An AMI contains block-device mappings describing storage for newly launched instances.
- Launching from an AMI creates a new EC2 instance rather than reconnecting to the original instance.
- The new instance receives its own root EBS volume.
- Installed applications such as Nginx can already be present when captured in the AMI.
- User Data is not required to reinstall software that is already captured in the machine image.
- A custom AMI can launch multiple independent EC2 instances.
- The source EC2 instance does not need to remain running for instances to be launched from an available AMI.
- Security Groups, instance types, networking choices, and other launch-time properties can be selected for the new instance.
- `Delete on termination` can automatically remove an instance's root EBS volume when the instance is terminated.
- Terminating an instance created from an AMI does not automatically delete the AMI's backing snapshot.

---

## Security and Cost Notes

- No private key contents are included in this repository.
- No AWS access keys or secret credentials are included.
- Public IPv4 addresses are represented using placeholders.
- The temporary EC2 instance was terminated after verification.
- Its temporary root EBS volume was automatically deleted.
- The primary learning EC2 instance remains stopped.
- AMI backing snapshots and manually created snapshots should be reviewed when they are no longer required because stored snapshots can incur charges.

