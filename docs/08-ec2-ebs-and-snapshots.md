# EC2 EBS and Snapshots Lab

## Objective

Understand how Amazon EBS provides persistent block storage for EC2 instances and gain hands-on experience with:

- EBS root volumes
- EBS volume states
- Availability Zone relationships
- EBS volume types
- EBS snapshots
- Restoring a new EBS volume from a snapshot
- Cross-Availability-Zone restoration
- Resource cleanup

---

## Initial EC2 Storage

The EC2 instance used for this lab was:

`ec2-nginx-learning`

Its root EBS volume was configured as:

| Setting | Value |
|---|---|
| Volume size | 30 GiB |
| Volume type | gp3 |
| IOPS | 3000 |
| Encryption | Disabled |
| Availability Zone | ap-south-1b |
| AZ ID | aps1-az3 |

The root EBS volume contained the operating system, installed Nginx software, configuration files, and other data stored on the instance's root filesystem.

---

## Understanding EBS

Amazon Elastic Block Store (EBS) provides block storage that can be attached to EC2 instances.

Conceptually:

```text
EC2 Instance
     |
     |
     v
EBS Volume
     |
     +-- Operating system
     +-- Applications
     +-- Configuration
     +-- Files and data
```

Unlike temporary instance memory, EBS storage can persist independently of an EC2 instance's running state.

For example, stopping an EC2 instance does not automatically delete its EBS root volume.

---

## EBS Volume States

Two important EBS volume states were observed during this lab.

### In-use

```text
State: In-use
```

This means the EBS volume is currently attached to an EC2 instance.

The root volume of `ec2-nginx-learning` was shown as `In-use` while attached to the instance.

### Available

```text
State: Available
```

This means the EBS volume exists but is currently not attached to an EC2 instance.

A restored test volume created later in this lab entered the `Available` state because it was created without being attached to an instance.

---

## EBS and Availability Zones

An EBS volume belongs to a specific Availability Zone.

The original EC2 instance and its root EBS volume were located in:

```text
ap-south-1b
```

A normal EBS volume attachment requires the EC2 instance and EBS volume to be in the same Availability Zone.

For example:

```text
EC2
ap-south-1b
    |
    | attach
    v
EBS
ap-south-1b
```

is valid.

However:

```text
EC2
ap-south-1b
    |
    | X
    v
EBS
ap-south-1a
```

cannot normally be attached directly because the resources are in different Availability Zones.

---

## Understanding gp3

The root volume used the:

```text
gp3
```

EBS volume type.

`gp3` is a general-purpose SSD volume type suitable for many common workloads.

Three useful storage concepts were studied:

### Capacity

Capacity represents how much data the volume can store.

In this lab:

```text
30 GiB
```

### IOPS

IOPS means Input/Output Operations Per Second.

It represents the number of storage read/write operations that can be performed per second.

The root volume was configured with:

```text
3000 IOPS
```

### Throughput

Throughput represents the amount of data that can be transferred over a period of time and is commonly measured in MiB/s.

A useful distinction is:

```text
Capacity   = How much data can be stored
IOPS       = How many read/write operations can occur
Throughput = How much data can be transferred per second
```

---

## gp3 vs io2

The difference between general-purpose and provisioned-IOPS SSD storage was also studied.

### gp3

Suitable as a starting point for many general workloads where balanced price and performance are required.

### io2

Designed for workloads requiring high and consistent IOPS and high durability.

An important lesson was that a workload should not automatically use `io2` simply because it involves a database.

The storage type should be selected according to the actual performance, durability, and workload requirements.

---

## Creating an EBS Snapshot

A manual snapshot was created from the original EBS volume.

The snapshot was named/tagged:

`ec2-nginx-learning-snapshot`

An EBS snapshot is a point-in-time backup of an EBS volume's data.

Conceptually:

```text
EBS Volume
     |
     | Create snapshot
     v
EBS Snapshot
```

The snapshot can later be used to create another EBS volume.

---

## Snapshot Storage

EBS snapshots are stored using AWS-managed storage infrastructure.

Although AWS uses Amazon S3 infrastructure for EBS snapshots, the snapshots do not appear as normal objects inside an S3 bucket owned and managed through the standard S3 object interface.

Therefore:

```text
EBS Snapshot
     !=
Normal file/object in my S3 bucket
```

Snapshots are managed through the EBS snapshot functionality.

---

## Incremental Snapshots

