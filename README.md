# velero-cluster-backup-restore-using-helm
EKS cluster backup and restor using Velero


Backup-and-Restore-EKS-using-Velero
Install jq:
yum install jq -y
Provide AWS_REGION=
export AWS_REGION=us-east-1
Create an S3 bucket to backup cluster:
If you are running this workshop in a region other than us-east-1, use the command below to create S3 bucket. Regions outside of us-east-1 require the appropriate LocationConstraint to be specified in order to create the bucket in the desired region.

export VELERO_BUCKET=$(aws s3api create-bucket \
--bucket eksworkshop-backup-$(date +%s)-$RANDOM \
--region $AWS_REGION \
--create-bucket-configuration LocationConstraint=$AWS_REGION \
--| jq -r '.Location' \
--| cut -d'/' -f3 \
--| cut -d'.' -f1)
For us-east-1, use the command below to create S3 bucket.

export VELERO_BUCKET=$(aws s3api create-bucket \
--bucket eksworkshop-backup-$(date +%s)-$RANDOM \
--region $AWS_REGION \
--| jq -r '.Location' \
--| tr -d /)  
Now, let’s save the VELERO_BUCKET environment variable into the bash_profile
echo "export VELERO_BUCKET=${VELERO_BUCKET}" | tee -a ~/.bash_profile
Create an IAM role Velero:
aws iam create-user --user-name velero
Attach policies to give velero the necessary permissions:
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
                "arn:aws:s3:::${VELERO_BUCKET}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${VELERO_BUCKET}"
            ]
        }
    ]
  }
EOF
Attach policy to velero IAM User:
 aws iam put-user-policy \
--user-name velero \
--policy-name velero \
--policy-document file://velero-policy.json
