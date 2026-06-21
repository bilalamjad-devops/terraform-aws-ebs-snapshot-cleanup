# Stop Paying for Unused AWS EBS Snapshots Using Terraform and Lambda

*Automatically identify and delete stale EBS snapshots before they quietly inflate your AWS bill.*

---

## Introduction

As cloud environments grow, AWS costs can creep up in ways that aren't obvious at first glance. One of the most common culprits is forgotten Amazon EBS snapshots.

Teams routinely create snapshots before deploying a new release, performing maintenance, or making infrastructure changes — a quick rollback option in case something goes wrong. Once the work is done, the EC2 instance gets terminated and its EBS volume is deleted along with it. The snapshot, though, doesn't go anywhere. EBS snapshots exist independently of the volume they came from, so AWS keeps charging for that storage indefinitely — until someone notices and deletes it manually.

In a small account, that might be a few cents a month. In an organization where teams deploy and tear down infrastructure daily, snapshots accumulate fast, and "a few forgotten backups" turns into a real, recurring cost.

Instead of relying on someone to remember to clean these up, we can automate it with AWS Lambda. In this hands-on project, we'll build a serverless cleanup solution using **Terraform** to provision the infrastructure and **AWS Lambda** to scan the account and delete snapshots that are no longer tied to an existing volume.

This lab runs the Lambda function manually so we can verify the logic works correctly first. The same function can later be wired to an **Amazon EventBridge** schedule to run automatically — no manual trigger required.

---

## Business Problem

Imagine a team working on a staging environment.

Before deploying a new release, they create an EBS snapshot as a backup. Testing goes well, and the EC2 instance is terminated. AWS automatically deletes the attached EBS volume, because *Delete on Termination* is enabled by default.

The snapshot, though, is forgotten.

A month later, another deployment creates another snapshot. Then another. Then another.

None of these are needed anymore — but AWS keeps storing all of them, and keeps billing for it. In an organization where multiple teams deploy infrastructure every day, manually tracking which snapshots are still relevant becomes nearly impossible.

This is exactly the kind of repetitive cleanup task that should be automated rather than left as a chore someone has to remember.

---

## Solution

This project uses:

- **Terraform** to provision the infrastructure
- **AWS Lambda** to identify and delete stale snapshots
- **IAM Role and Policy** to grant the Lambda function only the permissions it needs
- **CloudWatch Logs** to record exactly what the function did and why

The workflow:

1. Terraform creates the Lambda function and its IAM role/policy.
2. We create a snapshot from an EC2 instance's root volume.
3. The EC2 instance is terminated, and its volume is deleted automatically.
4. The snapshot remains in the account, now orphaned.
5. Lambda scans every snapshot in the account.
6. If a snapshot's source volume no longer exists or isn't attached to anything, Lambda deletes it.
7. Every deletion is logged to CloudWatch.

## Steps We'll Follow

Here's the full path we'll walk through, so you know what's coming before diving in:

1. **Launch an EC2 instance** — to give ourselves something to snapshot
2. **Create an EBS snapshot** of its root volume — simulating a pre-deployment backup
3. **Terminate the EC2 instance** — its volume gets deleted automatically, leaving the snapshot orphaned
4. **Deploy the infrastructure with Terraform** — creates the IAM role, policy, and Lambda function
5. **Understand the Lambda function** — how it detects which snapshots are stale
6. **Verify the deployment** — confirm the function and its permissions in the console
7. **Test the solution** — manually trigger the function and watch it delete the stale snapshot
8. **Clean up** — `terraform destroy` to avoid leftover costs

By the end, you'll have a working Lambda function that can correctly identify and delete an orphaned EBS snapshot — and a clear idea of what to harden before running it against a real account.

## Architecture

```
EC2 Instance
      │
      ▼
Create Snapshot
      │
      ▼
Terminate EC2
      │
      ▼
Volume Deleted (Delete on Termination)
      │
      ▼
Snapshot Becomes Orphaned
      │
      ▼
AWS Lambda (scans & deletes stale snapshots)
      │
      ▼
CloudWatch Logs
```

## Technologies Used

- Terraform
- AWS Lambda
- Amazon EC2
- Amazon EBS
- IAM
- Amazon CloudWatch

## Prerequisites