EBS snapshots are incremental.

The first snapshot contains the blocks required for that initial snapshot.

Subsequent snapshots only need to store changed blocks that are not already represented in previous snapshots.

From the user's perspective, however, each snapshot can still be treated as a point-in-time recovery point.

AWS manages the underlying incremental storage relationships.

---

## Restoring a Volume from the Snapshot

To verify that a snapshot could be used to recreate storage, a new EBS volume was created from:

`ec2-nginx-learning-snapshot`

The restored volume was configured as:

```text
Volume type: gp3
Size:        30 GiB
AZ:          ap-south-1a
State:       Available
```

The restored volume was tagged:

`ec2-nginx-restored-volume`

Conceptually:

```text
Original EBS Volume
    ap-south-1b
         |
         | Snapshot
         v
EBS Snapshot
         |
         | Create volume
         v
New EBS Volume
    ap-south-1a
```

This demonstrated that a snapshot can be used to create a new EBS volume in another Availability Zone within the Region.

---

## Why the Restored Volume Could Not Be Attached to the Existing EC2 Instance

The original EC2 instance was located in:

```text
ap-south-1b
```

The restored test volume was deliberately created in:

```text
ap-south-1a
```

Because EBS volumes are Availability-Zone-specific, the restored `ap-south-1a` volume could not normally be attached directly to the existing `ap-south-1b` EC2 instance.

This experiment demonstrated an important distinction:

```text
EBS Volume
→ tied to one Availability Zone

EBS Snapshot
→ can be used to create volumes in different Availability Zones
   within the supported Region context
```

---

## Stop vs Terminate and EBS

Stopping an EC2 instance and terminating an EC2 instance have different implications.

### Stop

When the EC2 instance was stopped:

```text
EC2 compute stops
        |
        v
Root EBS volume remains
        |
        v
Stored data persists
```

The operating system, Nginx installation, configuration, and files remained on the root EBS volume.

After the instance was started again, that persistent disk data was still available.

### Terminate

An EBS volume can have:

```text
Delete on termination
```

enabled.

When this setting is enabled for a root volume, terminating the associated EC2 instance causes AWS to automatically delete that EBS volume.

This behavior was later verified during the custom AMI test using a temporary EC2 instance.

---

## Temporary Volume Cleanup

The restored volume:

`ec2-nginx-restored-volume`

was created only for learning and verification.

After confirming the snapshot restoration and Availability Zone behavior, the unused restored volume was deleted.

This prevented an unnecessary unattached EBS volume from being left in the AWS account.

A useful operational lesson is:

> Resources created for temporary testing should be reviewed and cleaned up when they are no longer required.

---

## Snapshot vs EBS Volume

The practical difference can be summarized as:

```text
EBS Volume
    |
    | Active block storage
    | Can be attached to EC2
    v
Used by an instance


EBS Snapshot
    |
    | Point-in-time backup
    | Used for recovery/copy
    v
Can create new EBS volumes
```

An EBS volume is usable block storage, while a snapshot is a backup representation that can be used to recreate a volume.

---

## Key Learnings

- EBS provides persistent block storage for EC2 instances.
- A root EBS volume can contain the operating system, applications, configuration, and files.
- Stopping an EC2 instance does not automatically delete its EBS volume.
- `In-use` means a volume is attached.
- `Available` means a volume exists but is currently unattached.
- EBS volumes are Availability-Zone-specific.
- An EBS volume normally needs to be in the same Availability Zone as the EC2 instance to which it is attached.
- `gp3` is a general-purpose SSD EBS volume type.
- Capacity, IOPS, and throughput describe different storage characteristics.
- EBS snapshots provide point-in-time backups of EBS volume data.
- EBS snapshots are incremental.
- A snapshot can be used to create a new EBS volume.
- A snapshot can be used to recreate a volume in another Availability Zone within the Region.
- A restored volume is a separate EBS volume from the original.
- `Delete on termination` controls whether an associated EBS volume is automatically deleted when an EC2 instance is terminated.
- Unused EBS volumes and snapshots should be reviewed because storage resources may continue to incur charges.

---

## Security and Cost Notes

- No AWS credentials are included in this documentation.
- No sensitive data from the EBS volume is published.
- The restored test volume was deleted after the experiment.
- The primary EC2 root volume was retained because the primary instance was only stopped.
- EBS volumes and snapshots are storage resources and should be monitored and cleaned up when no longer required.


