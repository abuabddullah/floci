# দিন ২: AWS IAM — Identity and Access Management
### Floci দিয়ে হাতে-কলমে শেখো (Git Bash)

> সব কমান্ড **Git Bash**-এ রান করতে হবে।
> কমান্ড রান করার আগে **Docker Desktop** অবশ্যই চালু থাকতে হবে।

---

## আজকের লক্ষ্য

একটি কাল্পনিক company-র জন্য AWS IAM সেটআপ করব:
- `dev-user` → developer, EC2 ব্যবহার করতে পারবে
- `admin-user` → administrator, সবকিছু করতে পারবে
- `readonly-user` → auditor, শুধু S3 দেখতে পারবে

এই পুরো কাজটা Floci দিয়ে locally করব — কোনো real AWS account লাগবে না।

---

## পার্ট ১ — তত্ত্ব (Theory)

### IAM কী?

**IAM = Identity and Access Management**

IAM হলো AWS-এর সেই সার্ভিস যেটা নিয়ন্ত্রণ করে:
- **কে** তোমার AWS resource-এ প্রবেশ করতে পারবে
- **কী** করার অনুমতি তাদের আছে

IAM ছাড়া — যে কেউ তোমার সার্ভার মুছে দিতে পারে, S3 bucket খালি করে দিতে পারে, বা পুরো database ধ্বংস করে দিতে পারে।

---

### IAM কেন দরকার?

একটি কোম্পানিতে সাধারণত এরকম বিভিন্ন দল থাকে:

| দল | কী করে | কোন AWS service লাগে |
|----|--------|----------------------|
| DevOps Engineers | Infrastructure manage করে | EC2, S3, RDS, VPC সব |
| Developers | Application code লেখে | EC2, S3 (limited) |
| Security Team | Audit করে | শুধু দেখার permission |

IAM ছাড়া সবাইকে একই permission দিতে হতো — যেটা বিপজ্জনক। IAM দিয়ে প্রত্যেককে শুধু তার কাজের permission দেওয়া যায়।

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

> **বাস্তব উদাহরণ:** কোম্পানিতে ১০ জন নতুন developer আসলো। `Developers` group-এ add করলেই হলো — ১০টি আলাদা permission set করতে হলো না।

---

#### ৩. IAM Policies (অনুমতিপত্র)

একটি **policy** নির্ধারণ করে কোন কাজগুলো করা যাবে বা যাবে না।

Policy লেখা হয় **JSON format**-এ। প্রতিটি policy-তে থাকে:
- `Effect` → `Allow` (অনুমতি দাও) বা `Deny` (নিষেধ করো)
- `Action` → কোন কাজ (যেমন `s3:GetObject` মানে S3 থেকে file পড়া)
- `Resource` → কোন resource-এ (`*` মানে সবকিছুতে)

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

Policy দুই ধরনের:
- **AWS Managed Policy** → AWS নিজে তৈরি করা (যেমন `AmazonS3FullAccess`, `AdministratorAccess`)
- **Customer Managed Policy** → তুমি নিজে তৈরি করা (custom needs-এর জন্য)

---

#### ৪. IAM Roles (ভূমিকা)

একটি **role** হলো একটি অস্থায়ী পরিচয় যা AWS সার্ভিসগুলো বা মানুষ গ্রহণ করতে পারে।

ব্যবহারের উদাহরণ:
- EC2 instance-এর S3 থেকে পড়ার দরকার → EC2-তে S3 role attach করো
- Lambda function-কে DynamoDB-তে লেখার দরকার → Lambda-তে DynamoDB role attach করো
- Developer-এর সাময়িক admin access দরকার → সে admin role assume করবে

Role-এর কোনো স্থায়ী credential নেই। এটি STS-এর মাধ্যমে **অস্থায়ী token** দেয় যেটা নির্দিষ্ট সময় পরে expire হয়।

> **User vs Role:** User-এর password/access key স্থায়ী। Role-এর credential সাময়িক (১ ঘণ্টা থেকে ১২ ঘণ্টা)। তাই machine-এর জন্য সবসময় role ব্যবহার করা উচিত।

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
| Machine/service-এর জন্য Role ব্যবহার করো | User credential machine-এ রাখা বিপজ্জনক |

---