- An AWS account
- [Terraform](https://developer.hashicorp.com/terraform/downloads) installed and on your PATH
- AWS CLI configured (`aws configure`) — avoid hardcoding access keys in `main.tf`, especially if this repo will be public

---

## Step 1 — Launch an EC2 Instance

To simulate a real scenario, we launch a temporary EC2 instance — think of it as a staging application server used briefly for testing:

<img width="1600" height="900" alt="EC2 launch instance configuration" src="https://github.com/user-attachments/assets/70195d33-ea0c-4b77-a5ad-a52362403585" />

<img width="1600" height="900" alt="EC2 instance launching" src="https://github.com/user-attachments/assets/cfbaa6a1-f23a-4f68-9801-421f5bcd8fff" />

<img width="1600" height="900" alt="EC2 instance running" src="https://github.com/user-attachments/assets/205604e8-b990-4305-8b76-89447d5d5187" />

---

## Step 2 — Create an EBS Snapshot

When EC2 launches an instance, AWS automatically provisions a root EBS volume for it — the disk holding the operating system. We'll take a snapshot of that volume, simulating a backup taken before a deployment:

Go to **EC2 → Elastic Block Store → Volumes**, select the root volume, then **Create Snapshot**.

<img width="1600" height="900" alt="Create snapshot screen" src="https://github.com/user-attachments/assets/0350eca3-53b0-4526-b513-956926bb1b0a" />

<img width="1600" height="900" alt="Selecting the volume to snapshot" src="https://github.com/user-attachments/assets/9eb1dac2-6bb4-41f1-b201-de06b4c73aeb" />

<img width="1600" height="900" alt="Snapshot creation confirmation" src="https://github.com/user-attachments/assets/d1c1024f-f373-410b-8217-4f08252bf74b" />

<img width="1600" height="900" alt="Snapshot listed as completed" src="https://github.com/user-attachments/assets/9c43ec01-fbe6-413d-90ea-be9949faddcf" />

A snapshot, unlike a volume, is **not** tied to the lifecycle of the instance or volume it came from — it's an independent resource AWS keeps storing (and billing for) on its own, until something explicitly deletes it.

---

## Step 3 — Terminate the EC2 Instance

The deployment is done, so the temporary server is no longer needed:

<img width="1600" height="900" alt="Terminating the EC2 instance" src="https://github.com/user-attachments/assets/4b679345-938a-44f6-b65f-fc31dab9e45c" />

Because *Delete on Termination* is enabled by default, AWS deletes the attached EBS volume along with the instance:

<img width="1600" height="900" alt="EBS volume deleted along with the instance" src="https://github.com/user-attachments/assets/b969d968-fb4d-44bd-a403-2afc264f5754" />

But the snapshot is still there:

<img width="1600" height="900" alt="Snapshot still present after volume deletion" src="https://github.com/user-attachments/assets/3a8e704f-5f53-4600-bc39-a8e3379ff813" />

At this point: the instance is gone, the volume is gone, and the snapshot is now an orphaned backup that continues consuming storage — exactly the resource our Lambda function is built to find and remove.

---

## Step 4 — Deploy the Infrastructure

```bash
terraform init
terraform plan
```

<img width="1600" height="900" alt="terraform init output" src="https://github.com/user-attachments/assets/c4ae3564-bc59-4f08-b139-a1f13e987309" />

<img width="1600" height="900" alt="terraform plan output" src="https://github.com/user-attachments/assets/3e899be1-13b9-4aa4-b016-75e9a1fe9800" />

Terraform creates the IAM role, the IAM policy, and the Lambda function itself.

**`main.tf`**

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.74.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1" # change to your preferred region
}

# --- IAM Role for the Lambda Function ---
resource "aws_iam_role" "ebs_snapshot_cleaner_role" {
  name = "DeleteStaleEBS_LambdaRole"

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

# --- IAM Policy: exactly what the function is allowed to do ---
resource "aws_iam_role_policy" "ebs_snapshot_cleaner_policy" {
  name = "DeleteStaleEBS_Policy"
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
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Effect   = "Allow"
        Resource = "*" # fine for a lab; scope this down to specific ARNs in production
      }
    ]
  })
}

# --- Lambda Code Packaging ---
data "archive_file" "ebs_snapshot_cleaner_zip" {
  type        = "zip"
  source_dir  = "${path.module}/python/"
  output_path = "${path.module}/python/lambda_function.zip"
}

# --- Lambda Function ---
resource "aws_lambda_function" "ebs_snapshot_cleaner_function" {
  function_name    = "DeleteStaleEBSSnapshot"
  role             = aws_iam_role.ebs_snapshot_cleaner_role.arn
  handler          = "delete_unused_snapshots.lambda_handler"
  runtime          = "python3.9"
  timeout          = 60
  memory_size      = 128

  filename         = data.archive_file.ebs_snapshot_cleaner_zip.output_path
  source_code_hash = data.archive_file.ebs_snapshot_cleaner_zip.output_base64sha256

  tags = {
    ManagedBy = "Terraform"
    Purpose   = "EBS Snapshot Cleaner"
  }
}
```

---

## Understanding the Lambda Function

The function does four things, in order:

1. Retrieves every EBS snapshot owned by the account.
2. For each one, checks whether its source volume still exists and is attached to anything.
3. If the volume is gone, or exists but isn't attached, the snapshot is considered stale.
4. Deletes the stale snapshot and logs the action to CloudWatch.

**`python/delete_unused_snapshots.py`**

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')

    # Get every EBS snapshot owned by this account
    response = ec2.describe_snapshots(OwnerIds=['self'])

    # Get all currently running EC2 instance IDs
    instances_response = ec2.describe_instances(
        Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
    )
    active_instance_ids = set()
    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            active_instance_ids.add(instance['InstanceId'])

    # Check each snapshot: delete it if its source volume is gone or unattached
    for snapshot in response['Snapshots']:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId')

        if not volume_id:
            # No volume reference at all — definitely orphaned
            ec2.delete_snapshot(SnapshotId=snapshot_id)
            print(f"Deleted EBS snapshot {snapshot_id} as it was not attached to any volume.")
        else:
            try:
                volume_response = ec2.describe_volumes(VolumeIds=[volume_id])
                if not volume_response['Volumes'][0]['Attachments']:
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as it was taken from a volume not attached to any running instance.")
            except ec2.exceptions.ClientError as e:
                if e.response['Error']['Code'] == 'InvalidVolume.NotFound':
                    # The source volume no longer exists at all
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as its associated volume was not found.")
```

