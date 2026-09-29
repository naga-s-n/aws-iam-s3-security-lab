# IAM Role Access to Amazon S3

## Objective

Practice granting an EC2 instance access to an S3 bucket through an IAM role and verify which S3 operations are allowed or denied.

## Lab Architecture

The EC2 instance used an IAM role to access the S3 bucket without storing AWS access keys on the instance.

EC2 Instance
→ IAM Role
→ IAM Permissions Policy
→ S3 Bucket

## Permissions Tested

The IAM role was configured with read-only access to the S3 bucket.

The following S3 operations were tested:

- `s3:ListBucket` — Allowed
- `s3:GetObject` — Allowed
- `s3:PutObject` — Denied
- `s3:DeleteObject` — Denied

## CLI Testing

The permissions were tested from the EC2 instance using the AWS CLI.

### List Objects

The following command was used to list objects in the S3 bucket:

```bash
aws s3 ls s3://<BUCKET_NAME>
```

Result: **Allowed**

### Download Object

The following command was used to download an object:

```bash
aws s3 cp s3://<BUCKET_NAME>/<OBJECT_KEY> .
```

Result: **Allowed**

### Upload Object

An upload operation was attempted to verify that the read-only IAM role could not write objects to the bucket.

```bash
aws s3 cp <LOCAL_FILE> s3://<BUCKET_NAME>/<OBJECT_KEY>
```

Result: **Denied**

The operation was denied because the IAM role did not have the `s3:PutObject` permission.

### Delete Object

A delete operation was attempted to verify that the IAM role could not delete objects from the bucket.

```bash
aws s3 rm s3://<BUCKET_NAME>/<OBJECT_KEY>
```

Result: **Denied**

The operation was denied because the IAM role did not have the `s3:DeleteObject` permission.

## What I Learned

- EC2 instances can use IAM roles to access AWS services without storing long-term IAM user access keys on the instance.
- `s3:ListBucket` controls the ability to list objects in a bucket.
- `s3:GetObject` controls the ability to read or download objects.
- Permissions that are not explicitly allowed are implicitly denied.
- Least privilege means granting only the permissions required for a specific task.
