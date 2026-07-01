# দিন ২: AWS IAM — Identity and Access Management
### Floci দিয়ে হাতে-কলমে শেখো (Local AWS Emulator)

> **Floci কী?**
> Floci হলো একটি ফ্রি, ওপেন-সোর্স AWS emulator। এটি তোমার PC-তে `http://localhost:4566` ঠিকানায় চলে।
> Real AWS account ছাড়াই সমস্ত AWS কমান্ড practice করা যায়।

---

## আজকে যা যা শিখবে

- IAM কী এবং কেন দরকার
- IAM Users, Groups, Policies এবং Roles — তত্ত্ব + হাতে-কলমে
- AWS CLI + Floci দিয়ে real IAM resource তৈরি করা
- Policy attach করা, user-কে group-এ যোগ করা, role তৈরি করা
- STS দিয়ে Role Assume করা (real DevOps workflow)

---

## পার্ট ১ — তত্ত্ব (Theory)

### IAM কী?

**IAM = Identity and Access Management**

IAM হলো AWS-এর সেই সার্ভিস যেটা নিয়ন্ত্রণ করে:
- **কে** তোমার AWS resource-এ প্রবেশ করতে পারবে
- **কী** করার অনুমতি তাদের আছে

IAM ছাড়া — যে কেউ তোমার সার্ভার মুছে দিতে পারে, S3 bucket খালি করে দিতে পারে, বা পুরো database ধ্বংস করে দিতে পারে।

---

### IAM-এর মূল উপাদানসমূহ

#### ১. IAM Users (ব্যবহারকারী)
একটি **user** একজন মানুষ বা একটি application-কে প্রতিনিধিত্ব করে।

উদাহরণ:
```
dev-user        ← একজন developer
admin-user      ← একজন system administrator
ci-bot          ← একটি CI/CD pipeline automation account
```

প্রত্যেক user-এর নিজস্ব credential (access key বা password) থাকে।

---

#### ২. IAM Groups (দল)
একটি **group** হলো একাধিক user-এর সমষ্টি যারা একই permission ভোগ করে।

উদাহরণ:
```
Developers      ← সমস্ত developer
Admins          ← সমস্ত administrator
ReadOnlyUsers   ← যারা শুধু resource দেখতে পারবে
```

Group-এ permission দাও → group-এর প্রতিটি user সেই permission পেয়ে যায়।
প্রতিটি user-কে আলাদাভাবে permission দেওয়ার চেয়ে এটা অনেক সহজ।

---

#### ৩. IAM Policies (অনুমতিপত্র)
একটি **policy** নির্ধারণ করে কোন কাজগুলো করা যাবে বা যাবে না।

Policy লেখা হয় **JSON format**-এ।

