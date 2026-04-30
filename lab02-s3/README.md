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

📸 Screenshot:

(Add LocalStack running screenshot here)

2. Create S3 Buckets

Three buckets were created:

awslocal s3 mb s3://lecafe-assets
awslocal s3 mb s3://lecafe-website
awslocal s3 mb s3://lecafe-logs

📸 Screenshot:

(Add bucket creation screenshot here)

3. Enable Versioning on Assets Bucket
   awslocal s3api put-bucket-versioning --bucket lecafe-assets --versioning-configuration Status=Enabled

Verify:

awslocal s3api get-bucket-versioning --bucket lecafe-assets

📸 Screenshot:

(Add versioning enabled screenshot here)

4. Upload Object (Version 1 and 2)
   echo "Le Café App Config — version 1" > config.txt
   awslocal s3 cp config.txt s3://lecafe-assets/app/config.txt

echo "Le Café App Config — version 2 (updated endpoint)" > config.txt
awslocal s3 cp config.txt s3://lecafe-assets/app/config.txt

📸 Screenshot:

(Add upload + overwrite screenshot here)

5. List Object Versions
   awslocal s3api list-object-versions --bucket lecafe-assets

📸 Screenshot:

(Add version list screenshot here)

6. Restore Previous Version
   awslocal s3api get-object \
    --bucket lecafe-assets \
    --key app/config.txt \
    --version-id <OLD_VERSION_ID> \
    recovered-config.txt

Verify:

type recovered-config.txt

📸 Screenshot:

(Add restored file screenshot here)

7. Delete Object (Create Delete Marker)
   awslocal s3 rm s3://lecafe-assets/app/config.txt

Check versions again:

awslocal s3api list-object-versions --bucket lecafe-assets

📸 Screenshot:

(Add delete marker screenshot here)

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

📸 Screenshot:

(Add cleanup screenshot here)
