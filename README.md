# Velero

## Run Velero on AWS

To set up Velero on AWS, you:

Download an official release of Velero
Create your S3 bucket
Create an AWS IAM user for Velero
Install the server

**_If you do not have the aws CLI locally installed, follow the user [guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html) to set it up._**

## Download Velero

Download the latest official [release’s](https://github.com/vmware-tanzu/velero/releases) tarball for your client platform.

## Create S3 bucket

Velero requires an object storage bucket to store backups in, preferably unique to a single Kubernetes cluster (see the FAQ for more details). Create an S3 bucket, replacing placeholders appropriately:

``` bash
BUCKET=<YOUR_BUCKET>
REGION=<YOUR_REGION>
aws s3api create-bucket \
    --bucket $BUCKET \
    --region $REGION \
    --create-bucket-configuration LocationConstraint=$REGION
```

**_NOTE_**: us-east-1 does not support a LocationConstraint. If your region is us-east-1, omit the bucket configuration:

``` bash
aws s3api create-bucket \
    --bucket $BUCKET \
    --region $REGION
```

## Create IAM user

For more information, see the AWS documentation on IAM users.

Create the IAM user:

``` bash
aws iam create-user --user-name velero
```

If you’ll be using Velero to backup multiple clusters with multiple S3 buckets, it may be desirable to create a unique username per cluster rather than the default velero.

``` bash
{
    "User": {
        "Path": "/",
        "UserName": "velero",
        "UserId": "USER ID",
        "Arn": "ARN",
        "CreateDate": "DATE"
    }
}
```

Attach policies to give velero the necessary permissions:

``` bash
cat > velero-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}"
            ]
        }
    ]
}
EOF
```

``` bash
iam put-user-policy \
  --user-name velero \
  --policy-name velero \
  --policy-document file://velero-policy.json
```

view the policy at velero-policy.json

Create access keys:

``` bash
aws iam create-access-key --user-name velero

{
    "AccessKey": {
        "UserName": "velero",
        "AccessKeyId": "<AWS_ACCESS_KEY_ID>",
        "Status": "Active",
        "SecretAccessKey": "<AWS_SECRET_ACCESS_KEY>",
        "CreateDate": "2022-05-30T09:04:17+00:00"
    }
}
```

Create a Velero-specific credentials file (credentials-velero) in your local directory:

``` bash
[default]
aws_access_key_id=<AWS_ACCESS_KEY_ID>
aws_secret_access_key=<AWS_SECRET_ACCESS_KEY>
```

## Install and start Velero

Install Velero, including all prerequisites, into the cluster and start the deployment. This will create a namespace called velero, and place a deployment named velero in it.

``` bash
velero install \
    --provider aws \
    --bucket $BUCKET \
    --secret-file ./credentials-velero \
    --backup-location-config region=$REGION \
    --snapshot-location-config region=$REGION \
    --plugins velero/velero-plugin-for-aws:v1.0.0 \
    --use-restic \
```
