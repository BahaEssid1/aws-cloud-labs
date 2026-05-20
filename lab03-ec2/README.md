# Lab 03 — EC2: Compute, Key Pairs, and Security Groups

## 🎯 Objective

The objective of this lab is to provision and manage EC2 infrastructure using LocalStack while understanding how compute resources interact with IAM, S3, and SQS services inside AWS.

This lab simulates a real-world backend deployment for the Le Café ordering application.

---

## 🧠 What I learned

- What Amazon EC2 is and how virtual machines are provisioned
- The role of AMIs, instance types, key pairs, and security groups
- How to create and manage EC2 key pairs
- How to configure inbound firewall rules using security groups
- How to launch EC2 instances with IAM roles attached
- How EC2 instance lifecycle states work
- How AWS services integrate together into a complete architecture

---

## 🛠️ Tools used

- LocalStack
- AWS CLI Local (`awslocal`)
- EC2
- IAM
- S3
- SQS
- Bash scripting

---

## 🏪 Scenario — Le Café Needs a Server

Le Café’s ordering application now needs a proper backend server capable of reading assets from S3 and sending orders to SQS.

The infrastructure team decided to deploy the backend on EC2.

In this lab, the following infrastructure was created:

- IAM role for the application
- S3 bucket for assets
- SQS queue for orders
- EC2 key pair
- Security group
- EC2 instance attached to the IAM role

---

# ⚙️ Part 1 — Prepare the Environment

## Step 1 — Start LocalStack

```bash
localstack start -d
localstack status services
export AWS_PROFILE=localstack
```

<img width="1168" height="457" alt="1" src="https://github.com/user-attachments/assets/40695ee2-2074-496c-a235-25cbeff4c4e3" />


---

## Step 2 — Create IAM Trust Policy

```bash
cat > /tmp/trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF
```

---

## Step 3 — Create IAM Role

```bash
awslocal iam create-role \
  --role-name lecafe-app-role \
  --assume-role-policy-document file:///tmp/trust-policy.json 2>/dev/null || \
  echo "Role already exists — continuing."
```

<img width="948" height="558" alt="1" src="https://github.com/user-attachments/assets/aea69df7-dda5-44a8-9c14-acc7e9dfb059" />


---

## Step 4 — Create Inline IAM Policy

```bash
cat > /tmp/app-role-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3AssetRead",
      "Effect": "Allow",
      "Action": ["s3:GetObject","s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::lecafe-assets",
        "arn:aws:s3:::lecafe-assets/*"
      ]
    },
    {
      "Sid": "AllowSQSOrderWrite",
      "Effect": "Allow",
      "Action": ["sqs:SendMessage","sqs:GetQueueUrl","sqs:GetQueueAttributes"],
      "Resource": "arn:aws:sqs:us-east-1:000000000000:lecafe-orders"
    }
  ]
}
EOF
```

---

## Step 5 — Attach Policy to the Role

```bash
awslocal iam put-role-policy \
  --role-name lecafe-app-role \
  --policy-name LeCafe-App-Permissions \
  --policy-document file:///tmp/app-role-policy.json
```

```bash
echo "IAM setup complete."
```



---

# 🪣 Part 2 — Recreate S3 and SQS Resources

## Step 6 — Create the S3 Bucket

```bash
awslocal s3 mb s3://lecafe-assets 2>/dev/null || echo "Bucket exists — continuing."
```

---

## Step 7 — Upload a Configuration File

```bash
echo "Le Café App Config — v1.0" > config.txt

awslocal s3 cp config.txt s3://lecafe-assets/app/config.txt
```

<img width="1137" height="232" alt="part 1 step 2" src="https://github.com/user-attachments/assets/d1cc9401-d06d-4ff6-866c-2f1581dde995" />


---

## Step 8 — Create the SQS Queue

```bash
awslocal sqs create-queue --queue-name lecafe-orders 2>/dev/null || echo "Queue exists — continuing."
```

```bash
echo "S3 and SQS setup complete."
```



---

# 🔑 Part 3 — Create a Key Pair

## Step 9 — Generate the Key Pair

```bash
awslocal ec2 create-key-pair \
  --key-name lecafe-keypair \
  --query 'KeyMaterial' \
  --output text > lecafe-keypair.pem
```


---

