# S3 Prefix-Based Access Control

## Objective

Practice restricting S3 access so that a developer can list and read objects only within the `developers/` prefix while preventing access to other prefixes such as `uploads/`.

## Access Design

The S3 bucket contained different object prefixes for different purposes.

- `developers/` — accessible to the developer role
- `uploads/` — not accessible to the developer role

## How the Permissions Work

Two S3 permissions were used for the `developers/` prefix.

### List Objects

`s3:ListBucket` allows objects in the bucket to be listed.

The `s3:prefix` condition restricts the listing to the `developers/` prefix.

### Read Objects

`s3:GetObject` allows objects to be read or downloaded.

The resource was restricted to objects under:

`arn:aws:s3:::<BUCKET_NAME>/developers/*`

## Test Results

The prefix restrictions were tested using the AWS CLI.

- Listing the `developers/` prefix — **Allowed**
- Listing the `uploads/` prefix — **Denied**
- Downloading an object from `developers/` — **Allowed**
- Downloading an object from `uploads/` — **Denied**

## What I Learned

- S3 prefixes can be used to organize and restrict access to groups of objects.
- `s3:ListBucket` is a bucket-level permission used for listing objects.
- The `s3:prefix` condition can restrict which object prefixes can be listed.
- `s3:GetObject` is an object-level permission used for reading or downloading objects.
- Prefix-based permissions help apply the principle of least privilege by limiting access to only the required objects.
