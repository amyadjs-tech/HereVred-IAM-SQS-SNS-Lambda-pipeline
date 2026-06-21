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

## Step 3.4 is shown in below 9 pictures

<img width="1917" height="962" alt="image" src="https://github.com/user-attachments/assets/a4f1f18e-0bac-4a9d-9b0c-604b9895e366" />
<img width="1912" height="917" alt="image" src="https://github.com/user-attachments/assets/f3f47ca3-87f9-46e9-b9de-14280bb5b02a" />
<img width="1917" height="925" alt="image" src="https://github.com/user-attachments/assets/26eb480c-f9c5-4648-a82a-53b7ae999ecb" />
<img width="1917" height="931" alt="image" src="https://github.com/user-attachments/assets/ecb96eb3-1fc0-4da5-8e92-585b7d374f60" />
<img width="1907" height="926" alt="image" src="https://github.com/user-attachments/assets/4869189a-7112-4f92-8f3d-bc0c4b2c0c5d" />
<img width="1917" height="937" alt="image" src="https://github.com/user-attachments/assets/5413e817-ed56-4810-a080-9b77fa6ea991" />
<img width="1912" height="917" alt="image" src="https://github.com/user-attachments/assets/d760af1a-e064-4918-9237-cb2e7e4956f1" />
<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/acd32639-1505-4a38-a639-3d35c045f659" />
<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/b8662e69-2b1f-4440-a29b-dc84a1c93c71" />

---

## 3.5 Subscribe Your Email to OrderNotifications

This lets you see the business notification that downstream services would receive.

1. Open `OrderNotifications` → **Create subscription**.

| Field | Value |
|-------|-------|
| Protocol | **Email** |
| Endpoint | your email address |

Click **Create subscription** and confirm from your inbox.

## Step 3.5 is shown in below 7 pictures

<img width="1915" height="932" alt="image" src="https://github.com/user-attachments/assets/d6ba1553-ed88-4ab8-97e8-7f0ddae761d3" />
<img width="1917" height="921" alt="image" src="https://github.com/user-attachments/assets/64501003-e3e8-42b4-9152-494370daaae7" />
<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/5827b5e2-ce79-47a3-b585-175823d7b472" />
<img width="1917" height="932" alt="image" src="https://github.com/user-attachments/assets/7c4a4bf6-1659-46d5-88e0-eb552031e75b" />
<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/543ddfdc-88bd-4e4f-b0a6-bf00335db86e" />
<img width="1917" height="952" alt="image" src="https://github.com/user-attachments/assets/84c1f8bf-a210-44b0-8a31-dfaf41d0b757" />
<img width="1917" height="962" alt="image" src="https://github.com/user-attachments/assets/cf529ce3-779b-4c2b-9a68-949cfda3e615" />

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

## Step 3.6 is shown in below 7 pictures

<img width="1917" height="930" alt="image" src="https://github.com/user-attachments/assets/1e131a8d-7c07-4db0-b1e1-a8696e5730fe" />
<img width="1917" height="931" alt="image" src="https://github.com/user-attachments/assets/1c4a3d29-3818-40f9-88e8-8c6955449e64" />
<img width="1917" height="930" alt="image" src="https://github.com/user-attachments/assets/88c290ab-0a29-4069-9d97-ad2194c4dab2" />
<img width="1915" height="925" alt="image" src="https://github.com/user-attachments/assets/06126fd7-376c-482a-a074-744e7e6611dd" />
<img width="1917" height="926" alt="image" src="https://github.com/user-attachments/assets/f10d5eb5-45ae-49a9-951c-6bf095e28f37" />
<img width="1917" height="930" alt="image" src="https://github.com/user-attachments/assets/dc8afbfd-e45b-491b-a1f5-094f30407ee7" />
<img width="1917" height="930" alt="image" src="https://github.com/user-attachments/assets/7c452a4d-d792-4050-890f-e7c6b46a113a" />

## ARN :- arn:aws:sqs:ap-south-1:939365917679:ProcessedOrders
---

## 3.7 Subscribe ProcessedOrders Queue to OrderNotifications

1. Back in **SNS** → **Topics** → `OrderNotifications` → **Create subscription**.

| Field | Value |
|-------|-------|
| Protocol | **Amazon SQS** |
| Endpoint | ARN of `ProcessedOrders` |

Click **Create subscription**.

## Step 3.7 is shown in below 4 pictures

<img width="1917" height="931" alt="image" src="https://github.com/user-attachments/assets/c0cc4261-c07c-4a58-b6d3-ec233f61d31f" />
<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/339953e2-6f21-4134-9131-37f4961249f9" />
<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/ed1dda1a-671a-4482-9406-52284dbb5c8c" />
<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/4d9e8231-8c1d-4dfc-85a9-249ae6ecb059" />

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

## Step 3.8 is shown in below 8 pictures

<img width="1911" height="921" alt="image" src="https://github.com/user-attachments/assets/126f587d-56f7-47cc-ab7c-432abbdb9bc3" />
<img width="1917" height="922" alt="image" src="https://github.com/user-attachments/assets/5188cf23-5f76-4df2-986f-47d3f35aecd3" />
<img width="1917" height="920" alt="image" src="https://github.com/user-attachments/assets/6a203425-7911-4a46-a35b-50026bde9978" />
<img width="1915" height="930" alt="image" src="https://github.com/user-attachments/assets/5c2caa34-a230-4678-b681-ae526a96532b" />
<img width="1917" height="930" alt="image" src="https://github.com/user-attachments/assets/d85b58b0-368e-4eb8-b7cc-ca3412f02918" />
<img width="1917" height="936" alt="image" src="https://github.com/user-attachments/assets/eebb5e93-3948-4f0f-a0f8-fd80da152d05" />
<img width="1917" height="936" alt="image" src="https://github.com/user-attachments/assets/217ee335-afef-4f81-9316-b12af809aa63" />
<img width="1917" height="917" alt="image" src="https://github.com/user-attachments/assets/4b1096fd-f6ee-4c59-a818-d5bd573f6d30" />

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
