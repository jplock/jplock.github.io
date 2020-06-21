layout: page
title: "S3 Access Points for fun and profit"
date: 2020-06-20 20:24:00 +4000
categories: aws s3 access point vpc

TL;DR - S3 access points and an S3 VPC endpoint policy can be combined to allow access to private and public buckets without needing to constantly update the S3 VPC endpoint policy as new buckets are created.

[S3 Access Points](https://docs.aws.amazon.com/AmazonS3/latest/dev/access-points.html) were released at re:Invent 2019 as a way to enhance access to S3 buckets in your account. Each access point has its own IAM policy and can be attached to a VPC. Since S3 is a global service, with access points, we can now limit access to a bucket from within a specific VPC in an account.

At [work](https://www.thehartford.com), we wanted to limit S3 access through an S3 VPC endpoint to an approved list of public and private buckets. However, having to continually update the S3 VPC endpoint policy, whenever a new private bucket was created, was becoming a maintenance burden.

To solve the problem, we added the below policy to our S3 buckets to [delegate control](https://docs.aws.amazon.com/AmazonS3/latest/dev/creating-access-points.html#access-points-delegating-control) to access points:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Delegate access control to access points",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "*",
            "Resource": [
                "arn:aws:s3:::example-bucket",
                "arn:aws:s3:::example-bucket/*"
            ],
            "Condition": {
                "StringEquals": {
                    "s3:DataAccessPointAccount": "000000000000",
                    "s3:AccessPointNetworkOrigin": "VPC",
                    "aws:PrincipalOrgId": "o-0000000000"
                }
            }
        }
    ]
}
```

This policy allows all principals in the o-0000000000 organization to access the bucket if the request originates from an access point owned by account 000000000000 in a VPC.

We then added an access point to the _example-bucket_ called _my-access-point_ and added this S3 access point policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Allow access through access point",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:AbortMultipartUpload",
                "s3:DeleteObject*",
                "s3:GetObject*",
                "s3:ListBucket",
                "s3:ListMultipartUploadParts",
                "s3:PutObject*",
                "s3:RestoreObject"
            ],
            "Resource": [
                "arn:aws:s3:us-east-1:000000000000:accesspoint/my-access-point/object/*",
                "arn:aws:s3:us-east-1:000000000000:accesspoint/my-access-point"
            ],
            "Condition": {
                "StringEquals": {
                    "s3:DataAccessPointAccount": "000000000000",
                    "s3:AccessPointNetworkOrigin": "VPC",
                    "aws:PrincipalOrgId": "o-0000000000"
                }
            }
        }
    ]
}
```

This policy allows all principals in the o-0000000000 organization to access the bucket objects if the request originates from an access point owned by account 000000000000 in a VPC.

We then need to configure the S3 VPC endpoint policy to allow access to the access points and to the approved list of public buckets:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Allow account-owned access points",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:us-east-1:000000000000:accesspoint/*",
            "Condition": {
                "StringEquals": {
                    "s3:AccessPointNetworkOrigin": "VPC",
                    "aws:PrincipalOrgId": "o-0000000000",
                    "s3:DataAccessPointAccount": "000000000000"
                }
            }
        },
        {
            "Sid": "Allow access to public buckets",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::aws-ssm-us-east-1/*",
                "arn:aws:s3:::aws-windows-downloads-us-east-1/*",
                "arn:aws:s3:::amazon-ssm-us-east-1/*",
                "arn:aws:s3:::amazon-ssm-packages-us-east-1/*",
                "arn:aws:s3:::us-east-1-birdwatcher-prod/*",
                "arn:aws:s3:::aws-ssm-document-attachments-us-east-1/*",
                "arn:aws:s3:::patch-baseline-snapshot-us-east-1/*",
                "arn:aws:s3:::packages.us-east-1.amazonaws.com/*",
                "arn:aws:s3:::repo.us-east-1.amazonaws.com/*",
                "arn:aws:s3:::amazonlinux.us-east-1.amazonaws.com/*",
                "arn:aws:s3:::us-east-1.elasticmapreduce/*"
            ]
        },
        {
            "Sid": "Allow access to CloudFormation buckets",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::cloudformation-custom-resource-response-us-east-1/*",
                "arn:aws:s3:::cloudformation-waitcondition-us-east-1/*"
            ]
        }
    ]
}
```

The above policy allows object-level access to the account-owned S3 access points and to an approved list of public S3 buckets (for example, the [SSM Agent](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent-minimum-s3-permissions.html) and [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-vpce-bucketnames.html)). Bucket operations would not be able to traverse the VPC endpoint and must be done either in the Console or through the API.

I'm happy with the outcome as we are able to ensure private access to our buckets and continue to allow approved access to specific public buckets without any maintenance overhead.