## পার্ট ২ — Floci দিয়ে হাতে-কলমে (Hands-On)

### আজকের লক্ষ্য

একটি কাল্পনিক company-র জন্য AWS IAM সেটআপ করব:
- `dev-user` → developer, EC2 ব্যবহার করতে পারবে
- `admin-user` → administrator, সবকিছু করতে পারবে
- `readonly-user` → auditor, শুধু S3 দেখতে পারবে

---

## হাতে-কলমে শুরু করো

---

### ধাপ ০ — Floci চালু করো

**কেন করছি?**
Floci না চাললে কোনো AWS কমান্ডই কাজ করবে না। `--persist` flag ছাড়া চালালে session শেষে সব তৈরি করা IAM resource মুছে যাবে।

```bash
floci start --persist ./floci-data
```

**প্রত্যাশিত output:**
```
Pulling floci/floci:latest ...
Starting Floci container...
✓ Floci is running on http://localhost:4566
Data will be persisted to: ./floci-data
```

---

### ধাপ ০.১ — Environment Variable সেট করো

**কেন করছি?**
AWS CLI জানে না কোথায় request পাঠাবে। এই variable গুলো দিয়ে বলছি "real AWS-এ না গিয়ে আমার PC-তে Floci-এ যাও।" `AWS_ACCESS_KEY_ID` এবং `AWS_SECRET_ACCESS_KEY`-এর মান কিছু হলেই চলে — Floci real credential যাচাই করে না।

```bash
eval $(floci env)
```

**প্রত্যাশিত output:**
```
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
```

Variable সেট হয়েছে কিনা যাচাই করো:

```bash
echo $AWS_ENDPOINT_URL
```

**প্রত্যাশিত output:**
```
http://localhost:4566
```

> **মনে রেখো:** Floci-এ AWS account ID হলো `000000000000` (বারোটি শূন্য)। সব ARN-এ এটাই দেখবে।

---

### ধাপ ১ — IAM User তৈরি করো

**কেন করছি?**
Company-তে তিনজন মানুষ আছে — একজন developer, একজন admin, একজন auditor। প্রত্যেকের জন্য আলাদা IAM user তৈরি করব। এতে কে কী করলো সেটা track করা যায় এবং যার যেটুকু permission দরকার ঠিক ততটুকুই দেওয়া যায়।

```bash
aws iam create-user --user-name dev-user
```

**প্রত্যাশিত output:**
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

```bash
aws iam create-user --user-name admin-user
aws iam create-user --user-name readonly-user
```

**প্রত্যাশিত output (প্রতিটির জন্য একই রকম, শুধু UserName আলাদা):**
```json
{
    "User": {
        "Path": "/",
        "UserName": "admin-user",
        "UserId": "AIXXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::000000000000:user/admin-user",
        "CreateDate": "2026-07-01T00:00:00+00:00"
    }
}
```

**যাচাই করো — তিনটি user তৈরি হয়েছে কিনা দেখো:**

```bash
aws iam list-users
```

**প্রত্যাশিত output:**
```json
{
    "Users": [
        { "UserName": "dev-user",      "Arn": "arn:aws:iam::000000000000:user/dev-user" },
        { "UserName": "admin-user",    "Arn": "arn:aws:iam::000000000000:user/admin-user" },
        { "UserName": "readonly-user", "Arn": "arn:aws:iam::000000000000:user/readonly-user" }
    ]
}
```

---

### ধাপ ২ — IAM Group তৈরি করো

**কেন করছি?**
প্রতিটি user-কে আলাদাভাবে permission দিলে ভুল হওয়ার সম্ভাবনা বেশি। Group তৈরি করে group-এ permission দিলে — সেই group-এ যে user আছে সে স্বয়ংক্রিয়ভাবে সেই permission পেয়ে যায়। কোম্পানিতে নতুন developer আসলে শুধু `Developers` group-এ add করলেই হবে, আলাদাভাবে permission দিতে হবে না।

```bash
aws iam create-group --group-name Developers
```

**প্রত্যাশিত output:**
```json
{
    "Group": {
        "Path": "/",
        "GroupName": "Developers",
        "GroupId": "AGPAXXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::000000000000:group/Developers",
        "CreateDate": "2026-07-01T00:00:00+00:00"
    }
}
```

