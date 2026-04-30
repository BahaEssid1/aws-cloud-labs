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

## ⚙️ Steps performed

### 1. Install LocalStack and dependencies

LocalStack and AWS CLI local wrapper were installed using pip:



pip install localstack awscli-local


2. Start LocalStack

LocalStack was started in local mode:

localstack start

<img width="1168" height="457" alt="1" src="https://github.com/user-attachments/assets/6c8d3486-0cf5-42c8-8874-43b5eb0ff425" />

3. Check LocalStack health

Verified that LocalStack services are running:

Invoke-RestMethod http://localhost:4566/_localstack/health

<img width="1421" height="151" alt="4" src="https://github.com/user-attachments/assets/153f4c57-2715-4825-a257-bdf302e889a2" />

<img width="755" height="426" alt="2" src="https://github.com/user-attachments/assets/c2202544-f775-439c-9a53-00834ea31eb4" />



4. Create S3 bucket

Created a local S3 bucket:

awslocal s3 mb s3://my-bucket

📸 Screenshot:

(Add bucket creation screenshot here)

5. Upload file to S3

Uploaded a file to the bucket:

awslocal s3 cp menu.txt s3://my-bucket

📸 Screenshot:

(Add file upload screenshot here)

6. List bucket contents

Verified uploaded file:

awslocal s3 ls s3://my-bucket

📸 Screenshot:

(Add list objects screenshot here)

7. Download file from S3

Downloaded the file from the bucket:

awslocal s3 cp s3://my-bucket/menu.txt menu-downloaded.txt

📸 Screenshot:

(Add file download screenshot here)
```