উদাহরণ — EC2 Read-Only Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2ReadOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:Get*"
      ],
      "Resource": "*"
    }
  ]
}
```

এই policy EC2 instance দেখার অনুমতি দেয়, কিন্তু তৈরি বা মুছে ফেলার অনুমতি দেয় না।

---

#### ৪. IAM Roles (ভূমিকা)
একটি **role** হলো একটি অস্থায়ী পরিচয় যা AWS সার্ভিসগুলো গ্রহণ করতে পারে।

ব্যবহারের উদাহরণ:
- EC2 instance-এর S3 থেকে পড়ার দরকার → EC2-তে S3 role attach করো
- Lambda function-কে DynamoDB-তে লেখার দরকার → Lambda-তে DynamoDB role attach করো
- Developer-এর সাময়িক admin access দরকার → সে admin role assume করবে

Role-এর কোনো স্থায়ী credential নেই। এটি STS-এর মাধ্যমে **অস্থায়ী token** দেয়।

---

### IAM Best Practices (সর্বোত্তম অভ্যাস)

| নিয়ম | কেন |
|------|-----|
| দৈনন্দিন কাজে root account ব্যবহার করো না | Root-এর সীমাহীন ক্ষমতা আছে — একটি ভুলে সবকিছু ধ্বংস হতে পারে |
| আলাদা IAM user তৈরি করো | প্রত্যেক ব্যক্তির নিজস্ব credential থাকা উচিত |
| Group ব্যবহার করে permission manage করো | প্রতিটি user আলাদাভাবে manage করার চেয়ে সহজ |
| Least privilege নীতি মেনে চলো | শুধু প্রয়োজনীয় minimum permission দাও |
| MFA চালু করো | গুরুত্বপূর্ণ account-এর জন্য অতিরিক্ত নিরাপত্তা স্তর |
| নিয়মিত access key পরিবর্তন করো | পুরনো key নিরাপত্তার ঝুঁকি |

---

## পার্ট ২ — Floci দিয়ে হাতে-কলমে (Hands-On)

### পূর্বশর্ত

নিশ্চিত করো এগুলো install করা আছে (দিন ১ থেকে):
- Docker Desktop (চালু অবস্থায়)
- AWS CLI v2
- Floci CLI

---

### ধাপ ০ — Floci চালু করো

Terminal খোলো এবং রান করো:

```bash
floci start
```

প্রত্যাশিত output:
```
✓ Floci is running on http://localhost:4566
```

---

### ধাপ ০.১ — Environment Variable সেট করো

**Windows (PowerShell):**
```powershell
$env:AWS_ENDPOINT_URL      = "http://localhost:4566"
$env:AWS_DEFAULT_REGION    = "us-east-1"
$env:AWS_ACCESS_KEY_ID     = "test"
$env:AWS_SECRET_ACCESS_KEY = "test"
```

**Mac / Linux / Git Bash:**
```bash
eval $(floci env)
```

অথবা manually:
```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
```

> **মনে রেখো:** Floci-এ AWS account ID হলো `000000000000` (বারোটি শূন্য)। ARN-গুলোতে এটা দেখবে।

---

### ধাপ ১ — IAM User তৈরি করো

তিনটি user তৈরি করো, যারা ভিন্ন ভিন্ন team member-কে প্রতিনিধিত্ব করে:

```bash
aws iam create-user --user-name dev-user
aws iam create-user --user-name admin-user
aws iam create-user --user-name readonly-user
```

প্রতিটির জন্য প্রত্যাশিত output:
```json
{
    "User": {
        "Path": "/",
        "UserName": "dev-user",
        "UserId": "AIXXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::000000000000:user/dev-user",
        "CreateDate": "2026-07-01T00:00:00+00:00"
    }
}
```

**যাচাই করো — সব user দেখো:**
```bash
aws iam list-users
```

---

### ধাপ ২ — IAM Group তৈরি করো

তিনটি দল বা team-এর জন্য তিনটি group তৈরি করো:

```bash
aws iam create-group --group-name Developers
aws iam create-group --group-name Admins
aws iam create-group --group-name ReadOnlyUsers
```

**যাচাই করো — সব group দেখো:**
```bash
aws iam list-groups
```

---

### ধাপ ৩ — Custom IAM Policy তৈরি করো

একটি custom policy তৈরি করব যেটা শুধু S3 read access দেবে।

প্রথমে policy document file তৈরি করো। `s3-readonly-policy.json` নামে save করো:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadOnlyAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "*"
    }
  ]
}
```

এখন Floci-এ policy তৈরি করো:

```bash
aws iam create-policy \
  --policy-name S3ReadOnlyPolicy \
  --policy-document file://s3-readonly-policy.json
```

প্রত্যাশিত output:
```json
{
    "Policy": {
        "PolicyName": "S3ReadOnlyPolicy",
        "PolicyId": "ANPAXXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "IsAttachable": true,
        "CreateDate": "2026-07-01T00:00:00+00:00",
        "UpdateDate": "2026-07-01T00:00:00+00:00"
    }
}
```

**যাচাই করো — নিজের তৈরি policy দেখো:**
```bash
aws iam list-policies --scope Local
```

---

### ধাপ ৪ — Group-এ Policy Attach করো

`ReadOnlyUsers` group-এ custom S3 policy attach করো:

```bash
aws iam attach-group-policy \
  --group-name ReadOnlyUsers \
  --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy
```

`Developers` group-এ AWS managed EC2 full access attach করো:

```bash
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
```

`Admins` group-এ AWS managed AdministratorAccess attach করো:

