# Lab 01 — IAM: Identity and Access Management (Le Café ☕)

## 🎯 Objective

This lab demonstrates how to design and implement a secure IAM architecture using LocalStack.  
It focuses on managing access using users, groups, policies, roles, and STS (Security Token Service), following AWS best practices and the principle of least privilege.

The goal is to simulate a real AWS IAM environment for a small application called **Le Café**.

---

## 🏪 Scenario — Le Café Cloud Architecture

Le Café is a growing application with different types of actors:

- Developers need access to S3 assets only
- Operations team manages infrastructure but not application data
- The backend application must read from S3 and send messages to SQS

Each actor must have **only the permissions they need and nothing more**.

---

## 🧠 What I learned

- Difference between IAM users, groups, policies, and roles
- How IAM enforces security using least privilege
- How to create and attach IAM policies in JSON format
- How IAM roles are assumed using STS
- How temporary credentials improve security over long-lived keys
- How real AWS authentication works in backend applications

---

## 🛠️ Tools used

- LocalStack (AWS simulation environment)
- AWS CLI Local (`awslocal`)
- PowerShell (Windows terminal)
- Docker (used internally by LocalStack)

---

## 👤 IAM Users Created

- alice → Developer
- bob → Developer
- charlie → Operations

<img width="1160" height="687" alt="1" src="https://github.com/user-attachments/assets/b14b2346-bd78-4555-86a0-6797497076b8" />

<img width="846" height="608" alt="2" src="https://github.com/user-attachments/assets/503958d6-af51-4f26-8ca9-8aff51fb6588" />


---

## 👥 IAM Groups Created

- cafe-developers
- cafe-operations

Users were assigned to groups instead of attaching policies directly, following AWS best practices.

<img width="1062" height="481" alt="3" src="https://github.com/user-attachments/assets/d2e1126f-b218-491e-b85e-7e97003404c4" />

<img width="1232" height="660" alt="4" src="https://github.com/user-attachments/assets/72b6a2fa-fc16-48f3-800a-35f56d522e6b" />




---

## 🔐 IAM Role Created

### Role Name:

- `lecafe-app-role`

This role represents the backend application identity.

It is configured with a **trust policy** allowing EC2 to assume it.

<img width="1000" height="456" alt="5" src="https://github.com/user-attachments/assets/7794da37-1125-413c-9ef8-b81e319ea731" />


---

## 📄 IAM Policies

### 1. Developer Policy

Allows developers to:

- Upload and download objects in `lecafe-assets` S3 bucket

### 2. Application Policy

Allows the backend application to:

- Read from S3 bucket (`lecafe-assets`)
- Send messages to SQS queue (`lecafe-orders`)

Policies follow the **principle of least privilege**.

<img width="1397" height="362" alt="6" src="https://github.com/user-attachments/assets/4421ef3a-f9fd-4d27-8d53-7e1df0666ac1" />

---

## 🏗️ AWS Resources Created

- S3 Bucket: `lecafe-assets`
- SQS Queue: `lecafe-orders`

<img width="1413" height="237" alt="10" src="https://github.com/user-attachments/assets/585fb509-58a6-47b8-9e44-cdf7a0324df9" />

---

## 🔄 Role Assumption (STS)

The application role was tested using STS:

```bash
awslocal sts assume-role --role-arn arn:aws:iam::000000000000:role/lecafe-app-role --role-session-name test-session
```
<img width="1396" height="398" alt="11" src="https://github.com/user-attachments/assets/adda4ae9-5870-4a06-901d-fe6a0ec96e99" />

