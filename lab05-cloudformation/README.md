# Lab 05 — Infrastructure as Code: CloudFormation

## 🎯 Objective

The objective of this lab is to learn how to provision and manage AWS infrastructure using Infrastructure as Code (IaC) with CloudFormation inside LocalStack.

Instead of manually creating AWS resources one command at a time, the entire Le Café infrastructure is described in a single YAML template that can be versioned, deployed, updated, and deleted automatically.

This lab demonstrates how CloudFormation handles IAM, S3, SQS, SNS, dependencies, parameters, outputs, and stack lifecycle management.

---

## 🧠 What I learned

- What Infrastructure as Code (IaC) means in DevOps
- Difference between declarative and imperative infrastructure
- Structure of a CloudFormation template
- How to use Parameters, Resources, and Outputs
- How CloudFormation automatically handles dependencies
- How to deploy, update, validate, and delete stacks
- How to create IAM, S3, SQS, and SNS resources using YAML
- How to use intrinsic functions like `!Ref`, `!GetAtt`, and `!Sub`
- How to monitor CloudFormation stack events
- How to create and execute CloudFormation change sets

---

## 🛠️ Tools used

- LocalStack
- AWS CLI Local (`awslocal`)
- CloudFormation
- YAML
- S3
- SQS
- SNS
- IAM

---

## ⚙️ Part 1 — Start LocalStack

### Step 1 — Start Services

```bash
localstack start -d

localstack status services

export AWS_PROFILE=localstack
```

<img width="1365" height="418" alt="1" src="https://github.com/user-attachments/assets/11111111-aaaa-bbbb-cccc-111111111111" />

---

### Step 2 — Create Workspace

```bash
mkdir -p ~/lecafe-iac

cd ~/lecafe-iac
```

<img width="982" height="196" alt="2" src="https://github.com/user-attachments/assets/22222222-aaaa-bbbb-cccc-222222222222" />

---

## 📄 Part 2 — Create the CloudFormation Template

### Step 3 — Create Template Header

```bash
cat > lecafe-stack.yaml << 'YAML'
AWSTemplateFormatVersion: "2010-09-09"

Description: >
  Le Café complete infrastructure stack.
  Defines IAM roles, S3 buckets, SQS queues, and SNS topics
  for the Le Café ordering and notification platform.

Parameters:

  EnvironmentName:
    Type: String
    Default: development
    AllowedValues:
      - development
      - staging
      - production

  LogRetentionDays:
    Type: Number
    Default: 30

  OrderQueueVisibilityTimeout:
    Type: Number
    Default: 30

YAML
```

<img width="1402" height="329" alt="3" src="https://github.com/user-attachments/assets/33333333-aaaa-bbbb-cccc-333333333333" />

---

### Step 4 — Add IAM Resources

```bash
cat >> lecafe-stack.yaml << 'YAML'

Resources:

  LeCafeAppRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "lecafe-app-role-${EnvironmentName}"

      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole

      Policies:
        - PolicyName: LeCafeAppPermissions
          PolicyDocument:
            Version: "2012-10-17"
            Statement:

              - Sid: AllowS3AssetRead
                Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:ListBucket
                Resource:
                  - !GetAtt LeCafeAssetsBucket.Arn
                  - !Sub "${LeCafeAssetsBucket.Arn}/*"

              - Sid: AllowSQSOrderWrite
                Effect: Allow
                Action:
                  - sqs:SendMessage
                Resource: !GetAtt LeCafeKitchenOrders.Arn

  LeCafeAppInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      InstanceProfileName: !Sub "lecafe-app-profile-${EnvironmentName}"
      Roles:
        - !Ref LeCafeAppRole

YAML
```

<img width="1447" height="557" alt="4" src="https://github.com/user-attachments/assets/44444444-aaaa-bbbb-cccc-444444444444" />

---

### Step 5 — Add S3 Resources

```bash
cat >> lecafe-stack.yaml << 'YAML'

  LeCafeAssetsBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    Properties:
      BucketName: !Sub "lecafe-assets-${EnvironmentName}"

      VersioningConfiguration:
        Status: Enabled

      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256

  LeCafeLogsBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    Properties:
      BucketName: !Sub "lecafe-logs-${EnvironmentName}"

      LifecycleConfiguration:
        Rules:
          - Id: DeleteOldLogs
            Status: Enabled
            Prefix: ""
            ExpirationInDays: !Ref LogRetentionDays

YAML
```

<img width="1427" height="505" alt="5" src="https://github.com/user-attachments/assets/55555555-aaaa-bbbb-cccc-555555555555" />

---

### Step 6 — Add SQS Resources