```bash
aws iam attach-group-policy \
  --group-name Admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

**যাচাই করো — group-এ attached policy দেখো:**
```bash
aws iam list-attached-group-policies --group-name Developers
aws iam list-attached-group-policies --group-name ReadOnlyUsers
```

---

### ধাপ ৫ — User-দের Group-এ যোগ করো

এখন প্রতিটি user-কে তার সঠিক group-এ রাখো:

```bash
aws iam add-user-to-group --group-name Developers    --user-name dev-user
aws iam add-user-to-group --group-name Admins        --user-name admin-user
aws iam add-user-to-group --group-name ReadOnlyUsers --user-name readonly-user
```

**যাচাই করো — group-এর user দেখো:**
```bash
aws iam get-group --group-name Developers
aws iam get-group --group-name Admins
```

**যাচাই করো — একজন user কোন কোন group-এ আছে:**
```bash
aws iam list-groups-for-user --user-name dev-user
```

---

### ধাপ ৬ — User-এ সরাসরি Policy Attach করো

কখনো কখনো একজন specific user-কে তার group-এর বাইরে অতিরিক্ত permission দিতে হয়।

`dev-user`-এ সরাসরি S3 read access দাও:

```bash
aws iam attach-user-policy \
  --user-name dev-user \
  --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy
```

**যাচাই করো — user-এ সরাসরি attached policy দেখো:**
```bash
aws iam list-attached-user-policies --user-name dev-user
```

---

### ধাপ ৭ — User-এর জন্য Access Key তৈরি করো

Access key দিয়ে user CLI বা SDK-এ authenticate করতে পারে (password ছাড়া)।

```bash
aws iam create-access-key --user-name dev-user
```

প্রত্যাশিত output:
```json
{
    "AccessKey": {
        "UserName": "dev-user",
        "AccessKeyId": "AKIAXXXXXXXXXXXXXXXX",
        "Status": "Active",
        "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
        "CreateDate": "2026-07-01T00:00:00+00:00"
    }
}
```

**User-এর access key দেখো:**
```bash
aws iam list-access-keys --user-name dev-user
```

---

### ধাপ ৮ — IAM Role তৈরি করো

Role AWS সার্ভিসগুলো ব্যবহার করে। চলো একটি role তৈরি করি যেটা EC2-কে S3 access করতে দেবে।

প্রথমে trust policy document তৈরি করো। `ec2-trust-policy.json` নামে save করো:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Role তৈরি করো:

```bash
aws iam create-role \
  --role-name EC2-S3-Access-Role \
  --assume-role-policy-document file://ec2-trust-policy.json
```

প্রত্যাশিত output:
```json
{
    "Role": {
        "Path": "/",
        "RoleName": "EC2-S3-Access-Role",
        "RoleId": "AROAXXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::000000000000:role/EC2-S3-Access-Role",
        "CreateDate": "2026-07-01T00:00:00+00:00"
    }
}
```

এই role-এ S3 read policy attach করো:

```bash
aws iam attach-role-policy \
  --role-name EC2-S3-Access-Role \
  --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy
```

**যাচাই করো — সব role দেখো:**
```bash
aws iam list-roles
```

**যাচাই করো — role-এ attached policy দেখো:**
```bash
aws iam list-attached-role-policies --role-name EC2-S3-Access-Role
```

---

### ধাপ ৯ — Role Assume করো (STS)

এটি একটি real DevOps workflow — সাময়িকভাবে একটি role assume করে তার permission পাওয়া।

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::000000000000:role/EC2-S3-Access-Role \
  --role-session-name my-test-session
```

প্রত্যাশিত output:
```json
{
    "Credentials": {
        "AccessKeyId": "ASIAXXXXXXXXXXXXXXXX",
        "SecretAccessKey": "...",
        "SessionToken": "...",
        "Expiration": "2026-07-01T01:00:00+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAXXXXXXXXXXXXXXXXXX:my-test-session",
        "Arn": "arn:aws:iam::000000000000:assumed-role/EC2-S3-Access-Role/my-test-session"
    }
}
```

অস্থায়ী credential (`AccessKeyId`, `SecretAccessKey`, `SessionToken`) দিয়ে এখন এই role-এর মতো কাজ করা যাবে।

---

### ধাপ ১০ — বর্তমান পরিচয় যাচাই করো

তুমি এখন কে হিসেবে authenticated আছো তা দেখো:

```bash
aws sts get-caller-identity
```

