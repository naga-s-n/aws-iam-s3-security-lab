Implement and test an Amazon S3 bucket policy that enforces HTTPS-only access, while using least-privilege IAM permissions to control bucket listing, object downloads, and uploads.
This lab demonstrates:
- Identity-based policies versus resource-based policies
- Implicit Deny versus explicit Deny
- HTTPS-only access using aws:SecureTransport
- Bucket-level versus object-level permissions
- Troubleshooting AWS AccessDenied errors
- Secure cleanup of temporary AWS resources
The S3 bucket was configured with:
- ACLs disabled
- Block All Public Access enabled
- Versioning disabled
- Default encryption: SSE-S3
All resources were temporary and removed after testing.
The following resource-based policy was applied to the S3 bucket.
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::novatech-bucket-policy-lab-20261023-01",
        "arn:aws:s3:::novatech-bucket-policy-lab-20261023-01/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false",
          "aws:PrincipalIsAWSService": "false"
        }
      }
    }
  ]
}
```
- Effect: Deny explicitly blocks matching requests.
- Principal: "*" applies the statement broadly to principals.
- Action: "s3:*" covers S3 actions against the specified resources.
- Resource includes both the bucket ARN and its object ARNs.
- aws:SecureTransport: false matches requests using insecure HTTP.
- aws:PrincipalIsAWSService: false excludes direct AWS service principals from this Deny statement.
This bucket policy does not grant access. It restricts requests that would otherwise be authorized.
Security principle: An explicit Deny overrides an Allow.
A harmless text file named novatech-https-test.txt was uploaded to the S3 bucket.
Contents:
```
NovaTech S3 Bucket Policy Lab
Testing HTTPS-only access using an S3 bucket policy.
```
Opening the private object's HTTPS URL without authentication returned AccessDenied.
This was expected because HTTPS alone does not grant permission to access a private S3 object.
Command:
```
aws s3 ls s3://novatech-bucket-policy-lab-20261023-01 \
  --region ap-south-1
```
The request failed with AccessDenied.
AWS indicated that no identity-based policy allowed the s3:ListBucket action.
The IAM user dev-01 did not have permission to list objects in the lab bucket.
This demonstrated an implicit Deny caused by the absence of a required Allow permission.
A customer-managed IAM policy was created and attached to dev-01.
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListNovaTechLabBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::novatech-bucket-policy-lab-20261023-01"
    }
  ]
}
```
The policy grants permission to list objects only in the specified bucket.
After attaching the policy, the same AWS CLI listing command succeeded and displayed novatech-https-test.txt.
An authorized HTTPS request succeeded because the IAM policy allowed the operation and the bucket policy's HTTP Deny condition did not match.
The following command deliberately specified an HTTP endpoint:
```
aws s3 ls s3://novatech-bucket-policy-lab-20261023-01 \
  --region ap-south-1 \
  --endpoint-url http://s3.ap-south-1.amazonaws.com
```
AWS returned AccessDenied, identifying:
```
with an explicit deny in a resource-based policy
```
The request matched the bucket policy's insecure-transport Deny condition.
Even though dev-01 had s3:ListBucket permission, the explicit Deny in the bucket policy overrode that Allow.
HTTP does not encrypt traffic. This was a controlled test involving a temporary lab resource and should not be used for production workloads or sensitive credentials.
Command:
```
aws s3 cp \
  s3://novatech-bucket-policy-lab-20261023-01/novatech-https-test.txt \
  ./novatech-download-test.txt \
  --region ap-south-1
```
The AWS CLI returned:
```
An error occurred (403) when calling the HeadObject operation: Forbidden
```
The AWS CLI may call HeadObject to retrieve object metadata before downloading.
For this operation, s3:GetObject permission is required.
The IAM policy initially granted only s3:ListBucket, so the download was not authorized.
The generic 403 Forbidden message did not independently identify the exact policy responsible.
The existing customer-managed IAM policy was updated to:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListNovaTechLabBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::novatech-bucket-policy-lab-20261023-01"
    },
    {
      "Sid": "AllowGetNovaTechTestObject",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::novatech-bucket-policy-lab-20261023-01/novatech-https-test.txt"
    }
  ]
}
```
The policy now grants:
- s3:ListBucket on the lab bucket.
- s3:GetObject on one specific test object.
It does not grant general upload, delete, or administrative permissions.
The download command succeeded.
The downloaded file was verified using:
```
cat novatech-download-test.txt
```
Output:
```
NovaTech S3 Bucket Policy Lab
Testing HTTPS-only access using an S3 bucket policy.
```
Command:
```
aws s3 cp \
  novatech-download-test.txt \
  s3://novatech-bucket-policy-lab-20261023-01/unauthorized-upload-test.txt \
  --region ap-south-1
```
The upload failed with AccessDenied.
AWS indicated:
```
because no identity-based policy allows the s3:PutObject action
```
The IAM policy did not include s3:PutObject.
Read access does not automatically grant write access.
This was another example of implicit Deny.
An IAM policy attached to a user, group, or role that specifies which actions the identity is allowed or denied.
A policy attached to an AWS resource, such as an S3 bucket, that controls access to that resource.
Access is denied when no applicable Allow grants the requested permission.
A matching Deny statement directly blocks the requested operation, even if another policy contains an Allow.
Grant only the permissions necessary for the required task, scoped to the appropriate resources.
Use an S3 bucket policy with aws:SecureTransport to explicitly deny insecure requests.
After completing the tests:
1. Detached NovaTech-S3-ListBucket-Lab-Policy from dev-01.
2. Deleted the temporary customer-managed IAM policy.
3. Emptied the S3 lab bucket.
4. Deleted novatech-bucket-policy-lab-20261023-01.
All four cleanup operations completed successfully.
This lab provided hands-on experience with:
- Creating and testing an HTTPS-only S3 bucket policy.
- Understanding how identity-based and resource-based policies interact.
- Identifying implicit and explicit Deny conditions from AWS error messages.
- Applying bucket-level and object-level permissions.
- Troubleshooting ListObjectsV2, HeadObject, and PutObject authorization failures.
- Verifying least-privilege access using the AWS CLI.
- Cleaning up temporary AWS security lab resources.
Final takeaway: Secure S3 access requires both appropriate permissions and correctly configured security restrictions. An IAM Allow does not override an applicable explicit Deny.
