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

## Step 1.3 is shown in below 2 pictures

<img width="1917" height="926" alt="image" src="https://github.com/user-attachments/assets/35a8e838-430b-4759-a2a5-e835e5fa45e2" />
<img width="1917" height="930" alt="image" src="https://github.com/user-attachments/assets/f5daf987-7ff1-4700-8257-a178e273697d" />

---

## 1.4 Name the Role

| Field | Value |
|-------|-------|
| Role name | `OrderProcessorLambdaRole` |
| Description | `Execution role for OrderProcessor Lambda — SQS trigger + SNS publish` |

Click **Create role**.

## Step 1.4 is shown in below 3 pictures

<img width="1917" height="922" alt="image" src="https://github.com/user-attachments/assets/c7134cda-8224-4665-92d3-d9b1f52bcd91" />

<img width="1917" height="921" alt="image" src="https://github.com/user-attachments/assets/bec00cee-bc6a-4a47-b3f2-b87854c29b54" />

<img width="1917" height="920" alt="image" src="https://github.com/user-attachments/assets/1f53b7fa-e463-4306-a30e-0070c8f3a704" />

---

## 1.5 Note the Role ARN

1. Open the role you just created.
2. Copy the **ARN** (looks like `arn:aws:iam::123456789012:role/OrderProcessorLambdaRole`).

You will paste this ARN when creating the Lambda function in Step 4.

## Step 1.5 is shown in below 2 pictures

<img width="1917" height="955" alt="image" src="https://github.com/user-attachments/assets/2fffec8f-af9c-4b74-b84e-7ac136c8fff2" />

## Click on View role you will get the below page where you can copy the ARN

<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/e7220760-fc47-469b-8c95-068e3a0c3a0c" />

## ARN : arn:aws:iam::939365917679:role/OrderProcessorLambdaRole

---

## CloudWatch Logs

`AWSLambdaSQSQueueExecutionRole` already includes the `logs:CreateLogGroup`, `logs:CreateLogStream`, and `logs:PutLogEvents` permissions that Lambda needs to write to CloudWatch. No extra policy is needed.

---
