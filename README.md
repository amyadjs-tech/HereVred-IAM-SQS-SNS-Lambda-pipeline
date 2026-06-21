# HereVred-IAM-SQS-SNS-Lambda-pipeline
Lambda Triggered by SQS with SNS Notification

# Step 1 — IAM: Create the Lambda Execution Role

Lambda needs an IAM role to:
- Be assumed by the Lambda service
- Read messages from SQS (and delete them after processing)
- Publish messages to SNS
- Write logs to CloudWatch

---

## 1.1 Open IAM

1. In the AWS Console, search for **IAM** and open it.
2. In the left sidebar click **Roles** → **Create role**.

---

## 1.2 Configure the Trust Policy

| Field | Value |
|-------|-------|
| Trusted entity type | **AWS service** |
| Use case | **Lambda** |

Click **Next**.

## Steps 1.1 and 1.2 are shown in below 3 pictures

<img width="1917" height="922" alt="image" src="https://github.com/user-attachments/assets/b7192f18-84a2-4d80-adf6-6eb88cc6a457" />

<img width="1912" height="930" alt="image" src="https://github.com/user-attachments/assets/33d4fb1a-613a-4c57-8dd5-3ce3c00f2526" />

<img width="1917" height="921" alt="image" src="https://github.com/user-attachments/assets/5f06441c-e8cf-4429-9c45-4939e171695c" />



---

## 1.3 Attach Permissions

Search for and attach these two AWS managed policies:

| Policy name | Why |
|-------------|-----|
| `AWSLambdaSQSQueueExecutionRole` | Grants `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes` — the minimum for SQS triggers |
| `AmazonSNSFullAccess` | Grants `sns:Publish` so Lambda can post notifications |

> **Least-privilege note:** `AmazonSNSFullAccess` is used here for simplicity. In production, create a custom policy scoped to only `sns:Publish` on your specific topic ARN.

Click **Next**.

---

## 1.4 Name the Role

| Field | Value |
|-------|-------|
| Role name | `OrderProcessorLambdaRole` |
| Description | `Execution role for OrderProcessor Lambda — SQS trigger + SNS publish` |

Click **Create role**.

---

## 1.5 Note the Role ARN

1. Open the role you just created.
2. Copy the **ARN** (looks like `arn:aws:iam::123456789012:role/OrderProcessorLambdaRole`).

You will paste this ARN when creating the Lambda function in Step 4.

---

## CloudWatch Logs

`AWSLambdaSQSQueueExecutionRole` already includes the `logs:CreateLogGroup`, `logs:CreateLogStream`, and `logs:PutLogEvents` permissions that Lambda needs to write to CloudWatch. No extra policy is needed.

---