## Step 10 — Secure the PEM File

```bash
chmod 400 lecafe-keypair.pem
```

---

## Step 11 — Verify the Key Pair

```bash
awslocal ec2 describe-key-pairs
```

<img width="992" height="692" alt="step 3" src="https://github.com/user-attachments/assets/8a50d085-043c-44d9-9f84-cb5b8604a734" />


---

# 🔥 Part 4 — Create a Security Group

## Step 12 — Retrieve the Default VPC

```bash
VPC_ID=$(awslocal ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)

echo "Default VPC ID: $VPC_ID"
```

---

## Step 13 — Create the Security Group

```bash
SG_ID=$(awslocal ec2 create-security-group \
  --group-name lecafe-app-sg \
  --description "Security group for the Le Cafe ordering application" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

echo "Security Group ID: $SG_ID"
```

<img width="1027" height="337" alt="step5" src="https://github.com/user-attachments/assets/1d17d232-5fb6-4f8b-968f-59cc242b0405" />


---

## Step 14 — Allow HTTP Traffic

```bash
awslocal ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

---

## Step 15 — Allow HTTPS Traffic

```bash
awslocal ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

---

## Step 16 — Allow SSH Access

```bash
awslocal ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```
<img width="971" height="823" alt="step 6" src="https://github.com/user-attachments/assets/a6d325f1-5162-43df-99dd-7896b85455c2" />

---

## Step 17 — Verify Security Group Rules

```bash
awslocal ec2 describe-security-groups --group-ids $SG_ID
```

<img width="1141" height="830" alt="step 6 (2)" src="https://github.com/user-attachments/assets/4ad834f7-2804-4ace-929e-94d643293856" />
<img width="1042" height="827" alt="step 6 (3)" src="https://github.com/user-attachments/assets/e27d30fe-66d1-4dde-9df1-7015a128568e" />


---

# 🚀 Part 5 — Launch the EC2 Instance

## Step 18 — Define the AMI

```bash
AMI_ID="ami-0c02fb55956c7d316"

echo "Using AMI: $AMI_ID"
```
<img width="812" height="97" alt="step 7" src="https://github.com/user-attachments/assets/63d46a19-576e-4e15-a9be-56b78169c09b" />

---

## Step 19 — Create the Instance Profile

```bash
awslocal iam create-instance-profile \
  --instance-profile-name lecafe-app-profile
```

```bash
awslocal iam add-role-to-instance-profile \
  --instance-profile-name lecafe-app-profile \
  --role-name lecafe-app-role
```

```bash
echo "Instance profile ready."
```

<img width="977" height="827" alt="step 8" src="https://github.com/user-attachments/assets/2c1fe804-0f61-479f-83e3-9eda51b42015" />


---

## Step 20 — Create the User Data Script

```bash
cat > /tmp/userdata.sh << 'EOF'
#!/bin/bash

yum update -y
yum install -y aws-cli

mkdir -p /opt/lecafe

cat > /opt/lecafe/config.env << 'ENVEOF'
APP_NAME=lecafe-ordering
S3_BUCKET=lecafe-assets
SQS_QUEUE=lecafe-orders
AWS_REGION=us-east-1
ENVEOF

aws s3 cp s3://lecafe-assets/app/config.txt /opt/lecafe/app-config.txt \
  --region us-east-1

echo "Le Café bootstrap complete." >> /var/log/lecafe-setup.log
EOF
```

<img width="695" height="271" alt="step 9" src="https://github.com/user-attachments/assets/567d8bb9-9390-4edc-a82f-707ed10628e8" />


---

## Step 21 — Launch the EC2 Instance

```bash
INSTANCE_ID=$(awslocal ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name lecafe-keypair \
  --security-group-ids $SG_ID \
  --iam-instance-profile Name=lecafe-app-profile \
  --user-data file:///tmp/userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lecafe-app-server},{Key=Project,Value=LeCafe},{Key=Environment,Value=development}]' \
  --count 1 \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Instance launched: $INSTANCE_ID"
```

<img width="1362" height="250" alt="step 10" src="https://github.com/user-attachments/assets/0aec411c-1d44-4804-b825-9be9527a456e" />


---

# 🔍 Part 6 — Inspect the EC2 Instance

## Step 22 — Check Instance Information