```bash
cat >> lecafe-stack.yaml << 'YAML'

  LeCafeKitchenOrdersDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "lecafe-kitchen-orders-dlq-${EnvironmentName}"

  LeCafeKitchenOrders:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "lecafe-kitchen-orders-${EnvironmentName}"

      VisibilityTimeout: !Ref OrderQueueVisibilityTimeout

      RedrivePolicy:
        deadLetterTargetArn: !GetAtt LeCafeKitchenOrdersDLQ.Arn
        maxReceiveCount: 3

  LeCafeInventoryUpdates:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "lecafe-inventory-updates-${EnvironmentName}"

  LeCafeLoyaltyPoints:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "lecafe-loyalty-points-${EnvironmentName}"

  LeCafeManagerAlerts:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "lecafe-manager-alerts-${EnvironmentName}"

YAML
```

<img width="1443" height="693" alt="6" src="https://github.com/user-attachments/assets/66666666-aaaa-bbbb-cccc-666666666666" />

---

### Step 7 — Add SNS Resources

```bash
cat >> lecafe-stack.yaml << 'YAML'

  LeCafeOrdersTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub "lecafe-orders-topic-${EnvironmentName}"

  InventorySubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref LeCafeOrdersTopic
      Protocol: sqs
      Endpoint: !GetAtt LeCafeInventoryUpdates.Arn

  LoyaltySubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref LeCafeOrdersTopic
      Protocol: sqs
      Endpoint: !GetAtt LeCafeLoyaltyPoints.Arn

  ManagerAlertSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref LeCafeOrdersTopic
      Protocol: sqs
      Endpoint: !GetAtt LeCafeManagerAlerts.Arn

      FilterPolicy: '{"Priority": ["high"]}'

YAML
```

<img width="1430" height="640" alt="7" src="https://github.com/user-attachments/assets/77777777-aaaa-bbbb-cccc-777777777777" />

---

### Step 8 — Add Outputs Section

```bash
cat >> lecafe-stack.yaml << 'YAML'

Outputs:

  AssetsBucketName:
    Description: Assets Bucket
    Value: !Ref LeCafeAssetsBucket

  KitchenQueueUrl:
    Description: Kitchen Queue URL
    Value: !Ref LeCafeKitchenOrders

  OrdersTopicArn:
    Description: SNS Topic ARN
    Value: !Ref LeCafeOrdersTopic

  AppRoleArn:
    Description: IAM Role ARN
    Value: !GetAtt LeCafeAppRole.Arn

YAML
```

<img width="1408" height="462" alt="8" src="https://github.com/user-attachments/assets/88888888-aaaa-bbbb-cccc-888888888888" />

---

## 🚀 Part 3 — Validate and Deploy the Stack

### Step 9 — Validate the Template

```bash
awslocal cloudformation validate-template \
  --template-body file://lecafe-stack.yaml
```

<img width="1444" height="303" alt="9" src="https://github.com/user-attachments/assets/99999999-aaaa-bbbb-cccc-999999999999" />

---

### Step 10 — Deploy the Stack

```bash
awslocal cloudformation create-stack \
  --stack-name lecafe-stack \
  --template-body file://lecafe-stack.yaml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=development \
    ParameterKey=LogRetentionDays,ParameterValue=30 \
    ParameterKey=OrderQueueVisibilityTimeout,ParameterValue=30 \
  --capabilities CAPABILITY_NAMED_IAM
```

<img width="1436" height="357" alt="10" src="https://github.com/user-attachments/assets/aaaaaaaa-1111-bbbb-cccc-aaaaaaaaaaaa" />

---

## 📊 Part 4 — Monitor Stack Deployment

### Step 11 — Check Stack Events

```bash
awslocal cloudformation describe-stack-events \
  --stack-name lecafe-stack \
  --output table
```

<img width="1451" height="764" alt="11" src="https://github.com/user-attachments/assets/bbbbbbbb-1111-bbbb-cccc-bbbbbbbbbbbb" />

---

### Step 12 — Check Stack Status

```bash
awslocal cloudformation describe-stacks \
  --stack-name lecafe-stack \
  --query 'Stacks[0].StackStatus' \
  --output text
```

Expected:

```bash
CREATE_COMPLETE
```

<img width="918" height="138" alt="12" src="https://github.com/user-attachments/assets/cccccccc-1111-bbbb-cccc-cccccccccccc" />

---

## 🔍 Part 5 — Inspect Outputs

### Step 13 — Retrieve Stack Outputs

```bash
awslocal cloudformation describe-stacks \
  --stack-name lecafe-stack \
  --query 'Stacks[0].Outputs[*].{Key:OutputKey,Value:OutputValue}' \
  --output table
```

<img width="1441" height="397" alt="13" src="https://github.com/user-attachments/assets/dddddddd-1111-bbbb-cccc-dddddddddddd" />