```bash
aws iam create-group --group-name Admins
aws iam create-group --group-name ReadOnlyUsers
```

**যাচাই করো — তিনটি group তৈরি হয়েছে কিনা দেখো:**

```bash
aws iam list-groups
```

**প্রত্যাশিত output:**
```json
{
    "Groups": [
        { "GroupName": "Admins",        "Arn": "arn:aws:iam::000000000000:group/Admins" },
        { "GroupName": "Developers",    "Arn": "arn:aws:iam::000000000000:group/Developers" },
        { "GroupName": "ReadOnlyUsers", "Arn": "arn:aws:iam::000000000000:group/ReadOnlyUsers" }
    ]
}
```

---

### ধাপ ৩ — Custom IAM Policy তৈরি করো

**কেন করছি?**
AWS-এর built-in policy গুলো (যেমন `AmazonS3FullAccess`) সব S3 কাজের permission দেয়। কিন্তু auditor শুধু দেখতে পারবে, কিছু মুছতে বা লিখতে পারবে না। তাই নিজে একটি custom policy বানাচ্ছি যেটা শুধু S3 read করার permission দেবে।

**প্রথমে policy-র JSON file তৈরি করো:**

```bash
cat > s3-readonly-policy.json << 'EOF'
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
EOF
```

**প্রত্যাশিত output:**
```
(কোনো output আসবে না — এটা স্বাভাবিক, মানে file তৈরি হয়ে গেছে)
```

File সঠিক তৈরি হয়েছে কিনা দেখো:

```bash
cat s3-readonly-policy.json
```

**প্রত্যাশিত output:**
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

**এখন Floci-এ policy তৈরি করো:**

```bash
aws iam create-policy \
  --policy-name S3ReadOnlyPolicy \
  --policy-document file://s3-readonly-policy.json
```

**প্রত্যাশিত output:**
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
        "CreateDate": "2026-07-01T00:00:00+00:00"
    }
}
```

> **ARN-টা note করো:** `arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy` — পরের ধাপে লাগবে।

**যাচাই করো — নিজের তৈরি policy দেখো:**

```bash
aws iam list-policies --scope Local
```

**প্রত্যাশিত output:**
```json
{
    "Policies": [
        {
            "PolicyName": "S3ReadOnlyPolicy",
            "Arn": "arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy",
            "AttachmentCount": 0
        }
    ]
}
```

---

### ধাপ ৪ — Group-এ Policy Attach করো

**কেন করছি?**
এতক্ষণ group তৈরি করা হয়েছে কিন্তু কোনো permission দেওয়া হয়নি। এই ধাপে প্রতিটি group-কে তাদের কাজ অনুযায়ী permission দেব। Policy attach মানে বলছি "এই group এই কাজগুলো করতে পারবে।"

**ReadOnlyUsers group-এ আমাদের custom S3 read-only policy দাও:**

```bash
aws iam attach-group-policy \
  --group-name ReadOnlyUsers \
  --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy
```

**প্রত্যাশিত output:**
```
(কোনো output আসবে না — এটা স্বাভাবিক, মানে সফল হয়েছে)
```

**Developers group-এ AWS-এর built-in EC2 full access policy দাও:**

```bash
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
```

**Admins group-এ AWS-এর built-in সব কিছুর access দাও:**

```bash
aws iam attach-group-policy \
  --group-name Admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

**যাচাই করো — Developers group-এ কোন policy আছে:**

```bash
aws iam list-attached-group-policies --group-name Developers
```

**প্রত্যাশিত output:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "AmazonEC2FullAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonEC2FullAccess"
        }
    ]
}
```

**যাচাই করো — ReadOnlyUsers group-এ কোন policy আছে:**

```bash
aws iam list-attached-group-policies --group-name ReadOnlyUsers
```

**প্রত্যাশিত output:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "S3ReadOnlyPolicy",
            "PolicyArn": "arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy"
        }
    ]
}
```

---

### ধাপ ৫ — User-দের Group-এ যোগ করো

**কেন করছি?**
Group-এ policy দেওয়া হয়েছে, কিন্তু কোনো user এখনো কোনো group-এ নেই। User-কে group-এ add করলে সে স্বয়ংক্রিয়ভাবে সেই group-এর সব permission পাবে। এটাই IAM-এর সবচেয়ে কার্যকর ব্যবস্থাপনা পদ্ধতি।

