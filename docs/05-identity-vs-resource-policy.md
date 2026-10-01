# Identity-Based vs Resource-Based Policies

## Objective

Understand how AWS evaluates permissions when both an identity-based policy and a resource-based policy apply to the same request.

This lab also demonstrates how to troubleshoot unexpected S3 access.

---

## Lab Setup

An EC2 instance used the IAM role:

`EC2-S3-ReadOnly-Role`

The S3 bucket used for the lab was:

`characters-bucket-0078`

The bucket contained two prefixes:

```text
characters-bucket-0078/
├── developers/
│   └── developers-test.txt
└── admins/
    └── admins-test.txt

## Lab Setup

The EC2 instance used the IAM role:

`EC2-S3-ReadOnly-Role`

The S3 bucket contained two prefixes:

- `developers/`
  - `developers-test.txt`
- `admins/`
  - `admins-test.txt`

The intended access was:

- `developers/` → List and download allowed
- `admins/` → List and download denied

## Identity-Based Policy

The `Characters-Bucket-0078_Read_Only` policy was attached to the `EC2-S3-ReadOnly-Role`.

The policy allowed `s3:ListBucket` only when the requested prefix matched `developers/*`:

```json
{
  "Effect": "Allow",
  "Action": "s3:ListBucket",
  "Resource": "arn:aws:s3:::characters-bucket-0078",
  "Condition": {
    "StringLike": {
      "s3:prefix": "developers/*"
    }
  }
}


This records the important distinction you learned:

```text
ListBucket → bucket-level resource + prefix condition
GetObject  → object-level resource restricted directly in the ARN

## Unexpected Result

After restricting the identity-based policy to `developers/*`, access to the `developers/` prefix worked as expected.

However, when testing the `admins/` prefix:

```bash
aws s3 ls s3://characters-bucket-0078/admins/

The admins-test.txt object was visible even though the identity-based policy did not allow listing the admins/ prefix.

## Troubleshooting and Root Cause

First, the active AWS identity was verified using:

```bash
aws sts get-caller-identity
```

The output confirmed that the EC2 instance was using:

`EC2-S3-ReadOnly-Role`

The role had only the `Characters-Bucket-0078_Read_Only` identity-based policy attached, and that policy was correctly restricted to `developers/*`.

The S3 bucket policy was then checked.

The bucket policy contained broader permissions that allowed the same EC2 role to:

- Perform `s3:ListBucket` on the entire `characters-bucket-0078` bucket.
- Perform `s3:GetObject` on `characters-bucket-0078/*`.

Therefore, the resource-based bucket policy was providing an applicable `Allow` for the `admins/` prefix even though the identity-based policy was restricted to `developers/*`.

## Permission Evaluation Lesson

The restricted identity-based policy did not explicitly deny access to `admins/`. It simply did not provide an `Allow` for that prefix.

This is an implicit deny.

The S3 bucket policy separately provided an applicable `Allow` for the same role across the entire bucket. Therefore, access to `admins/` was allowed.

A useful simplified model for this lab is:

```text
No applicable Allow
        ↓
Implicit Deny

Applicable Allow
        +
No Explicit Deny
        ↓
Allowed

Applicable Allow
        +
Applicable Explicit Deny
        ↓
Denied
```

An implicit deny is therefore different from an explicit deny.

AWS does not simply choose the most restrictive `Allow` policy. Applicable permissions from identity-based and resource-based policies are evaluated together, while an applicable explicit `Deny` overrides an `Allow`.

## Fix and Verification

The broad S3 bucket policy was removed so that access for this lab was controlled only by the identity-based policy attached to `EC2-S3-ReadOnly-Role`.

The following tests were then performed from the EC2 instance.

### Test 1: List the developers prefix

```bash
aws s3 ls s3://characters-bucket-0078/developers/
```

Result: **Allowed**

The `developers-test.txt` object was visible.

### Test 2: List the admins prefix

```bash
aws s3 ls s3://characters-bucket-0078/admins/
```

Result: **AccessDenied**

### Test 3: Download from developers

```bash
aws s3 cp s3://characters-bucket-0078/developers/developers-test.txt .
```

Result: **Allowed**

### Test 4: Download from admins

```bash
aws s3 cp s3://characters-bucket-0078/admins/admins-test.txt .
```

Result:

```text
403 Forbidden
```

The final behavior matched the intended least-privilege design:

```text
characters-bucket-0078/
│
├── developers/
│   └── developers-test.txt
│       ├── List ✅
│       └── Get  ✅
│
└── admins/
    └── admins-test.txt
        ├── List ❌
        └── Get  ❌
```

## Key Takeaways

- Identity-based and resource-based policies can both affect effective permissions.
- An implicit deny is the absence of an applicable `Allow`; it is not the same as an explicit `Deny`.
- An applicable resource-based `Allow` can grant access that a narrower identity-based `Allow` does not grant, subject to AWS policy-evaluation rules.
- An applicable explicit `Deny` overrides an `Allow`.
- `s3:ListBucket` is a bucket-level action and can be restricted using the `s3:prefix` condition.
- `s3:GetObject` is an object-level action and can be restricted using the object ARN.
- When troubleshooting `AccessDenied`, verify the active identity and inspect all applicable permission sources instead of changing policies immediately.

