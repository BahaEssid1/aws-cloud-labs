# Lab 02 — S3: Object Storage and Lifecycle Management

## 🎯 Objective

The objective of this lab is to understand how Amazon S3 works as an object storage system using LocalStack.  
We simulate real AWS behavior locally by creating buckets, enabling versioning, managing objects, and applying lifecycle concepts.

This lab demonstrates how S3 is not a traditional file system and how versioning and deletion behave in cloud storage systems.

---

## 🧠 What I learned

- How S3 stores data as objects (not folders)
- How bucket versioning works in AWS S3
- How object overwriting creates multiple versions
- How to restore previous versions using VersionId
- How delete markers work in versioned buckets
- How to permanently delete objects in S3
- Basic lifecycle and data management concepts in cloud storage

---

## 🛠️ Tools used

- LocalStack
- AWS CLI Local (`awslocal`)
- Docker (backend for LocalStack)
- PowerShell (Windows CLI)

---

## 📁 Project Files

- `config.txt` → test file used for versioning demonstration
- `recovered-config.txt` → file restored from an older version

---

## 🏗️ Lab Architecture

- `lecafe-assets` → private versioned bucket (application data)
- `lecafe-website` → static website bucket
- `lecafe-logs` → log storage bucket with lifecycle rules

---

## ⚙️ Steps performed

---

### 1. Start LocalStack

LocalStack was started in detached mode:

```bash
localstack start -d
```


2. Create S3 Buckets

Three buckets were created:

awslocal s3 mb s3://lecafe-assets
awslocal s3 mb s3://lecafe-website
awslocal s3 mb s3://lecafe-logs

<img width="1357" height="642" alt="1" src="https://github.com/user-attachments/assets/f5553327-e68c-4376-8a51-3b5f09f396a9" />


3. Enable Versioning on Assets Bucket
   awslocal s3api put-bucket-versioning --bucket lecafe-assets --versioning-configuration Status=Enabled

Verify:

awslocal s3api get-bucket-versioning --bucket lecafe-assets

<img width="1407" height="170" alt="2" src="https://github.com/user-attachments/assets/5d81728d-f1fa-473b-af9e-b22f140339be" />


4. Upload Object (Version 1 and 2)
   echo "Le Café App Config — version 1" > config.txt
   awslocal s3 cp config.txt s3://lecafe-assets/app/config.txt

echo "Le Café App Config — version 2 (updated endpoint)" > config.txt
awslocal s3 cp config.txt s3://lecafe-assets/app/config.txt

<img width="1116" height="132" alt="step5" src="https://github.com/user-attachments/assets/79bcbc12-1378-4216-816a-0a9a76f7e994" />


5. List Object Versions
   awslocal s3api list-object-versions --bucket lecafe-assets

   <img width="1300" height="786" alt="step55" src="https://github.com/user-attachments/assets/075c6454-ac63-49f8-a8d6-568c94c07a1c" />


<img width="931" height="358" alt="step6" src="https://github.com/user-attachments/assets/d0bb0022-6e2d-476e-a0c3-f2167a3a9fd9" />


6. Restore Previous Version
   awslocal s3api get-object \
    --bucket lecafe-assets \
    --key app/config.txt \
    --version-id <OLD_VERSION_ID> \
    recovered-config.txt

Verify:

type recovered-config.txt

<img width="1055" height="113" alt="step7" src="https://github.com/user-attachments/assets/a385b7a7-2899-45bc-b714-d27892b2ca57" />



7. Delete Object (Create Delete Marker)
   awslocal s3 rm s3://lecafe-assets/app/config.txt

Check versions again:

awslocal s3api list-object-versions --bucket lecafe-assets

<img width="1253" height="677" alt="step77" src="https://github.com/user-attachments/assets/836e6634-554b-45b0-b2ee-3706f30a2841" />


8. Permanent Cleanup

Remove all versions and delete markers:

$versions = awslocal s3api list-object-versions --bucket lecafe-assets | ConvertFrom-Json

foreach ($v in $versions.Versions) {
awslocal s3api delete-object --bucket lecafe-assets --key $v.Key --version-id $v.VersionId
}

foreach ($m in $versions.DeleteMarkers) {
awslocal s3api delete-object --bucket lecafe-assets --key $m.Key --version-id $m.VersionId
}

Delete buckets:

awslocal s3 rb s3://lecafe-assets
awslocal s3 rb s3://lecafe-website
awslocal s3 rb s3://lecafe-logs

<img width="1292" height="698" alt="step8" src="https://github.com/user-attachments/assets/cb6e0260-18f1-461a-9b0b-93a1c2bca664" />


