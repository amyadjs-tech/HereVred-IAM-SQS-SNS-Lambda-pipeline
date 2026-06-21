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

# Step 2 — SQS: Create OrderQueue and Dead-Letter Queue

You will create two queues:
- **OrderDLQ** — receives messages that Lambda fails to process after several attempts
- **OrderQueue** — the main inbound queue; Lambda is triggered by messages here

Create the DLQ first because the main queue references it.

---

## 2.1 Create the Dead-Letter Queue (OrderDLQ)

1. In the AWS Console, search for **SQS** and open it.
2. Click **Create queue**.

| Field | Value |
|-------|-------|
| Type | Standard |
| Name | `OrderDLQ` |

Leave all other settings at their defaults and click **Create queue**.

3. Copy the **Queue ARN** for `OrderDLQ` — you will need it in the next section.

## Step 2.1 is shown in below 7 pictures

<img width="1917" height="932" alt="image" src="https://github.com/user-attachments/assets/1c24eca8-034a-4740-bb1d-2113e47f36b3" />
<img width="1917" height="935" alt="image" src="https://github.com/user-attachments/assets/ad074938-8968-4878-8c22-1a4f7f07d47e" />
<img width="1917" height="936" alt="image" src="https://github.com/user-attachments/assets/1b107b15-64bc-4ae0-a7dd-eb3d786f17e4" />
<img width="1917" height="936" alt="image" src="https://github.com/user-attachments/assets/294fe683-f355-412a-981b-8971be3ebacd" />
<img width="1917" height="932" alt="image" src="https://github.com/user-attachments/assets/1fbed925-296c-4c46-9633-d0c7e88f3c20" />
<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/b057b9b5-c0b4-4942-9099-12c1173cef9a" />
<img width="1917" height="930" alt="image" src="https://github.com/user-attachments/assets/3c770b51-ba39-4df0-a7cc-1005b883fc9d" />

## ARN : arn:aws:sqs:ap-south-1:939365917679:OrderDLQ
---

## 2.2 Create the Main Queue (OrderQueue)

1. Click **Create queue** again.

| Field | Value |
|-------|-------|
| Type | Standard |
| Name | `OrderQueue` |

2. Scroll down to **Dead-letter queue** and expand it.

| Field | Value |
|-------|-------|
| Dead-letter queue | **Enabled** |
| Choose queue | `OrderDLQ` (paste the ARN you copied) |
| Maximum receives | `3` |

> **Maximum receives = 3** means: if the same message is received and not deleted 3 times (Lambda threw an exception each time), SQS moves it to the DLQ automatically.

3. Leave all other settings at defaults. Click **Create queue**.

4. Copy the **Queue URL** and **Queue ARN** for `OrderQueue` — you will need these in later steps.

## Step 2.2 is shown in below 8 pictures

<img width="1917" height="935" alt="image" src="https://github.com/user-attachments/assets/c1f66c25-0fb8-4200-b0ec-02db8cf20f61" />
<img width="1917" height="932" alt="image" src="https://github.com/user-attachments/assets/f9c708cc-455b-45de-9f3a-7accf015f819" />
<img width="1917" height="942" alt="image" src="https://github.com/user-attachments/assets/3465979d-91d8-40be-ab59-9f2f2c324fad" />
<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/c54a9804-3157-4f8f-abec-46da1fa36a23" />
<img width="1917" height="926" alt="image" src="https://github.com/user-attachments/assets/0ef8a40f-3167-4276-99f2-238190bde937" />
<img width="1917" height="941" alt="image" src="https://github.com/user-attachments/assets/315a29db-5481-4fc2-b964-51eb4b0ff19b" />
<img width="1917" height="932" alt="image" src="https://github.com/user-attachments/assets/47598fbc-a272-45c9-bb03-0e6c30137cd7" />
<img width="1917" height="935" alt="image" src="https://github.com/user-attachments/assets/c31ddda5-4da2-4ff4-8e0a-58c96e8d50a9" />

URL : - https://sqs.ap-south-1.amazonaws.com/939365917679/OrderQueue
ARN : - arn:aws:sqs:ap-south-1:939365917679:OrderQueue

<img width="1917" height="942" alt="image" src="https://github.com/user-attachments/assets/f7165ce6-ea6c-4c80-9698-139445d31385" />

---

## What You Created

```
OrderQueue (Standard)
  └── Dead-letter queue → OrderDLQ (maxReceiveCount: 3)
```

When Lambda fails to process a message 3 times, SQS automatically moves it to `OrderDLQ` so it isn't lost and can be investigated.

---
