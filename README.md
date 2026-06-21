# terraform-aws-lambda-snapshot









main.tf
```terraform
# main.tf
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "5.74.0"
    }
  }
}

# Configuration options
provider "aws" {
  region     = "ap-south-1"
  access_key = "paste-your-access-key"     # 👈 paste your access key
  secret_key = "paste-your-secret-key"     # 👈 paste your secret key 
}

# --- IAM Role for Lambda Function ---
# This role grants the Lambda function permission to interact with other AWS services.
resource "aws_iam_role" "ebs_snapshot_cleaner_role" {
  name = "DeleteStaleEBS_LambdaRole" # Unique name for the role

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    ManagedBy = "Terraform"
    Purpose   = "EBS Snapshot Cleaner"
  }
}

# --- IAM Policy for Lambda Role ---
# This policy defines the specific actions the Lambda function is allowed to perform on AWS resources,
# including logging its activity to CloudWatch Logs.
resource "aws_iam_role_policy" "ebs_snapshot_cleaner_policy" {
  name = "DeleteStaleEBS_Policy" # Unique name for the policy
  role = aws_iam_role.ebs_snapshot_cleaner_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "ec2:DescribeSnapshots",
          "ec2:DeleteSnapshot",
          "ec2:DescribeInstances",
          "ec2:DescribeVolumes",
          "logs:CreateLogGroup",   # Required for CloudWatch Logs
          "logs:CreateLogStream",  # Required for CloudWatch Logs
          "logs:PutLogEvents"      # Required for CloudWatch Logs
        ]
        Effect   = "Allow"
        Resource = "*" # Allows actions on all resources; consider narrowing for production.
      }
    ]
  })
}

# --- Lambda Code Packaging ---
# This data source creates a zip file containing the Python code for the Lambda function.
# It automatically reads from the 'lambda_code' directory.
data "archive_file" "ebs_snapshot_cleaner_zip" {
  type        = "zip"
  source_dir  = "${path.module}/python/" # Path to your Lambda function's Python code
  output_path = "${path.module}/python/lambda_function.zip" # Output zip file name
}

# --- Lambda Function ---
# Defines the AWS Lambda function that runs the snapshot cleanup logic.
resource "aws_lambda_function" "ebs_snapshot_cleaner_function" {
  function_name    = "DeleteStaleEBSSnapshot" # Name of the Lambda function in AWS
  role             = aws_iam_role.ebs_snapshot_cleaner_role.arn
  handler          = "lambda_function.lambda_handler" # Entry point (file.function_name)
  runtime          = "python3.9" # Python runtime version
  timeout          = 60          # Function timeout in seconds (1 minute) for lab testing
  memory_size      = 128 # Memory allocated to the function (MB)

  filename         = data.archive_file.ebs_snapshot_cleaner_zip.output_path
  source_code_hash = data.archive_file.ebs_snapshot_cleaner_zip.output_base64sha256

  tags = {
    ManagedBy = "Terraform"
    Purpose   = "EBS Snapshot Cleaner"
  }
}
```


python/delete_unused_snapshots.py

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')

    # Get all EBS snapshots
    response = ec2.describe_snapshots(OwnerIds=['self'])

    # Get all active EC2 instance IDs
    instances_response = ec2.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    active_instance_ids = set()

    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            active_instance_ids.add(instance['InstanceId'])

    # Iterate through each snapshot and delete if it's not attached to any volume or the volume is not attached to a running instance
    for snapshot in response['Snapshots']:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId')

        if not volume_id:
            # Delete the snapshot if it's not attached to any volume
            ec2.delete_snapshot(SnapshotId=snapshot_id)
            print(f"Deleted EBS snapshot {snapshot_id} as it was not attached to any volume.")
        else:
            # Check if the volume still exists
            try:
                volume_response = ec2.describe_volumes(VolumeIds=[volume_id])
                if not volume_response['Volumes'][0]['Attachments']:
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as it was taken from a volume not attached to any running instance.")
            except ec2.exceptions.ClientError as e:
                if e.response['Error']['Code'] == 'InvalidVolume.NotFound':
                    # The volume associated with the snapshot is not found (it might have been deleted)
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as its associated volume was not found.")
```