```bash
aws iam add-user-to-group --group-name Developers    --user-name dev-user
aws iam add-user-to-group --group-name Admins        --user-name admin-user
aws iam add-user-to-group --group-name ReadOnlyUsers --user-name readonly-user
```

**প্রত্যাশিত output (প্রতিটির জন্য):**
```
(কোনো output আসবে না — এটা স্বাভাবিক, মানে সফল হয়েছে)
```

**যাচাই করো — Developers group-এ কে কে আছে:**

```bash
aws iam get-group --group-name Developers
```

**প্রত্যাশিত output:**
```json
{
    "Users": [
        {
            "UserName": "dev-user",
            "Arn": "arn:aws:iam::000000000000:user/dev-user"
        }
    ],
    "Group": {
        "GroupName": "Developers",
        "Arn": "arn:aws:iam::000000000000:group/Developers"
    }
}
```

**যাচাই করো — dev-user কোন কোন group-এ আছে:**

```bash
aws iam list-groups-for-user --user-name dev-user
```

**প্রত্যাশিত output:**
```json
{
    "Groups": [
        {
            "GroupName": "Developers",
            "Arn": "arn:aws:iam::000000000000:group/Developers"
        }
    ]
}
```

---

### ধাপ ৬ — User-এ সরাসরি Policy Attach করো

**কেন করছি?**
মাঝেমধ্যে একজন specific user-কে তার group-এর বাইরে অতিরিক্ত কিছু permission দিতে হয়। যেমন, `dev-user` developer হলেও সে S3-এর কিছু file দেখতে পারবে (তার project-এর config file)। Group-এ না দিয়ে সরাসরি user-এ দিচ্ছি কারণ এটা সবার দরকার নেই, শুধু এই একজনের দরকার।

```bash
aws iam attach-user-policy \
  --user-name dev-user \
  --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy
```

**প্রত্যাশিত output:**
```
(কোনো output আসবে না — মানে সফল হয়েছে)
```

**যাচাই করো — dev-user-এর নিজস্ব policy দেখো:**

```bash
aws iam list-attached-user-policies --user-name dev-user
```

**প্রত্যাশিত output:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "S3ReadOnlyPolicy",
            "PolicyArn": "arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy"
        }
    ]
}
```

> **dev-user এখন দুইভাবে S3 read access পাচ্ছে:** group থেকে না, সরাসরি user policy থেকে। Group থেকে পাচ্ছে EC2 FullAccess।

---

### ধাপ ৭ — User-এর জন্য Access Key তৈরি করো

**কেন করছি?**
`dev-user` যদি তার নিজের কম্পিউটার থেকে AWS CLI বা কোনো application দিয়ে AWS-এ connect করতে চায়, তাহলে তার password দিয়ে হবে না — দরকার হবে Access Key (একটি ID + একটি secret)। এটা তৈরি করে দিচ্ছি যাতে সে programmatically AWS ব্যবহার করতে পারে।

```bash
aws iam create-access-key --user-name dev-user
```

**প্রত্যাশিত output:**
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

> **সতর্কতা:** `SecretAccessKey` একবারই দেখা যায়। এটা note করো — পরে আর দেখা যাবে না।

**যাচাই করো — dev-user-এর access key আছে কিনা দেখো:**

```bash
aws iam list-access-keys --user-name dev-user
```

**প্রত্যাশিত output:**
```json
{
    "AccessKeyMetadata": [
        {
            "UserName": "dev-user",
            "AccessKeyId": "AKIAXXXXXXXXXXXXXXXX",
            "Status": "Active",
            "CreateDate": "2026-07-01T00:00:00+00:00"
        }
    ]
}
```

---

### ধাপ ৮ — IAM Role তৈরি করো

**কেন করছি?**
ধরো একটি EC2 server-কে S3 থেকে file পড়তে হবে। EC2 server কোনো মানুষ না — তাকে user-এর মতো username/password দেওয়া নিরাপদ না। তার বদলে একটি **Role** তৈরি করি এবং সেই role EC2-কে দিই। EC2 নিজে থেকে সেই role assume করে S3-এ access নেবে।

**প্রথমে trust policy file তৈরি করো — এটা বলে "EC2 service এই role নিতে পারবে":**

```bash
cat > ec2-trust-policy.json << 'EOF'
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
EOF
```

**প্রত্যাশিত output:**
```
(কোনো output আসবে না — file তৈরি হয়ে গেছে)
```

**এখন role তৈরি করো:**

```bash
aws iam create-role \
  --role-name EC2-S3-Access-Role \
  --assume-role-policy-document file://ec2-trust-policy.json
