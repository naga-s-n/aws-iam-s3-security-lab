# AWS IAM & S3 Security Lab

A hands-on AWS security lab focused on learning IAM and S3 access control through practical exercises.

## What I Practiced

- Created and worked with IAM users, groups, and roles to understand how AWS identities receive permissions.
- Applied the principle of least privilege by granting only the permissions required for specific S3 operations.
- Practiced S3 bucket-level and object-level permissions, including `s3:ListBucket` and `s3:GetObject`.
- Restricted S3 access to specific object prefixes using the `s3:prefix` condition and prefix-scoped object permissions.
- Tested implicit deny by attempting S3 operations that were not granted by any IAM policy.