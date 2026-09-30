# AWS STS AssumeRole

## Objective

Practice using AWS Security Token Service (STS) to assume an IAM role and obtain temporary security credentials.

The lab also explored how a role's trust policy determines who is trusted to assume the role.

## Lab Architecture

The lab used an IAM user to assume an IAM role through AWS STS.

```text
IAM User
   |
   | sts:AssumeRole
   v
IAM Role
   |
   v
AWS STS
   |
   v
Temporary Security Credentials
   |
   v
Assumed Role Session
```

The temporary role session uses permissions granted to the assumed IAM role.

## Trust Policy

The IAM role contained a trust policy that directly trusted the IAM user.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:user/<IAM_USER>"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

The `Principal` identifies the IAM identity that the role trusts.

The `sts:AssumeRole` action allows the trusted principal to request an assumed-role session.

## Assuming the Role

The role was assumed using the AWS CLI.

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME> \
  --role-session-name <SESSION_NAME>
```

When the request succeeded, AWS STS returned temporary security credentials consisting of:

- Access key ID
- Secret access key
- Session token
- Expiration time

No real credentials or session tokens are included in this repository.

## Verifying the Assumed Role

After configuring the temporary credentials, the active AWS identity was verified using:

```bash
aws sts get-caller-identity
```

The returned ARN identified an assumed-role session rather than the original IAM user.

It followed a structure similar to:

```text
arn:aws:sts::<ACCOUNT_ID>:assumed-role/<ROLE_NAME>/<SESSION_NAME>
```

## Temporary Credentials

STS credentials are temporary and expire after a limited period.

When an expired set of temporary credentials was used during the lab, AWS rejected the request.

The role had to be assumed again to obtain a new set of valid temporary credentials.

## Trust Policy Experiment

The role was initially configured with a trust policy that directly named the IAM user as the trusted principal.

The IAM user was able to assume the role successfully.

The trust relationship was then changed so that the IAM user was no longer directly named as the trusted principal.

Without a corresponding identity-based permission authorizing the IAM user to call `sts:AssumeRole` on the role, the AssumeRole request was denied.

Restoring the direct IAM user trust allowed the AssumeRole request to succeed again.

This demonstrated that role assumption depends on the relationship between the caller's permissions and the role's trust policy.

## Trust Policy vs Permissions Policy

A trust policy and a permissions policy serve different purposes.

### Trust Policy

Determines who or what is trusted to assume the role.

```text
Who can assume the role?
```

### Permissions Policy

Determines what the resulting role session is allowed to do after the role has been assumed.

```text
What can the assumed role session do?
```

## What I Learned

- AWS STS can provide temporary security credentials through `sts:AssumeRole`.
- Temporary role credentials include an access key ID, secret access key, and session token.
- STS credentials expire and must be renewed by obtaining a new role session.
- A role trust policy defines which principals are trusted to assume the role.
- An assumed-role session uses permissions associated with the assumed role.
- Trust policies and permissions policies perform different functions.
- `aws sts get-caller-identity` can be used to verify which AWS identity is currently active.
- Temporary credentials reduce the need to rely on long-term credentials when accessing AWS resources.