```

**প্রত্যাশিত output:**
```json
{
    "Role": {
        "Path": "/",
        "RoleName": "EC2-S3-Access-Role",
        "RoleId": "AROAXXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::000000000000:role/EC2-S3-Access-Role",
        "CreateDate": "2026-07-01T00:00:00+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [{ "Effect": "Allow", "Principal": { "Service": "ec2.amazonaws.com" }, "Action": "sts:AssumeRole" }]
        }
    }
}
```

**এই role-কে S3 read করার permission দাও:**

```bash
aws iam attach-role-policy \
  --role-name EC2-S3-Access-Role \
  --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy
```

**প্রত্যাশিত output:**
```
(কোনো output আসবে না — মানে সফল হয়েছে)
```

**যাচাই করো — role তৈরি হয়েছে কিনা:**

```bash
aws iam list-roles
```

**প্রত্যাশিত output (আংশিক):**
```json
{
    "Roles": [
        {
            "RoleName": "EC2-S3-Access-Role",
            "Arn": "arn:aws:iam::000000000000:role/EC2-S3-Access-Role"
        }
    ]
}
```

**যাচাই করো — role-এ কোন policy আছে:**

```bash
aws iam list-attached-role-policies --role-name EC2-S3-Access-Role
```

**প্রত্যাশিত output:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "S3ReadOnlyPolicy",
            "PolicyArn": "arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy"
        }
    ]
}
```

---

### ধাপ ৯ — Role Assume করো (STS)

**কেন করছি?**
এটা real DevOps-এ সবচেয়ে বেশি ব্যবহৃত একটি workflow। ধরো তুমি একজন developer, কিন্তু production deployment-এর জন্য সাময়িকভাবে admin permission দরকার। তুমি admin role assume করবে → সাময়িক credential পাবে → কাজ করবে → সেই credential expire হয়ে যাবে। এতে নিরাপত্তা বজায় থাকে কারণ credential স্থায়ী না।

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::000000000000:role/EC2-S3-Access-Role \
  --role-session-name my-test-session
```

**প্রত্যাশিত output:**
```json
{
    "Credentials": {
        "AccessKeyId": "ASIAXXXXXXXXXXXXXXXX",
        "SecretAccessKey": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "SessionToken": "FQoGZXIvYXdzExxxxxxxxxxxxxxxxxxxxxxxxxx",
        "Expiration": "2026-07-01T01:00:00+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAXXXXXXXXXXXXXXXXXX:my-test-session",
        "Arn": "arn:aws:iam::000000000000:assumed-role/EC2-S3-Access-Role/my-test-session"
    }
}
```

> **Output-এ যা দেখলে বুঝবে সফল হয়েছে:**
> - `Credentials` section-এ তিনটি জিনিস আছে: `AccessKeyId`, `SecretAccessKey`, `SessionToken`
> - এই তিনটি মিলিয়ে হলো অস্থায়ী credential — `Expiration` পর্যন্ত কাজ করবে
> - `AssumedRoleUser`-এর ARN-এ `assumed-role/EC2-S3-Access-Role/my-test-session` দেখাচ্ছে — মানে role সফলভাবে assume হয়েছে

---

### ধাপ ১০ — বর্তমান পরিচয় যাচাই করো

**কেন করছি?**
তুমি এখন কে হিসেবে authenticated আছো সেটা যাচাই করার জন্য। Real-world-এ ভুল account-এ কাজ করার ঘটনা হরহামেশা ঘটে — এই কমান্ড দিয়ে সবসময় নিশ্চিত হওয়া যায়।

```bash
aws sts get-caller-identity
```

**প্রত্যাশিত output:**
```json
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

> `Account: 000000000000` দেখলে বুঝবে Floci-এ আছো (real AWS-এ তোমার real account number থাকত)।

---

