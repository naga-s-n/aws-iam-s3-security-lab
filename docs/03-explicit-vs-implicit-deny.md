# Implicit Deny vs Explicit Deny in AWS IAM

## Objective

Understand and test the difference between implicit deny and explicit deny in AWS IAM by performing S3 operations with different permission configurations.

## Implicit Deny

In AWS IAM, an action is implicitly denied when there is no applicable `Allow` granting permission to perform that action.

For example, if an IAM role has permission to list and read S3 objects but does not have `s3:DeleteObject`, an attempt to delete an object is denied.

The role may have permissions such as:

- `s3:ListBucket` — Allowed
- `s3:GetObject` — Allowed
- `s3:DeleteObject` — Not allowed by any policy

Therefore, attempting a delete operation results in an implicit deny.

### Test

A delete operation was attempted using the AWS CLI:

```bash
aws s3 rm s3://<BUCKET_NAME>/<OBJECT_KEY>
```

Result: **Denied**

No policy granted the `s3:DeleteObject` permission, so the request was implicitly denied.

## Explicit Deny

An explicit deny occurs when a policy contains an `Effect` of `Deny` for an action.

During the lab, `s3:PutObject` was first allowed so that objects could be uploaded.

An explicit deny for `s3:PutObject` was then applied.

This created the following situation:

```text
Allow s3:PutObject
        +
Deny s3:PutObject
        ↓
Explicit Deny wins
        ↓
Upload denied
```

### Test

An upload operation was attempted using the AWS CLI:

```bash
aws s3 cp <LOCAL_FILE> s3://<BUCKET_NAME>/<OBJECT_KEY>
```

Result: **Denied**

Even though another policy allowed `s3:PutObject`, the applicable explicit deny overrode the allow.

## Permission Evaluation

The experiments demonstrated two important IAM permission-evaluation rules.

### No Applicable Allow

```text
No applicable Allow
        ↓
Implicit Deny
        ↓
Request denied
```

### Allow and Explicit Deny

```text
Applicable Allow
      +
Applicable Explicit Deny
      ↓
Explicit Deny
      ↓
Request denied
```

An applicable explicit deny takes precedence over an applicable allow.

## What I Learned

- AWS IAM denies requests by default unless an applicable permission allows them.
- If no applicable allow exists for an action, the request is implicitly denied.
- An explicit deny is created using `"Effect": "Deny"`.
- An applicable explicit deny overrides an applicable allow.
- Testing both successful and denied operations helps verify that IAM permissions follow the intended least-privilege design.