```bash
awslocal ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].{
    State:State.Name,
    PublicIP:PublicIpAddress,
    PrivateIP:PrivateIpAddress,
    ImageId:ImageId,
    InstanceType:InstanceType,
    KeyName:KeyName,
    IAMProfile:IamInstanceProfile.Arn
  }' \
  --output table
```

<img width="790" height="185" alt="step 11" src="https://github.com/user-attachments/assets/5ac00a4c-2764-40c2-8219-e8f30eb16b96" />



---

## Step 23 — List All Instances

```bash
awslocal ec2 describe-instances \
  --query 'Reservations[].Instances[].{
    ID:InstanceId,
    Name:Tags[?Key==`Name`]|[0].Value,
    State:State.Name,
    Type:InstanceType
  }' \
  --output table
```

<img width="827" height="818" alt="step 11 (2)" src="https://github.com/user-attachments/assets/978d3b2d-184b-4698-b7cc-33785f6d3ff8" />


---

# 🔄 Part 7 — EC2 Lifecycle Management

## Step 24 — Stop the Instance

```bash
awslocal ec2 stop-instances --instance-ids $INSTANCE_ID
```

---

## Step 25 — Verify the Stopped State

```bash
awslocal ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text
```

<img width="1108" height="712" alt="step 12" src="https://github.com/user-attachments/assets/b1273075-e606-4029-abe0-7c5ba1d5af8c" />


---

## Step 26 — Start the Instance Again

```bash
awslocal ec2 start-instances --instance-ids $INSTANCE_ID
```

---

## Step 27 — Verify Running State

```bash
awslocal ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text
```

<img width="770" height="827" alt="step12" src="https://github.com/user-attachments/assets/fbe306cf-5e42-420f-8335-80d864e63ae3" />


---

# 🔐 Part 8 — Simulate SSH Access

## Step 28 — Retrieve the Public IP

```bash
PUBLIC_IP=$(awslocal ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "Would connect to: $PUBLIC_IP"
```

---

## Step 29 — SSH Command Example

```bash
ssh -i lecafe-keypair.pem ec2-user@$PUBLIC_IP
```

<img width="1102" height="581" alt="step 13 (2)" src="https://github.com/user-attachments/assets/43b7c119-5baf-4e9f-88d0-63a64a3d7514" />


---

# 🔗 Part 9 — Verify Full Stack Integration

## Step 30 — Verify the Attached IAM Role

```bash
awslocal ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].IamInstanceProfile'
```

<img width="948" height="187" alt="step 14" src="https://github.com/user-attachments/assets/45d4d2f8-6304-4327-9bcd-7fc153118ef0" />


---

## Step 31 — Review the Complete Architecture

### Show IAM Role

```bash
awslocal iam get-role --role-name lecafe-app-role \
  --query 'Role.{Name:RoleName, ARN:Arn, Created:CreateDate}'
```

---

### Show S3 Bucket Contents

```bash
awslocal s3 ls s3://lecafe-assets/ --recursive
```

---

### Show SQS Queue

```bash
awslocal sqs get-queue-url --queue-name lecafe-orders
```
<img width="986" height="321" alt="step 15" src="https://github.com/user-attachments/assets/e6735b7b-2413-4a85-a6e9-81cacc0a5a1e" />

---

### Show EC2 Instance Details

```bash
awslocal ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].{
    Instance:InstanceId,
    State:State.Name,
    Role:IamInstanceProfile.Arn,
    SecurityGroup:SecurityGroups[0].GroupName,
    KeyPair:KeyName
  }' \
  --output table
```

<img width="1253" height="342" alt="step 16" src="https://github.com/user-attachments/assets/191e6f35-2d76-48d8-a586-4e102d80d7fb" />


---

# 🧹 Cleanup

## Stop LocalStack

```bash
localstack stop
```

---

## Verify Container Shutdown

```bash
docker ps --filter name=localstack
```


---

# ✅ Conclusion

In this lab, a complete EC2 infrastructure stack was deployed locally using LocalStack.

The lab demonstrated how AWS compute resources integrate with IAM, S3, and SQS services while following real-world infrastructure practices such as:

- IAM role attachment
- Security group configuration
- Key pair authentication
- User data bootstrapping
- Instance lifecycle management

This lab connected identity, storage, messaging, and compute into a single working AWS architecture.
