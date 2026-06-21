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

# Step 3 — SNS: Create Topics and Subscribe

You will create two SNS topics:

| Topic | Purpose |
|-------|---------|
| `OrderNotifications` | **Business events** — published on successful processing; consumed by fulfillment, inventory, etc. |
| `OrderAlerts` | **Operational alerts** — published on every success AND failure; consumed by operators and monitoring tools |

Separating concerns means a failed message alert doesn't land in the same topic as a downstream fulfillment event.

---

## 3.1 Create OrderNotifications Topic

1. In the AWS Console, search for **SNS** and open it.
2. In the left sidebar click **Topics** → **Create topic**.

| Field | Value |
|-------|-------|
| Type | **Standard** |
| Name | `OrderNotifications` |

Click **Create topic**.

3. Copy the **Topic ARN** — you will set this as `SNS_TOPIC_ARN` in the Lambda environment variables.

## Step 3.1 is shown in below 7 pictures

<img width="1917" height="932" alt="image" src="https://github.com/user-attachments/assets/2d1e9347-0093-4fa7-b9bc-6dc0b58c2469" />
<img width="1912" height="922" alt="image" src="https://github.com/user-attachments/assets/6db40444-3514-45d0-9895-281bba6825d6" />
<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/90bf62ba-9b6d-4b34-9b16-253281e7ae86" />
<img width="1917" height="935" alt="image" src="https://github.com/user-attachments/assets/59895a3a-5bcb-4e46-b0e2-151d3b1f92fb" />
<img width="1917" height="916" alt="image" src="https://github.com/user-attachments/assets/3ae17e71-9560-4bc6-b2a9-5f959933c333" />
<img width="1917" height="930" alt="image" src="https://github.com/user-attachments/assets/b83f551e-81df-43eb-be05-a520ac7b029e" />
<img width="1917" height="922" alt="image" src="https://github.com/user-attachments/assets/e3613cd7-2c2b-414a-bc1b-ed75595dfcfa" />

## ARN :- arn:aws:sns:ap-south-1:939365917679:OrderNotifications

---

## 3.2 Create OrderAlerts Topic

1. Click **Create topic** again.

| Field | Value |
|-------|-------|
| Type | **Standard** |
| Name | `OrderAlerts` |

Click **Create topic**.

2. Copy the **Topic ARN** — you will set this as `ALERT_SNS_TOPIC_ARN` in the Lambda environment variables.

## Step 3.3 is shown in below 5 pictures

<img width="1917" height="917" alt="image" src="https://github.com/user-attachments/assets/aa64d2e7-a9de-42c8-90d5-6f4bf7eb9463" />
<img width="1917" height="930" alt="image" src="https://github.com/user-attachments/assets/69581b66-aab3-4687-b173-bfbdad70bc99" />
<img width="1912" height="922" alt="image" src="https://github.com/user-attachments/assets/5d614e94-f56e-46ca-90d4-6e570c9d5fc8" />
<img width="1917" height="916" alt="image" src="https://github.com/user-attachments/assets/6f383b44-c3cd-4c14-8988-59c7b5d341c0" />
<img width="1917" height="930" alt="image" src="https://github.com/user-attachments/assets/307cb1ee-26ec-4e3a-8328-ec88ca015866" />

## ARN :- arn:aws:sns:ap-south-1:939365917679:OrderAlerts

---

## 3.3 Subscribe Your Email to OrderAlerts

This gives you real-time visibility into every success and failure the Lambda processes.

1. With `OrderAlerts` open, click **Create subscription**.

| Field | Value |
|-------|-------|
| Protocol | **Email** |
| Endpoint | your email address |

Click **Create subscription**.

## Step 3.3 is shown in below 3 pictures

<img width="1917" height="935" alt="image" src="https://github.com/user-attachments/assets/72c07d3a-f2b5-49cb-9565-e44dfd8dea82" />
<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/891e6983-f25e-45ef-8a2c-293a1d615ad3" />


2. Check your inbox for a confirmation email from AWS and click **Confirm subscription**.

<img width="1901" height="962" alt="image" src="https://github.com/user-attachments/assets/ab49ac93-fe92-463a-86b4-3a3e45c2a6a3" />
<img width="1917" height="952" alt="image" src="https://github.com/user-attachments/assets/f55a9b27-46d7-4be5-801e-e05af72e107c" />


> You will not receive any alerts until you confirm. Check your spam folder if it does not arrive within a minute.


---

## 3.4 (Optional) Filter Alerts by Status

If you only want to receive failure emails — not one for every successful order — add a subscription filter policy:

1. SNS → `OrderAlerts` → **Subscriptions** → click your email subscription → **Edit**.
2. Under **Subscription filter policy** paste:

```json
{
  "status": ["FAILED"]
}
```

Click **Save changes**. Now your email only receives `FAILED` alerts; `SUCCESS` alerts are still published but filtered out for this subscriber.

> The `status` attribute is set by the Lambda function in `MessageAttributes` — `SUCCESS` or `FAILED` — so SNS can route them differently per subscriber.

## Step 3.4 is shown in below 3 pictures

<img width="1917" height="962" alt="image" src="https://github.com/user-attachments/assets/a4f1f18e-0bac-4a9d-9b0c-604b9895e366" />
<img width="1912" height="917" alt="image" src="https://github.com/user-attachments/assets/f3f47ca3-87f9-46e9-b9de-14280bb5b02a" />
<img width="1917" height="925" alt="image" src="https://github.com/user-attachments/assets/26eb480c-f9c5-4648-a82a-53b7ae999ecb" />




---

## 3.5 Subscribe Your Email to OrderNotifications

This lets you see the business notification that downstream services would receive.

1. Open `OrderNotifications` → **Create subscription**.

| Field | Value |
|-------|-------|
| Protocol | **Email** |
| Endpoint | your email address |

Click **Create subscription** and confirm from your inbox.

---

## 3.6 Create a "ProcessedOrders" SQS Queue

This queue simulates a downstream service (e.g., a fulfillment system) consuming processed-order events.

1. Go to **SQS** → **Create queue**.

| Field | Value |
|-------|-------|
| Type | Standard |
| Name | `ProcessedOrders` |

Click **Create queue**.

2. Copy the **Queue ARN** for `ProcessedOrders`.

---

## 3.7 Subscribe ProcessedOrders Queue to OrderNotifications

1. Back in **SNS** → **Topics** → `OrderNotifications` → **Create subscription**.

| Field | Value |
|-------|-------|
| Protocol | **Amazon SQS** |
| Endpoint | ARN of `ProcessedOrders` |

Click **Create subscription**.

---

## 3.8 Add SQS Access Policy for SNS Delivery

SNS must be allowed to send messages to the `ProcessedOrders` queue.

1. Go to **SQS** → `ProcessedOrders` → **Access policy** tab → **Edit**.
2. Replace the existing policy with the following (substitute your values):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSNSPublish",
      "Effect": "Allow",
      "Principal": {
        "Service": "sns.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "<ProcessedOrders-Queue-ARN>",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "<OrderNotifications-Topic-ARN>"
        }
      }
    }
  ]
}
```

Click **Save**.

---

## What You Created

```
OrderNotifications (SNS Topic)  ← business events (success only)
  ├── Email subscription  → your inbox
  └── SQS subscription   → ProcessedOrders queue

OrderAlerts (SNS Topic)          ← ops visibility (success + failure)
  └── Email subscription  → your inbox
        (optional filter: status = FAILED)
```

---
