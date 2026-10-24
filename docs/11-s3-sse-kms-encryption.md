# Lab 06 — Amazon S3 Server-Side Encryption with AWS KMS (SSE-KMS)

## 1. Lab Overview

This hands-on lab demonstrates how to protect Amazon S3 objects using AWS Key Management Service (AWS KMS), configure a customer-managed KMS key, and troubleshoot access to encrypted objects using AWS Identity and Access Management (IAM).

The lab was performed using a fictional organization named **NovaTech**.

**AWS Region:** `ap-south-1` (Mumbai)

**Services used:**
- Amazon S3
- AWS Key Management Service (KMS)
- AWS Identity and Access Management (IAM)

## 2. Learning Objectives

- Understand server-side encryption in Amazon S3.
- Differentiate between AWS-managed and customer-managed KMS keys.
- Create and configure a customer-managed KMS key.
- Enable SSE-KMS as the default encryption for an S3 bucket.
- Understand the relationship between S3 permissions and KMS permissions.
- Troubleshoot `AccessDenied` errors involving encrypted S3 objects.
- Understand the consequences of disabling or deleting KMS keys.
- Safely clean up temporary AWS resources.

## 3. Lab Architecture

The lab involved the following components:

- **IAM user:** `dev-01`
- **S3 bucket:** `novatech-sse-kms-lab-20261022-01`
- **Customer-managed KMS key alias:** `alias/novatech-s3-kms-lab`
- **Test object 1:** `novatech-kms-test.txt`
- **Test object 2:** `novatech-cmk-test.txt`

### Architecture Flow

```text
IAM User (dev-01)
       |
       | s3:GetObject
       v
Amazon S3 Bucket
       |
       | SSE-KMS encrypted object
       v
AWS KMS
       |
       | kms:Decrypt authorization
       v
S3 decrypts and returns object data
```

For an SSE-KMS object download, the caller must have appropriate S3 access and permission to use the KMS key for decryption. The KMS key policy must also permit the authorization.

## 4. Creating the Customer-Managed KMS Key

Created a customer-managed KMS key with the following configuration:

| Setting | Configuration |
|---|---|
| Key type | Symmetric |
| Key usage | Encrypt and decrypt |
| Key material | AWS KMS generated |
| Regionality | Single-Region key |
| Alias | `novatech-s3-kms-lab` |
| Region | `ap-south-1` |

The key was initially enabled.

The key policy retained the default **Enable IAM User Permissions** statement, allowing the AWS account to delegate KMS permissions through IAM policies.

No additional key administrators or key users were explicitly selected during key creation.

## 5. Creating and Configuring the S3 Bucket

Created the S3 bucket:

`novatech-sse-kms-lab-20261022-01`

Configuration included:

- Bucket versioning disabled.
- Bucket kept private.
- Default encryption configured to use **SSE-KMS**.
- Customer-managed KMS key selected using the alias `alias/novatech-s3-kms-lab`.

This configuration enabled S3 to encrypt newly uploaded objects using the selected customer-managed KMS key by default, unless an upload explicitly specified a different supported encryption configuration.

## 6. Testing Different KMS Key Types

Uploaded two test objects:

| Object | Encryption key |
|---|---|
| `novatech-kms-test.txt` | AWS-managed KMS key (`aws/s3`) |
| `novatech-cmk-test.txt` | Customer-managed KMS key (`alias/novatech-s3-kms-lab`) |

Verified the encryption configuration in each object's **Properties** tab.

Both objects were successfully downloaded using the administrative AWS Console session.

### Key Difference

**AWS-managed KMS key (`aws/s3`):**
- Created and managed by AWS for the service.
- Key policy is managed by AWS.
- Offers less direct control over key management.

**Customer-managed KMS key:**
- Created and managed by the AWS customer.
- Supports customer-defined key policies.
- Allows more granular control over access, lifecycle, and rotation.
- Can incur a monthly key storage charge.

## 7. Testing IAM Access to Encrypted S3 Objects

Used the IAM user `dev-01` in a separate browser session.

Initially, the user did not have the required S3 permissions.

Attempting to list S3 buckets resulted in an access-denied error because the user lacked:

`s3:ListAllMyBuckets`

A custom read-only IAM policy was then created and attached to the IAM user.

The policy granted access to the specific lab bucket and its objects.

