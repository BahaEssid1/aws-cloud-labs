# Lab 00 — LocalStack Setup

## 🎯 Objective

The objective of this lab is to set up a local AWS simulation environment using LocalStack and perform basic AWS S3 operations without using a real AWS account.

This allows developers to test and experiment with AWS services locally without incurring cloud costs.

---

## 🧠 What I learned

- How to install and run LocalStack
- How AWS services can be simulated locally
- How to use `awslocal` as a replacement for AWS CLI
- Basic Amazon S3 operations (create bucket, upload file, list objects, download file)

---

## 🛠️ Tools used

- LocalStack
- AWS CLI Local (`awslocal`)
- Docker (used internally by LocalStack)

---

## 📁 Project Files

- `menu.txt` → file uploaded to S3 bucket
- `menu-downloaded.txt` → file downloaded from S3 bucket after retrieval

---

## ⚙️ Part 1 — Installation

### 1. Install LocalStack and dependencies

LocalStack and AWS CLI local wrapper were installed using pip:



pip install localstack awscli-local


)

🚀 Part 2 — Start LocalStack

LocalStack was started in local mode:

localstack start -d

<img width="1168" height="457" alt="1" src="https://github.com/user-attachments/assets/6c8d3486-0cf5-42c8-8874-43b5eb0ff425" />

3. Check LocalStack health

Verified that LocalStack services are running:

Invoke-RestMethod http://localhost:4566/_localstack/health
localstack status services

<img width="1421" height="151" alt="4" src="https://github.com/user-attachments/assets/153f4c57-2715-4825-a257-bdf302e889a2" />

<img width="755" height="426" alt="2" src="https://github.com/user-attachments/assets/c2202544-f775-439c-9a53-00834ea31eb4" />


🔑 Part 3 — Configure AWS CLI
Step 6 — Create LocalStack Profile
aws configure --profile localstack

Use:

AWS Access Key: test  
AWS Secret Key: test  
Region: us-east-1  
Output: json

<img width="781" height="172" alt="5" src="https://github.com/user-attachments/assets/ded6bbf5-56ca-4e7c-8382-9672fecc2a7c" />

Step 7 — Set Profile
export AWS_PROFILE=localstack

<img width="1082" height="361" alt="6" src="https://github.com/user-attachments/assets/baef9fba-7351-42a1-be8c-7d323d5ce187" />


🪣 Part 4 — Create Resources
Step 8 — Create S3 Bucket
awslocal s3 mb s3://lecafe-menus
awslocal s3 ls


Upload & Retrieve File
echo "Espresso: 2.50 | Latte: 3.50 | Croissant: 2.00" > menu.txt

awslocal s3 cp menu.txt s3://lecafe-menus/menu-paris.txt
awslocal s3 ls s3://lecafe-menus/

awslocal s3 cp s3://lecafe-menus/menu-paris.txt menu-downloaded.txt
cat menu-downloaded.txt

<img width="1082" height="361" alt="6" src="https://github.com/user-attachments/assets/fa864824-332f-4e51-8339-f485a02fdee0" />


Step 9 — Create IAM User
awslocal iam create-user --user-name lecafe-app
awslocal iam list-users


Attach Policy
awslocal iam attach-user-policy \
  --user-name lecafe-app \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

awslocal iam list-attached-user-policies --user-name lecafe-app

<img width="1438" height="257" alt="7" src="https://github.com/user-attachments/assets/f9c0c76e-ca5b-4df5-9b24-c7f7a7e41e92" />


Step 10 — Create SQS Queue
awslocal sqs create-queue --queue-name lecafe-orders
awslocal sqs get-queue-url --queue-name lecafe-orders

<img width="1432" height="408" alt="8" src="https://github.com/user-attachments/assets/2cb0093b-23d3-4466-8855-c1c03c8dda52" />


Send & Receive Message
awslocal sqs send-message \
  --queue-url http://localhost:4566/000000000000/lecafe-orders \
  --message-body '{"item": "Latte", "size": "large", "table": 7}'

awslocal sqs receive-message \
  --queue-url http://localhost:4566/000000000000/lecafe-orders

<img width="1450" height="302" alt="9" src="https://github.com/user-attachments/assets/f5fc1634-7e19-404b-841c-5771109adbba" />


🔍 Part 5 — Inspection
Step 11 — Swagger UI

Access:

http://localhost:4566/_localstack/swagger

<img width="1427" height="968" alt="10" src="https://github.com/user-attachments/assets/9d5315a8-8891-44be-a0ac-44e5600e45df" />

<img width="1916" height="965" alt="11" src="https://github.com/user-attachments/assets/23de936a-4fff-487d-9195-99df9593ad41" />



Step 12 — Logs
localstack logs

<img width="1192" height="392" alt="12" src="https://github.com/user-attachments/assets/49b3083b-8b05-4b99-818c-2b92b397e222" />


🧹 Cleanup
localstack stop
docker ps --filter name=localstack

<img width="896" height="121" alt="13" src="https://github.com/user-attachments/assets/a1c7bfb9-1aba-4754-8c7c-f03a97c811ab" />