প্রত্যাশিত output:
```json
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

---

### ধাপ ১১ — পরিষ্কার করো (ঐচ্ছিক)

তৈরি করা সব resource মুছে ফেলো:

```bash
# Group থেকে user সরাও
aws iam remove-user-from-group --group-name Developers    --user-name dev-user
aws iam remove-user-from-group --group-name Admins        --user-name admin-user
aws iam remove-user-from-group --group-name ReadOnlyUsers --user-name readonly-user

# Group থেকে policy detach করো
aws iam detach-group-policy --group-name Developers    --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
aws iam detach-group-policy --group-name Admins        --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam detach-group-policy --group-name ReadOnlyUsers --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy

# Group মুছো
aws iam delete-group --group-name Developers
aws iam delete-group --group-name Admins
aws iam delete-group --group-name ReadOnlyUsers

# User থেকে policy detach করো
aws iam detach-user-policy --user-name dev-user --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy

# User মুছার আগে access key মুছো
aws iam list-access-keys --user-name dev-user
# উপরের output থেকে AccessKeyId কপি করো, তারপর:
aws iam delete-access-key --user-name dev-user --access-key-id <AccessKeyId>

# User মুছো
aws iam delete-user --user-name dev-user
aws iam delete-user --user-name admin-user
aws iam delete-user --user-name readonly-user

# Role থেকে policy detach করো
aws iam detach-role-policy --role-name EC2-S3-Access-Role --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy

# Role মুছো
aws iam delete-role --role-name EC2-S3-Access-Role

# Custom policy মুছো
aws iam delete-policy --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy
```

---

## দ্রুত তথ্যসূত্র — IAM কমান্ডসমূহ

| কী করতে হবে | কমান্ড |
|------------|--------|
| User তৈরি | `aws iam create-user --user-name <name>` |
| সব user দেখো | `aws iam list-users` |
| Group তৈরি | `aws iam create-group --group-name <name>` |
| সব group দেখো | `aws iam list-groups` |
| User-কে group-এ যোগ করো | `aws iam add-user-to-group --group-name <g> --user-name <u>` |
| Policy তৈরি | `aws iam create-policy --policy-name <n> --policy-document file://policy.json` |
| Group-এ policy attach | `aws iam attach-group-policy --group-name <g> --policy-arn <arn>` |
| User-এ policy attach | `aws iam attach-user-policy --user-name <u> --policy-arn <arn>` |
| Role তৈরি | `aws iam create-role --role-name <n> --assume-role-policy-document file://trust.json` |
| Role-এ policy attach | `aws iam attach-role-policy --role-name <n> --policy-arn <arn>` |
| Role assume করো | `aws sts assume-role --role-arn <arn> --role-session-name <session>` |
| বর্তমান পরিচয় দেখো | `aws sts get-caller-identity` |

---

## আজকে যা practice করলে

- [x] IAM User তৈরি করা (`dev-user`, `admin-user`, `readonly-user`)
- [x] IAM Group তৈরি করা (`Developers`, `Admins`, `ReadOnlyUsers`)
- [x] Custom IAM Policy তৈরি করা (S3 Read-Only)
- [x] Group-এ managed ও custom policy attach করা
- [x] User-দের group-এ যোগ করা
- [x] User-এ সরাসরি policy attach করা
- [x] User-এর জন্য access key তৈরি করা
- [x] Trust policy সহ IAM Role তৈরি করা
- [x] STS দিয়ে Role Assume করা
- [x] `get-caller-identity` দিয়ে পরিচয় যাচাই করা

---

## বাড়ির কাজ

১. একটি নতুন user `qa-user` এবং group `QA` তৈরি করো, যেখানে শুধু S3 full access থাকবে
২. একটি custom policy তৈরি করো যেটা Lambda read-only access দেবে
৩. একটি role তৈরি করো যেটা Lambda-কে DynamoDB-তে লিখতে দেবে

---

## রিসোর্স

- Floci অফিসিয়াল: [https://floci.io](https://floci.io)
- Floci AWS সার্ভিস তালিকা: [https://floci.io/aws](https://floci.io/aws)
- AWS IAM CLI Reference: [https://docs.aws.amazon.com/cli/latest/reference/iam/](https://docs.aws.amazon.com/cli/latest/reference/iam/)
- AWS IAM User Guide: [https://docs.aws.amazon.com/IAM/latest/UserGuide/](https://docs.aws.amazon.com/IAM/latest/UserGuide/)
