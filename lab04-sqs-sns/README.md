# Lab 04 — SQS & SNS: Messaging and Event-Driven Architecture

## 🎯 Objective

The objective of this lab is to explore AWS messaging services using LocalStack by creating message queues with Amazon SQS and implementing a publish/subscribe architecture using Amazon SNS.

This lab demonstrates how distributed systems communicate asynchronously and how event-driven architectures improve scalability, reliability, and loose coupling between services.

---

## 🧠 What I learned

- How distributed systems use asynchronous messaging
- Difference between SQS Standard Queues and FIFO Queues
- How visibility timeout and dead-letter queues work
- How to send, receive, and delete messages in SQS
- How SNS topics implement the publish/subscribe model
- How to connect SNS topics to SQS queues
- How SNS fan-out architecture distributes events to multiple systems
- How subscription filters work in SNS

---

## 🛠️ Tools used

- LocalStack
- AWS CLI Local (`awslocal`)
- Amazon SQS
- Amazon SNS
- Docker

---

# ⚙️ Part 1 — Start LocalStack

## Step 1 — Start LocalStack and Configure Environment

```bash
localstack start -d
localstack status services

export AWS_PROFILE=localstack
```

<img width="1400" height="500" alt="1" src="ADD_SCREENSHOT_HERE" />

---

# 📨 Part 2 — Amazon SQS

## Step 2 — Create Standard Queue

```bash
awslocal sqs create-queue \
  --queue-name lecafe-kitchen-orders \
  --attributes '{
    "VisibilityTimeout": "30",
    "MessageRetentionPeriod": "86400",
    "ReceiveMessageWaitTimeSeconds": "20"
  }'
```

Retrieve Queue URL:

```bash
KITCHEN_QUEUE_URL=$(awslocal sqs get-queue-url \
  --queue-name lecafe-kitchen-orders \
  --query 'QueueUrl' \
  --output text)

echo $KITCHEN_QUEUE_URL
```

<img width="1400" height="500" alt="2" src="ADD_SCREENSHOT_HERE" />

---

## Step 3 — Create Dead Letter Queue (DLQ)

```bash
awslocal sqs create-queue \
  --queue-name lecafe-kitchen-orders-dlq
```

Retrieve DLQ URL:

```bash
DLQ_URL=$(awslocal sqs get-queue-url \
  --queue-name lecafe-kitchen-orders-dlq \
  --query 'QueueUrl' \
  --output text)
```

Retrieve DLQ ARN:

```bash
DLQ_ARN=$(awslocal sqs get-queue-attributes \
  --queue-url $DLQ_URL \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text)

echo $DLQ_ARN
```

Attach DLQ to Main Queue:

```bash
awslocal sqs set-queue-attributes \
  --queue-url $KITCHEN_QUEUE_URL \
  --attributes "{
    \"RedrivePolicy\": \"{\\\"deadLetterTargetArn\\\":\\\"$DLQ_ARN\\\",\\\"maxReceiveCount\\\":\\\"3\\\"}\"
  }"
```

<img width="1400" height="500" alt="3" src="ADD_SCREENSHOT_HERE" />

---

## Step 4 — Send Messages to the Queue

### First Order

```bash
awslocal sqs send-message \
  --queue-url $KITCHEN_QUEUE_URL \
  --message-body '{
    "orderId": "ORD-001",
    "table": 4,
    "items": [
      {"product": "Flat White", "quantity": 1, "price": 3.50},
      {"product": "Croissant",  "quantity": 2, "price": 2.00}
    ],
    "totalAmount": 7.50,
    "timestamp": "2026-03-30T09:15:00Z"
  }' \
  --message-attributes '{
    "OrderType": {
      "DataType": "String",
      "StringValue": "dine-in"
    },
    "Priority": {
      "DataType": "String",
      "StringValue": "normal"
    }
  }'
```

### Additional Orders

```bash
awslocal sqs send-message \
  --queue-url $KITCHEN_QUEUE_URL \
  --message-body '{"orderId":"ORD-002","table":7,"items":[{"product":"Espresso","quantity":2,"price":2.50}],"totalAmount":5.00,"timestamp":"2026-03-30T09:16:00Z"}'

awslocal sqs send-message \
  --queue-url $KITCHEN_QUEUE_URL \
  --message-body '{"orderId":"ORD-003","table":1,"items":[{"product":"Cold Brew","quantity":4,"price":4.00},{"product":"Pain au Chocolat","quantity":4,"price":2.50}],"totalAmount":26.00,"timestamp":"2026-03-30T09:17:00Z"}'
```

Check Queue Depth:

```bash
awslocal sqs get-queue-attributes \
  --queue-url $KITCHEN_QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages
```

<img width="1400" height="500" alt="4" src="ADD_SCREENSHOT_HERE" />

---

## Step 5 — Receive and Delete Messages

Receive a Message:

```bash
MESSAGE=$(awslocal sqs receive-message \
  --queue-url $KITCHEN_QUEUE_URL \
  --max-number-of-messages 1 \
  --message-attribute-names All \
  --wait-time-seconds 5)

echo $MESSAGE | python3 -m json.tool
```

Extract Receipt Handle:

```bash
RECEIPT_HANDLE=$(echo $MESSAGE | python3 -c "
import json, sys
data = json.load(sys.stdin)
print(data['Messages'][0]['ReceiptHandle'])
")

echo $RECEIPT_HANDLE
```

Delete Message:

```bash
awslocal sqs delete-message \
  --queue-url $KITCHEN_QUEUE_URL \
  --receipt-handle "$RECEIPT_HANDLE"
```

<img width="1400" height="500" alt="5" src="ADD_SCREENSHOT_HERE" />

---

## Step 6 — Create FIFO Queue

```bash
awslocal sqs create-queue \
  --queue-name lecafe-payments.fifo \
  --attributes '{
    "FifoQueue": "true",
    "ContentBasedDeduplication": "true",
    "VisibilityTimeout": "60"
  }'
```

Retrieve FIFO Queue URL:

```bash
PAYMENTS_QUEUE_URL=$(awslocal sqs get-queue-url \
  --queue-name lecafe-payments.fifo \
  --query 'QueueUrl' \
  --output text)

echo $PAYMENTS_QUEUE_URL
```

Send Payment Message:

```bash
awslocal sqs send-message \
  --queue-url $PAYMENTS_QUEUE_URL \
  --message-body '{
    "paymentId": "PAY-001",
    "orderId":   "ORD-003",
    "amount":    26.00,
    "currency":  "EUR",
    "method":    "card",
    "status":    "authorised"
  }' \
  --message-group-id "table-1"
```

<img width="1400" height="500" alt="6" src="ADD_SCREENSHOT_HERE" />

---

# 📣 Part 3 — Amazon SNS

## Step 7 — Create SNS Topic

```bash
TOPIC_ARN=$(awslocal sns create-topic \
  --name lecafe-orders-topic \
  --query 'TopicArn' \
  --output text)

echo $TOPIC_ARN
```

<img width="1400" height="500" alt="7" src="ADD_SCREENSHOT_HERE" />

---

## Step 8 — Create Downstream Queues

```bash
awslocal sqs create-queue --queue-name lecafe-inventory-updates
awslocal sqs create-queue --queue-name lecafe-loyalty-points
awslocal sqs create-queue --queue-name lecafe-manager-alerts
```

Retrieve Queue URLs:

```bash
INVENTORY_QUEUE_URL=$(awslocal sqs get-queue-url --queue-name lecafe-inventory-updates --query 'QueueUrl' --output text)

LOYALTY_QUEUE_URL=$(awslocal sqs get-queue-url --queue-name lecafe-loyalty-points --query 'QueueUrl' --output text)

MANAGER_QUEUE_URL=$(awslocal sqs get-queue-url --queue-name lecafe-manager-alerts --query 'QueueUrl' --output text)
```

Retrieve Queue ARNs:

```bash
INVENTORY_ARN=$(awslocal sqs get-queue-attributes \
  --queue-url $INVENTORY_QUEUE_URL \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text)

LOYALTY_ARN=$(awslocal sqs get-queue-attributes \
  --queue-url $LOYALTY_QUEUE_URL \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text)

MANAGER_ARN=$(awslocal sqs get-queue-attributes \
  --queue-url $MANAGER_QUEUE_URL \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text)
```

<img width="1400" height="500" alt="8" src="ADD_SCREENSHOT_HERE" />

---

## Step 9 — Grant SNS Permission to Send Messages

```bash
apply_sns_policy() {
  local QUEUE_URL=$1
  local QUEUE_ARN=$2

  awslocal sqs set-queue-attributes \
    --queue-url "$QUEUE_URL" \
    --attributes "{
      \"Policy\": \"{
        \\\"Version\\\": \\\"2012-10-17\\\",
        \\\"Statement\\\": [{
          \\\"Sid\\\": \\\"AllowSNSPublish\\\",
          \\\"Effect\\\": \\\"Allow\\\",
          \\\"Principal\\\": {\\\"Service\\\": \\\"sns.amazonaws.com\\\"},
          \\\"Action\\\": \\\"sqs:SendMessage\\\",
          \\\"Resource\\\": \\\"$QUEUE_ARN\\\",
          \\\"Condition\\\": {
            \\\"ArnEquals\\\": {
              \\\"aws:SourceArn\\\": \\\"$TOPIC_ARN\\\"
            }
          }
        }]
      }\"
    }"
}

apply_sns_policy "$INVENTORY_QUEUE_URL" "$INVENTORY_ARN"
apply_sns_policy "$LOYALTY_QUEUE_URL" "$LOYALTY_ARN"
apply_sns_policy "$MANAGER_QUEUE_URL" "$MANAGER_ARN"
```

<img width="1400" height="500" alt="9" src="ADD_SCREENSHOT_HERE" />

---