### ধাপ ১১ — পরিষ্কার করো (ঐচ্ছিক)

**কেন করছি?**
ভবিষ্যতে আবার এই exercise করতে চাইলে নতুন করে শুরু করা দরকার হতে পারে। AWS-এ resource-এর উপর নির্ভর করে delete-এর একটি নির্দিষ্ট ক্রম মানতে হয় — নাহলে error আসে।

```bash
# ১. Group থেকে user সরাও (আগে এটা করতে হবে)
aws iam remove-user-from-group --group-name Developers    --user-name dev-user
aws iam remove-user-from-group --group-name Admins        --user-name admin-user
aws iam remove-user-from-group --group-name ReadOnlyUsers --user-name readonly-user
```

**প্রত্যাশিত output (প্রতিটির জন্য):**
```
(কোনো output আসবে না — মানে সফল হয়েছে)
```

```bash
# ২. Group থেকে policy detach করো
aws iam detach-group-policy --group-name Developers    --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
aws iam detach-group-policy --group-name Admins        --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam detach-group-policy --group-name ReadOnlyUsers --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy

# ৩. Group মুছো
aws iam delete-group --group-name Developers
aws iam delete-group --group-name Admins
aws iam delete-group --group-name ReadOnlyUsers

# ৪. User থেকে policy detach করো
aws iam detach-user-policy --user-name dev-user --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy

# ৫. User-এর access key-এর ID বের করো
aws iam list-access-keys --user-name dev-user
```

**প্রত্যাশিত output:**
```json
{
    "AccessKeyMetadata": [
        { "AccessKeyId": "AKIAXXXXXXXXXXXXXXXX", "Status": "Active" }
    ]
}
```

```bash
# ৬. সেই ID দিয়ে access key মুছো (AKIAXXXXXXXXXXXXXXXX-এর জায়গায় আসল ID বসাও)
aws iam delete-access-key --user-name dev-user --access-key-id AKIAXXXXXXXXXXXXXXXX

# ৭. User মুছো
aws iam delete-user --user-name dev-user
aws iam delete-user --user-name admin-user
aws iam delete-user --user-name readonly-user

# ৮. Role থেকে policy detach করো
aws iam detach-role-policy --role-name EC2-S3-Access-Role --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy

# ৯. Role মুছো
aws iam delete-role --role-name EC2-S3-Access-Role

# ১০. Custom policy মুছো (সবার শেষে)
aws iam delete-policy --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy
```

---

## দ্রুত তথ্যসূত্র — IAM কমান্ড চিট শিট

| কী করতে হবে | কমান্ড |
|------------|--------|
| User তৈরি | `aws iam create-user --user-name <নাম>` |
| সব user দেখো | `aws iam list-users` |
| Group তৈরি | `aws iam create-group --group-name <নাম>` |
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

## আজকে যা তৈরি করলে

```
Company IAM Structure
├── Users
│   ├── dev-user      → Developers group (EC2 FullAccess) + S3ReadOnly (direct)
│   ├── admin-user    → Admins group (AdministratorAccess)
│   └── readonly-user → ReadOnlyUsers group (S3ReadOnly)
├── Groups
│   ├── Developers    → AmazonEC2FullAccess
│   ├── Admins        → AdministratorAccess
│   └── ReadOnlyUsers → S3ReadOnlyPolicy (custom)
├── Policies
│   └── S3ReadOnlyPolicy (custom) → s3:GetObject, s3:ListBucket
└── Roles
    └── EC2-S3-Access-Role → EC2 service can assume → S3ReadOnly access
```

---

## বাড়ির কাজ

১. একটি নতুন user `qa-user` এবং group `QA` তৈরি করো, যেখানে শুধু S3 full access থাকবে
২. একটি custom policy তৈরি করো যেটা Lambda read-only access দেবে
৩. একটি role তৈরি করো যেটা Lambda service-কে DynamoDB-তে লিখতে দেবে

---

## রিসোর্স

- Floci অফিসিয়াল: [https://floci.io](https://floci.io)
- Floci AWS সার্ভিস তালিকা: [https://floci.io/aws](https://floci.io/aws)
- AWS IAM CLI Reference: [https://docs.aws.amazon.com/cli/latest/reference/iam/](https://docs.aws.amazon.com/cli/latest/reference/iam/)