> **Worth being upfront about:** this logic deletes any snapshot whose source volume is missing or unattached — it doesn't ask *why* the volume is gone. In a real account, someone might intentionally keep a snapshot as a long-term backup or disaster-recovery copy well after the original instance is terminated. Before running this against anything that matters, add a safeguard: only delete snapshots older than a chosen threshold (e.g. 30+ days), or require an explicit tag like `AutoCleanup: True` before a snapshot is even eligible for deletion. This lab keeps the simpler version to focus on the core mechanic.

---

## Step 5 — Verify the Deployment

Open the Lambda console. Terraform has created the function, with the IAM role attached giving it exactly the permissions defined above — describing snapshots and volumes, deleting snapshots, and writing logs:

<img width="1600" height="900" alt="Lambda function created with attached IAM role" src="https://github.com/user-attachments/assets/0cc1ed4d-cae3-4b6a-8cda-0b91b0502dec" />

---

## Step 6 — Test the Solution

Open the **Test** tab inside the Lambda console, create a test event (the name doesn't matter — e.g. `EBSStaleSnapshotEvent`, since this function doesn't read the event payload), and click **Test**:

<img width="1600" height="900" alt="Lambda test event configuration" src="https://github.com/user-attachments/assets/0aa69c1b-d78c-408a-8805-927200024863" />

<img width="1600" height="900" alt="Lambda test execution result" src="https://github.com/user-attachments/assets/ffcc2450-148a-4304-ae85-40f42be7e79a" />

<img width="1600" height="900" alt="CloudWatch log confirming snapshot deletion" src="https://github.com/user-attachments/assets/1e8a6d20-8c83-4c2d-9ff1-b7b415f9a195" />

The execution log confirms it: the function found the orphaned snapshot from Step 2 — left behind after we terminated its instance — and deleted it automatically. Refreshing the Snapshots page confirms it's gone.

---

## Step 7 — Clean Up

```bash
terraform destroy
```

<img width="1600" height="900" alt="terraform destroy output" src="https://github.com/user-attachments/assets/459099f5-502d-4b6a-a438-3ba1db0e5cad" />

<img width="1600" height="900" alt="terraform destroy completion" src="https://github.com/user-attachments/assets/87cc485b-9d69-45f5-958d-da39b71b4c8b" />

Confirm with `yes`. This removes the Lambda function, IAM role, and policy.

---

## Production Considerations

This project demonstrates the core cleanup logic with manual Lambda execution. To make it production-ready, consider:

- Triggering Lambda automatically on a schedule using Amazon EventBridge, instead of clicking Test manually
- Requiring an explicit tag (e.g. `AutoCleanup: True`) before a snapshot is eligible for deletion
- Adding an age threshold — only deleting snapshots that have been stale for 30+ days
- Sending an SNS notification after each cleanup run, summarizing what was deleted
- Scoping the IAM policy's `Resource` field down from `*` to specific ARNs

These changes turn a working lab into something safe enough to run against a real environment.

---

## Key Takeaways

In this project, we:

- Provisioned AWS infrastructure with Terraform
- Built a serverless cleanup function using AWS Lambda
- Identified orphaned EBS snapshots programmatically
- Reduced unnecessary AWS storage costs without manual auditing

The same pattern — find an orphaned resource, verify it's safe to remove, delete it — extends to other common AWS cost leaks: unattached EBS volumes, idle Elastic IPs, unused AMIs, or obsolete security groups. Once you've built this once, the rest follow the same shape.

---

Let's connect and share knowledge! You can find me on:

* **[GitHub](https://github.com/bilalamjad-devops)** — Check out my open-source projects and complete lab source code.
* **[LinkedIn](https://www.linkedin.com/in/bilalamjad-devops/)** — Let's connect professionally and discuss all things DevOps and Cloud Engineering.
* **[Medium](https://medium.com/@bilalamjad-devops)** — Follow my profile for more deep-dives, automation guides, and technical walkthroughs.

21-June-2026