### Example S3 Read-Only IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListLabBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::novatech-sse-kms-lab-20261022-01"
    },
    {
      "Sid": "ReadLabObjects",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::novatech-sse-kms-lab-20261022-01/*"
    }
  ]
}
```

This allowed the user to list objects within the specific bucket and attempt object downloads.

It did not grant account-wide S3 bucket listing or KMS decryption permissions.

## 8. Troubleshooting KMS AccessDenied

After attaching the S3 read-only policy, `dev-01` could view the objects inside the bucket.

However, downloading the object encrypted with the customer-managed KMS key failed with an `AccessDenied` error.

### Root Cause

The IAM user had:

- `s3:ListBucket`
- `s3:GetObject`

But lacked:

- `kms:Decrypt`

For an SSE-KMS encrypted object, S3 permissions alone are insufficient. AWS KMS must also authorize use of the encryption key.

### Resolution

Created a customer-managed IAM policy named:

`NovaTech-KMS-Decrypt-Policy`

The policy granted `kms:Decrypt` on the specific customer-managed KMS key.

### Example KMS Decrypt Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowDecryptWithLabKey",
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "<CUSTOMER_MANAGED_KMS_KEY_ARN>"
    }
  ]
}
```

**Note:** The placeholder above represents the actual ARN of the lab KMS key. It must be replaced with the correct key ARN before using this example.

After attaching this policy to `dev-01`, the previously denied object download succeeded.

### Key Learning

Successful access to an SSE-KMS encrypted S3 object depends on both S3 authorization and KMS authorization.

A successful `s3:GetObject` permission check does not automatically grant `kms:Decrypt`.

## 9. Understanding KMS Key Policies and IAM Policies

AWS KMS uses key policies as a fundamental authorization mechanism.

In this lab, the default key policy allowed the AWS account to delegate access through IAM policies.

Therefore, granting `kms:Decrypt` to the IAM user through an identity-based IAM policy was sufficient for this test.

In other environments, permissions may also be affected by:

- Explicit denies.
- Service control policies (SCPs).
- Permissions boundaries.
- KMS key policies.
- Other applicable authorization controls.

**Security principle:** Grant only the permissions required for the intended operation.

## 10. Understanding KMS Key Deletion

A customer-managed KMS key cannot be deleted immediately.

AWS KMS requires a configurable waiting period of **7–30 days** before permanent deletion.

During the waiting period:

- The key enters the `Pending deletion` state.
- The key cannot be used for encryption or decryption.
- The scheduled deletion can be canceled.
- The monthly KMS key storage charge is not applied while the key is pending deletion.

After permanent deletion, the key material cannot be recovered.

Any remaining data encrypted under that key may become permanently unreadable unless another usable copy exists.

Creating a new KMS key with the same alias does not restore access to data encrypted with the deleted key.

### Production Consideration

Before deleting a customer-managed KMS key, verify that no important encrypted data or AWS resources still depend on it.

Where necessary, migrate or re-encrypt data using another active KMS key before deleting the original.

## 11. Lab Cleanup

After completing all access tests, the following cleanup actions were performed:

1. Detached the temporary KMS decrypt policy from `dev-01`.
2. Detached the temporary S3 read-only policy from `dev-01`.
3. Deleted both temporary customer-managed IAM policies.
4. Deleted the S3 test objects.
5. Deleted the empty S3 lab bucket.
6. Scheduled deletion of the customer-managed KMS key with a 7-day waiting period.

The existing IAM user `dev-01` was preserved for future labs.

**Final KMS key status:** `Pending deletion`

The KMS key was not immediately destroyed; AWS will permanently delete it after the configured waiting period unless deletion is canceled.

## 12. Lab Results

| Test | Result |
|---|---|
| Create customer-managed KMS key | PASS |
| Configure S3 SSE-KMS encryption | PASS |
| Upload and verify encrypted objects | PASS |
| Test S3 access without sufficient permissions | PASS |
| Grant bucket-specific read access | PASS |
| Reproduce missing `kms:Decrypt` error | PASS |
| Grant KMS decrypt permission | PASS |
| Successfully download the encrypted object | PASS |
| Delete temporary S3 and IAM resources | PASS |
| Schedule KMS key deletion | PASS |

## 13. Key Takeaways

1. Amazon S3 supports server-side encryption using AWS KMS.
2. AWS-managed and customer-managed KMS keys offer different levels of control.
3. S3 access and KMS access are separate authorization requirements.
4. `s3:GetObject` alone does not guarantee access to SSE-KMS encrypted data.
5. `kms:Decrypt` is required for the relevant customer-managed key when downloading an SSE-KMS object.
6. KMS key policies and IAM policies work together to control access.
7. Permanently deleting a KMS key can make encrypted data unrecoverable.
8. Temporary lab resources should be cleaned up to minimize unnecessary AWS charges.

## 14. Conclusion

This lab provided practical experience with Amazon S3 encryption, AWS KMS key management, IAM permissions, and troubleshooting encrypted object access.

The most important exercise was reproducing and resolving an `AccessDenied` error caused by missing KMS decryption permissions.

The lab also demonstrated why careful key lifecycle management is essential for protecting data in production environments.

**Lab Status:** Completed

**Environment:** Cleaned up; customer-managed KMS key pending scheduled deletion.

