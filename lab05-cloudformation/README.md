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

<img width="1168" height="457" alt="1" src="https://github.com/user-attachments/assets/bcde2b05-4863-4366-8082-f617d3b93017" />


---

### Step 2 — Create Workspace

```bash
mkdir -p ~/lecafe-iac

cd ~/lecafe-iac
```

<img width="852" height="340" alt="step 1" src="https://github.com/user-attachments/assets/f74cafda-1df0-43e4-9f4e-98b60c8e1488" />


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

<img width="875" height="443" alt="step 2" src="https://github.com/user-attachments/assets/e8967fb4-1980-4d01-97dd-2b5bb5cfc8e4" />


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

<img width="1406" height="580" alt="step 4" src="https://github.com/user-attachments/assets/a80aa203-89ec-4c44-9c57-e8730b81ce63" />


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



---

## 🚀 Part 3 — Validate and Deploy the Stack

### Step 9 — Validate the Template

```bash
awslocal cloudformation validate-template \
  --template-body file://lecafe-stack.yaml
```



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



---

## 📊 Part 4 — Monitor Stack Deployment

### Step 11 — Check Stack Events

```bash
awslocal cloudformation describe-stack-events \
  --stack-name lecafe-stack \
  --output table
```

<img width="1321" height="695" alt="step 11" src="https://github.com/user-attachments/assets/17a99771-ef59-4190-922a-cf0a7bab4be4" />


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

<img width="1087" height="142" alt="step10" src="https://github.com/user-attachments/assets/bac130b2-c288-4c64-920d-5994bedcf22d" />


---

### Step 15 — Inspect the Change Set

```bash
awslocal cloudformation describe-change-set \
  --stack-name lecafe-stack \
  --change-set-name update-retention \
  --output table
```

<img width="1247" height="457" alt="step 12" src="https://github.com/user-attachments/assets/70214349-662d-4a9b-82ac-03c61fc64e10" />


---

### Step 16 — Execute the Change Set

```bash
awslocal cloudformation execute-change-set \
  --stack-name lecafe-stack \
  --change-set-name update-retention
```



---

## 🔍 Part 7 — Verify Resources

### Step 17 — List Stack Resources

```bash
awslocal cloudformation list-stack-resources \
  --stack-name lecafe-stack \
  --output table
```


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
