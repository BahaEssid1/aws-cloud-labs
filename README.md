# ☕ Le Café — AWS Hands-On Labs (LocalStack)

## 📌 Overview

This repository contains a complete **hands-on AWS learning series** built around a fictional system called **Le Café**.

The goal is to simulate real-world **cloud architecture and DevOps practices** using **LocalStack** instead of a real AWS account.

Each lab progressively builds a full **event-driven microservices architecture** using AWS services like S3, IAM, SQS, SNS, and CloudFormation.

---

## 📁 Repository Structure

The project is divided into independent lab folders:

```
lab00/ — LocalStack Setup
lab01/ — S3 Basics
lab02/ — IAM & Security
lab03/ — SQS Messaging
lab04/ — SQS & SNS Event-Driven Architecture
lab05/ — Infrastructure as Code (CloudFormation)
```

---

## 📄 Important Rule

👉 **Each lab contains its own README.md file**

Every lab is fully self-contained and includes:

- 🎯 Objectives
- 🧠 Theory and explanations
- 🛠️ Step-by-step CLI commands
- 📸 Screenshots
- ⚙️ Implementation details
- 🧹 Cleanup instructions
- ✅ Expected outputs

You must open and follow each lab individually.

---

## 🚀 How to Start

### 1. Start from Lab 00

```bash
cd lab00
```

Then follow the README inside that folder.

---

### 2. Progress in Order

Each lab depends on the previous one:

- **Lab 00** → Setup LocalStack environment
- **Lab 01** → S3 fundamentals
- **Lab 02** → IAM security & permissions
- **Lab 03** → SQS message queues
- **Lab 04** → SNS + SQS event-driven architecture (fan-out)
- **Lab 05** → CloudFormation (Infrastructure as Code)

---