---

## 🔄 Part 6 — Update the Stack

### Step 14 — Create a Change Set

```bash
awslocal cloudformation create-change-set \
  --stack-name lecafe-stack \
  --change-set-name update-retention \
  --template-body file://lecafe-stack.yaml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=development \
    ParameterKey=LogRetentionDays,ParameterValue=14 \
    ParameterKey=OrderQueueVisibilityTimeout,ParameterValue=30 \
  --capabilities CAPABILITY_NAMED_IAM
```

<img width="1437" height="354" alt="14" src="https://github.com/user-attachments/assets/eeeeeeee-1111-bbbb-cccc-eeeeeeeeeeee" />

---

### Step 15 — Inspect the Change Set

```bash
awslocal cloudformation describe-change-set \
  --stack-name lecafe-stack \
  --change-set-name update-retention \
  --output table
```

<img width="1450" height="378" alt="15" src="https://github.com/user-attachments/assets/ffffffff-1111-bbbb-cccc-ffffffffffff" />

---

### Step 16 — Execute the Change Set

```bash
awslocal cloudformation execute-change-set \
  --stack-name lecafe-stack \
  --change-set-name update-retention
```

<img width="1410" height="222" alt="16" src="https://github.com/user-attachments/assets/12121212-aaaa-bbbb-cccc-121212121212" />

---

## 🔍 Part 7 — Verify Resources

### Step 17 — List Stack Resources

```bash
awslocal cloudformation list-stack-resources \
  --stack-name lecafe-stack \
  --output table
```

<img width="1448" height="772" alt="17" src="https://github.com/user-attachments/assets/13131313-aaaa-bbbb-cccc-131313131313" />

---

### Step 18 — Verify SNS Subscriptions

```bash
TOPIC_ARN=$(awslocal cloudformation describe-stacks \
  --stack-name lecafe-stack \
  --query "Stacks[0].Outputs[?OutputKey=='OrdersTopicArn'].OutputValue" \
  --output text)

awslocal sns list-subscriptions-by-topic \
  --topic-arn $TOPIC_ARN \
  --output table
```

<img width="1455" height="434" alt="18" src="https://github.com/user-attachments/assets/14141414-aaaa-bbbb-cccc-141414141414" />

---

## 🧪 Part 8 — End-to-End Test

### Step 19 — Publish a High-Priority Order

```bash
awslocal sns publish \
  --topic-arn $TOPIC_ARN \
  --message '{"orderId":"ORD-200","table":5,"items":[{"product":"Cold Brew","quantity":8,"price":4.00}],"totalAmount":32.00}' \
  --message-attributes '{"Priority":{"DataType":"String","StringValue":"high"}}' \
  --subject "New Le Cafe Order"
```

<img width="1416" height="260" alt="19" src="https://github.com/user-attachments/assets/15151515-aaaa-bbbb-cccc-151515151515" />

---

### Step 20 — Verify Queue Messages

```bash
for QUEUE_NAME in \
  "lecafe-inventory-updates-development" \
  "lecafe-loyalty-points-development" \
  "lecafe-manager-alerts-development"; do

    COUNT=$(awslocal sqs get-queue-attributes \
      --queue-url "http://localhost:4566/000000000000/${QUEUE_NAME}" \
      --attribute-names ApproximateNumberOfMessages \
      --query 'Attributes.ApproximateNumberOfMessages' \
      --output text)

    echo "  ${QUEUE_NAME}: ${COUNT} message(s)"
done
```

<img width="1450" height="265" alt="20" src="https://github.com/user-attachments/assets/16161616-aaaa-bbbb-cccc-161616161616" />

---

## 🧹 Cleanup

### Delete the Stack

```bash
awslocal cloudformation delete-stack \
  --stack-name lecafe-stack
```

### Verify Deletion

```bash
awslocal cloudformation describe-stacks \
  --stack-name lecafe-stack
```

<img width="1401" height="217" alt="21" src="https://github.com/user-attachments/assets/17171717-aaaa-bbbb-cccc-171717171717" />

---

## ✅ Conclusion

In this lab, the complete Le Café infrastructure was transformed into Infrastructure as Code using CloudFormation.

The stack successfully automated:

- IAM role creation
- S3 bucket provisioning
- SQS queue creation with DLQ support
- SNS topic subscriptions
- Queue policies
- Outputs and parameters
- Stack deployment and updates

This approach makes infrastructure:

- Reproducible
- Version-controlled
- Automated
- Easier to maintain
- Easier to scale
- Safer to deploy in teams

CloudFormation demonstrated how modern DevOps teams manage cloud infrastructure consistently across development, staging, and production environments.