## Step 10 — Subscribe Queues to SNS Topic

```bash
awslocal sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint $INVENTORY_ARN

awslocal sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint $LOYALTY_ARN

awslocal sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint $MANAGER_ARN
```

Verify Subscriptions:

```bash
awslocal sns list-subscriptions-by-topic \
  --topic-arn $TOPIC_ARN
```

<img width="1400" height="500" alt="10" src="ADD_SCREENSHOT_HERE" />

---

## Step 11 — Add Subscription Filter

Retrieve Subscription ARN:

```bash
MANAGER_SUB_ARN=$(awslocal sns list-subscriptions-by-topic \
  --topic-arn $TOPIC_ARN \
  --query "Subscriptions[?Endpoint=='$MANAGER_ARN'].SubscriptionArn" \
  --output text)

echo $MANAGER_SUB_ARN
```

Apply Filter Policy:

```bash
awslocal sns set-subscription-attributes \
  --subscription-arn $MANAGER_SUB_ARN \
  --attribute-name FilterPolicy \
  --attribute-value '{"Priority": ["high"]}'
```

<img width="1400" height="500" alt="11" src="ADD_SCREENSHOT_HERE" />

---

# 🔗 Part 4 — Test the Fan-Out Architecture

## Step 12 — Publish High Priority Order

```bash
awslocal sns publish \
  --topic-arn $TOPIC_ARN \
  --message '{
    "orderId":     "ORD-100",
    "table":       12,
    "items": [
      {"product": "Cold Brew", "quantity": 6, "price": 4.00},
      {"product": "Pain au Chocolat", "quantity": 6, "price": 2.50}
    ],
    "totalAmount": 39.00,
    "timestamp":   "2026-03-30T10:00:00Z"
  }' \
  --message-attributes '{
    "Priority": {"DataType": "String", "StringValue": "high"},
    "OrderType": {"DataType": "String", "StringValue": "dine-in"}
  }' \
  --subject "New Le Cafe Order"
```

Publish Normal Priority Order:

```bash
awslocal sns publish \
  --topic-arn $TOPIC_ARN \
  --message '{
    "orderId":     "ORD-101",
    "table":       3,
    "items": [
      {"product": "Espresso", "quantity": 1, "price": 2.50}
    ],
    "totalAmount": 2.50,
    "timestamp":   "2026-03-30T10:01:00Z"
  }' \
  --message-attributes '{
    "Priority": {"DataType": "String", "StringValue": "normal"},
    "OrderType": {"DataType": "String", "StringValue": "dine-in"}
  }' \
  --subject "New Le Cafe Order"
```

<img width="1400" height="500" alt="12" src="ADD_SCREENSHOT_HERE" />

---

## Step 13 — Verify Fan-Out Results

Inventory Queue:

```bash
awslocal sqs get-queue-attributes \
  --queue-url $INVENTORY_QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages
```

Loyalty Queue:

```bash
awslocal sqs get-queue-attributes \
  --queue-url $LOYALTY_QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages
```

Manager Queue:

```bash
awslocal sqs get-queue-attributes \
  --queue-url $MANAGER_QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages
```

<img width="1400" height="500" alt="13" src="ADD_SCREENSHOT_HERE" />

---

## Step 14 — Read Message from Manager Queue

```bash
awslocal sqs receive-message \
  --queue-url $MANAGER_QUEUE_URL \
  --max-number-of-messages 1 | python3 -m json.tool
```

<img width="1400" height="500" alt="14" src="ADD_SCREENSHOT_HERE" />

---

# 🔍 Key Concepts Learned

## SQS Standard Queue

- High throughput
- At least once delivery
- Best-effort ordering
- Used for scalable workloads

## SQS FIFO Queue

- Exactly-once processing
- Strict message ordering
- Requires `.fifo` suffix
- Requires `MessageGroupId`

## Dead Letter Queue (DLQ)

- Stores failed messages
- Prevents poison messages from blocking processing
- Helps debugging and monitoring

## SNS Fan-Out Architecture

- One message published to SNS
- Multiple subscribers receive copies
- Systems remain loosely coupled
- Easy to scale independently

---

# 🧹 Cleanup

Delete Queues:

```bash
awslocal sqs delete-queue --queue-url $KITCHEN_QUEUE_URL
awslocal sqs delete-queue --queue-url $DLQ_URL
awslocal sqs delete-queue --queue-url $PAYMENTS_QUEUE_URL
awslocal sqs delete-queue --queue-url $INVENTORY_QUEUE_URL
awslocal sqs delete-queue --queue-url $LOYALTY_QUEUE_URL
awslocal sqs delete-queue --queue-url $MANAGER_QUEUE_URL
```

Delete SNS Topic:

```bash
awslocal sns delete-topic --topic-arn $TOPIC_ARN
```

Stop LocalStack:

```bash
localstack stop
docker ps --filter name=localstack
```

<img width="1400" height="500" alt="15" src="ADD_SCREENSHOT_HERE" />

---
