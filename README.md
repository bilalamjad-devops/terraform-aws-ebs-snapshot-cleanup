# Delete Unused AWS EBS Snapshots Automatically Using Terraform, and AWS Lambda

Business Problem

in real-world companies, they crete snapshot of volume or they create golden image. afterthat they forget to delte snapshot and it silently occurs bills. 

in this article, we tackle then prblem of creating snapshot of volume and then forget to delete. 

Solution:

we can use eventbridge scheduler, lambda to tackel then problem. but in this article, we use only lambda to run manally to delte unused snapshot. 

Tech Stack:

terraform
lambda 
Prerequsites:
aws 
terraform 


# Launch an EC2 Instance:




<img width="1600" height="900" alt="aws lambda 7 " src="https://github.com/user-attachments/assets/70195d33-ea0c-4b77-a5ad-a52362403585" />


<img width="1600" height="900" alt="aws lambda 8 " src="https://github.com/user-attachments/assets/cfbaa6a1-f23a-4f68-9801-421f5bcd8fff" />

<img width="1600" height="900" alt="aws lambda 9" src="https://github.com/user-attachments/assets/205604e8-b990-4305-8b76-89447d5d5187" />

# 2. Create a Snapshot of its Root Volume:
<img width="1600" height="900" alt="aws lambda 10 " src="https://github.com/user-attachments/assets/0350eca3-53b0-4526-b513-956926bb1b0a" />

<img width="1600" height="900" alt="aws lambda 12" src="https://github.com/user-attachments/assets/9eb1dac2-6bb4-41f1-b201-de06b4c73aeb" />

<img width="1600" height="900" alt="aws lambda 13" src="https://github.com/user-attachments/assets/d1c1024f-f373-410b-8217-4f08252bf74b" />


<img width="1600" height="900" alt="aws lambda 15" src="https://github.com/user-attachments/assets/9c43ec01-fbe6-413d-90ea-be9949faddcf" />



# 3. Terminate the EC2 Instance (Makes Snapshot Stale):


<img width="1600" height="900" alt="aws lambda 16" src="https://github.com/user-attachments/assets/4b679345-938a-44f6-b65f-fc31dab9e45c" />

<img width="1600" height="900" alt="aws lambda 18" src="https://github.com/user-attachments/assets/b969d968-fb4d-44bd-a403-2afc264f5754" />



<img width="1600" height="900" alt="aws lambda 19" src="https://github.com/user-attachments/assets/3a8e704f-5f53-4600-bc39-a8e3379ff813" />


# 4. Terraform Init, plan


<img width="1600" height="900" alt="aws lambda 21" src="https://github.com/user-attachments/assets/c4ae3564-bc59-4f08-b139-a1f13e987309" />


<img width="1600" height="900" alt="aws lambda 22" src="https://github.com/user-attachments/assets/3e899be1-13b9-4aa4-b016-75e9a1fe9800" />




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



# 5. result

<img width="1600" height="900" alt="aws lambda 24" src="https://github.com/user-attachments/assets/0cc1ed4d-cae3-4b6a-8cda-0b91b0502dec" />




# 6. test

<img width="1600" height="900" alt="aws lambda 25" src="https://github.com/user-attachments/assets/0aa69c1b-d78c-408a-8805-927200024863" />

<img width="1600" height="900" alt="aws lambda 26" src="https://github.com/user-attachments/assets/ffcc2450-148a-4304-ae85-40f42be7e79a" />

<img width="1600" height="900" alt="aws lambda 27" src="https://github.com/user-attachments/assets/1e8a6d20-8c83-4c2d-9ff1-b7b415f9a195" />

# 7. tf destroy

<img width="1600" height="900" alt="aws lambda 28" src="https://github.com/user-attachments/assets/459099f5-502d-4b6a-a438-3ba1db0e5cad" />


<img width="1600" height="900" alt="aws lambda 29" src="https://github.com/user-attachments/assets/87cc485b-9d69-45f5-958d-da39b71b4c8b" />



