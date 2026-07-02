# দিন ৩: AWS CLI — Command Line Interface
### Floci দিয়ে হাতে-কলমে শেখো (Git Bash)

> সব কমান্ড **Git Bash**-এ রান করতে হবে।
> কমান্ড রান করার আগে **Docker Desktop** অবশ্যই চালু থাকতে হবে।

---

## আজকে যা যা শিখবে

- AWS CLI কী এবং কেন DevOps-এ এত গুরুত্বপূর্ণ
- AWS CLI install এবং verify করা
- Real AWS-এ `aws configure` কীভাবে করে (তত্ত্ব)
- Floci দিয়ে configure ছাড়াই AWS CLI চালানো
- S3, EC2, IAM-এর উপর ১৫+ practical কমান্ড
- সাধারণ error এবং সমাধান

---

## পার্ট ১ — তত্ত্ব (Theory)

### AWS CLI কী?

**AWS CLI = AWS Command Line Interface**

AWS CLI হলো একটি tool যেটা দিয়ে terminal থেকে AWS-এর সব কাজ করা যায় — web console-এ না গিয়ে।

ব্রাউজারে click করার বদলে লেখো:
```bash
aws ec2 describe-instances
aws s3 ls
aws iam list-users
```

এক লাইনের কমান্ডে যা হয়, ব্রাউজারে সেটা করতে ৫-১০টা click লাগত।

---

### AWS CLI কেন দরকার? (DevOps দৃষ্টিকোণ)

| পরিস্থিতি | Console দিয়ে | CLI দিয়ে |
|-----------|-------------|---------|
| ১০০টা S3 bucket তৈরি | ১০০ বার click | একটা loop script |
| রাত ২টায় auto-deployment | মানুষকে জাগাতে হবে | CI/CD pipeline স্বয়ংক্রিয়ভাবে করবে |
| Production server restart | VPN → Console → click | `aws ec2 reboot-instances --instance-ids i-xxx` |
| Backup automation | প্রতিদিন manually | Cron job + CLI script |

**DevOps-এ যেসব tool CLI ব্যবহার করে:**
- **Jenkins** → deployment pipeline-এ `aws s3 cp` দিয়ে artifact upload করে
- **Terraform** → infrastructure তৈরি করার সময় AWS CLI-এর credentials ব্যবহার করে
- **GitHub Actions** → CI/CD-এ `aws ecr get-login-password` দিয়ে Docker push করে

---

### AWS CLI কীভাবে কাজ করে?

```
তুমি          AWS CLI        AWS API         AWS Service
  │               │               │               │
  │──command──▶   │               │               │
  │               │──HTTP req──▶  │               │
  │               │               │──process──▶   │
  │               │               │◀──response──  │
  │               │◀──JSON────    │               │
  │◀──output──    │               │               │
```

তুমি যখন `aws s3 ls` লেখো — CLI সেটাকে HTTPS request-এ রূপান্তর করে AWS-এর API-তে পাঠায়। AWS জবাব দেয় JSON-এ, CLI সেটা সুন্দরভাবে দেখায়।

---

### Real AWS-এ কীভাবে Configure করে (তত্ত্ব)

Real AWS account-এ কাজ করতে হলে CLI-কে জানাতে হয় তুমি কে এবং কোথায় কাজ করতে চাও। এটা করা হয় `aws configure` কমান্ড দিয়ে।

```bash
aws configure
```

এটা চারটি জিনিস জিজ্ঞেস করে:

| প্রশ্ন | কী দিতে হয় | কোথায় পাবে |
|--------|-----------|-----------|
| `AWS Access Key ID` | `AKIAXXXXXXXXXXXXXXXX` | IAM → Users → Security credentials |
| `AWS Secret Access Key` | `wJalrXUtnFEMI/...` | Access Key তৈরির সময় একবারই দেখা যায় |
| `Default region name` | `ap-southeast-1` | তোমার কাছের region |
| `Default output format` | `json` | json / table / text |

এই তথ্যগুলো save হয় দুটো file-এ:
```
~/.aws/credentials   ← Access Key এবং Secret Key
~/.aws/config        ← Region এবং output format
```

**সাধারণ AWS Region-সমূহ:**
```
ap-south-1       ← মুম্বাই (বাংলাদেশের কাছে)
ap-southeast-1   ← সিঙ্গাপুর
us-east-1        ← ভার্জিনিয়া (সবচেয়ে popular)
eu-west-1        ← আয়ারল্যান্ড
```

> **Floci-তে এটা দরকার নেই।** Floci local machine-এ চলে, real credential লাগে না। `eval $(floci env)` দিলেই কাজ হয়।

---

### IAM Access Key কী এবং কেন লাগে?

Real AWS-এ CLI ব্যবহার করতে হলে IAM user-এর **Access Key** লাগে।

Access Key দুটো অংশ নিয়ে:
- **Access Key ID** → username-এর মতো (public, দেখানো যায়)
- **Secret Access Key** → password-এর মতো (একবারই দেখা যায়, সংরক্ষণ করো)

Access Key তৈরির ধাপ (Real AWS):
```
AWS Console → IAM → Users → [user নির্বাচন]
→ Security credentials → Create access key
→ "Command Line Interface (CLI)" নির্বাচন
→ Create → Download .csv file
```

**Best Practices:**
- Root account-এর Access Key কখনো তৈরি করো না
- Secret Key হারিয়ে গেলে নতুন Key তৈরি করতে হবে (দেখা যায় না)
- Key নিয়মিত rotate করো
- কখনো code-এ বা GitHub-এ Key লিখো না

---

### Output Format তুলনা

একই কমান্ড তিনটি format-এ:

**json (default):**
```json
{
    "Buckets": [
        { "Name": "my-bucket", "CreationDate": "2026-07-01T00:00:00+00:00" }
    ]
}
```

**table:**
```
---------------------------------------------
|              ListBuckets                  |
+------------------+------------------------+
|  Name            |  CreationDate          |
+------------------+------------------------+
|  my-bucket       |  2026-07-01T00:00:00Z  |
+------------------+------------------------+
```

**text:**
```
my-bucket    2026-07-01T00:00:00Z
```

Format পরিবর্তন করতে: `aws s3 ls --output table`

---

### CLI Command-এর গঠন

```
aws  [service]  [operation]  [options]
 │      │           │            │
 │      │           │            └── --bucket-name, --region, --output
 │      │           └────────────── describe-instances, create-bucket
 │      └────────────────────────── s3, ec2, iam, lambda
 └───────────────────────────────── সবসময় "aws" দিয়ে শুরু
```

উদাহরণ:
```bash
aws    s3      ls        --output table
aws    ec2     describe-instances
aws    iam     list-users
aws    lambda  list-functions  --region us-east-1
```

---

### Best Practices

| নিয়ম | কেন |
|------|-----|
| Root account-এর Access Key তৈরি করো না | Root হারালে সব হারাবে |
| IAM user-এ least privilege দাও | CLI user-কেও minimum permission |
| Secret Key কখনো share করো না | এটাই তোমার AWS-এর চাবি |
| `~/.aws/` folder secure রাখো | এখানে credential আছে |
| Production-এ EC2 Role ব্যবহার করো, Access Key না | Role বেশি নিরাপদ |
| `--dry-run` flag ব্যবহার করো test-এর জন্য | আসল কাজ না করে দেখো command ঠিক আছে কিনা |

---

## পার্ট ২ — Floci দিয়ে হাতে-কলমে (Hands-On)

---

### ধাপ ০ — AWS CLI Install যাচাই করো

**কেন করছি?**
আগে নিশ্চিত হও AWS CLI আছে কিনা। না থাকলে install করতে হবে।

```bash
aws --version
```

**প্রত্যাশিত output:**
```
aws-cli/2.x.x Python/3.x.x ...
```

**যদি না থাকে — Linux/Git Bash-এ install করো:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Windows-এ:** [https://awscli.amazonaws.com/AWSCLIV2.msi](https://awscli.amazonaws.com/AWSCLIV2.msi) থেকে download করে install করো।

---

### ধাপ ১ — Floci চালু করো

**কেন করছি?**
Floci না চাললে কোনো `aws` কমান্ডই কাজ করবে না।

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

### ধাপ ২ — CLI Configure করো (Floci পদ্ধতি)

**কেন করছি?**
Real AWS-এ `aws configure` দিয়ে credential দিতে হয়। Floci-তে real credential লাগে না — `eval $(floci env)` একটা কমান্ডেই সব set হয়ে যায়।

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

**যাচাই করো:**
```bash
echo $AWS_ENDPOINT_URL
echo $AWS_DEFAULT_REGION
```

**প্রত্যাশিত output:**
```
http://localhost:4566
us-east-1
```

> **⚠️ গুরুত্বপূর্ণ — Real AWS-এ কানেক্ট করতে চাইলে:**
> `eval $(floci env)` একবার চালালে এই environment variable-গুলো তোমার Git Bash window বন্ধ না করা পর্যন্ত **সেট হয়েই থাকে**। একই session-এ পরে real AWS-এর সাথে কাজ করতে গেলে CLI তখনো `localhost:4566`-এ request পাঠাবে এবং এই error দেখাবে:
> ```
> Could not connect to the endpoint URL: "http://localhost:4566/"
> ```
> **সমাধান —** real AWS-এ যাওয়ার আগে এই variable-গুলো unset (cancel) করো:
> ```bash
> unset AWS_ENDPOINT_URL
> unset AWS_ACCESS_KEY_ID
> unset AWS_SECRET_ACCESS_KEY
> unset AWS_DEFAULT_REGION
> ```
> এরপর real credentials দিয়ে `aws configure` চালাও। সবচেয়ে নিরাপদ পদ্ধতি — real AWS-এর জন্য সবসময় **আলাদা, নতুন Git Bash window** ব্যবহার করা, যেখানে `floci env` কখনো চালানো হয়নি।

---

### ধাপ ৩ — পরিচয় যাচাই করো

**কেন করছি?**
Real AWS-এ `aws sts get-caller-identity` দিয়ে দেখা যায় তুমি ঠিক account-এ আছো কিনা। এটা DevOps-এ সবসময় প্রথম কাজ — ভুল account-এ কাজ করার ভুল এড়াতে।

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

> `Account: 000000000000` দেখলে বুঝবে Floci-এ আছো। Real AWS-এ তোমার ১২ সংখ্যার account number দেখাবে।

---

### ধাপ ৪ — S3 কমান্ড প্র্যাকটিস করো

**কেন করছি?**
S3 হলো AWS-এর সবচেয়ে বেশি ব্যবহৃত সার্ভিস। DevOps-এ প্রতিদিন S3 CLI কমান্ড লাগে — artifact store, log backup, static website hosting।

#### ৪.১ — Bucket তৈরি করো

```bash
aws s3 mb s3://my-demo-bucket
```

**প্রত্যাশিত output:**
```
make_bucket: my-demo-bucket
```

#### ৪.২ — সব Bucket দেখো

```bash
aws s3 ls
```

**প্রত্যাশিত output:**
```
2026-07-01 00:00:00 my-demo-bucket
```

#### ৪.৩ — ফাইল তৈরি করো

```bash
echo "Hello from AWS CLI!" > demo.txt
cat demo.txt
```

**প্রত্যাশিত output:**
```
Hello from AWS CLI!
```

#### ৪.৪ — S3-এ ফাইল upload করো

```bash
aws s3 cp demo.txt s3://my-demo-bucket/
```

**প্রত্যাশিত output:**
```
upload: ./demo.txt to s3://my-demo-bucket/demo.txt
```

#### ৪.৫ — Bucket-এর ভেতরের ফাইল দেখো

```bash
aws s3 ls s3://my-demo-bucket/
```

**প্রত্যাশিত output:**
```
2026-07-01 00:00:00         19 demo.txt
```

#### ৪.৬ — ফাইল download করো

```bash
aws s3 cp s3://my-demo-bucket/demo.txt downloaded.txt && cat downloaded.txt
```

**প্রত্যাশিত output:**
```
download: s3://my-demo-bucket/demo.txt to ./downloaded.txt
Hello from AWS CLI!
```

#### ৪.৭ — Bucket sync করো (folder আপলোড)

```bash
mkdir my-folder
echo "file 1" > my-folder/file1.txt
echo "file 2" > my-folder/file2.txt
aws s3 sync my-folder/ s3://my-demo-bucket/my-folder/
```

**প্রত্যাশিত output:**
```
upload: my-folder/file1.txt to s3://my-demo-bucket/my-folder/file1.txt
upload: my-folder/file2.txt to s3://my-demo-bucket/my-folder/file2.txt
```

#### ৪.৮ — Bucket মুছো

```bash
aws s3 rb s3://my-demo-bucket --force
```

**প্রত্যাশিত output:**
```
delete: s3://my-demo-bucket/demo.txt
delete: s3://my-demo-bucket/my-folder/file1.txt
delete: s3://my-demo-bucket/my-folder/file2.txt
remove_bucket: my-demo-bucket
```

> `--force` flag ছাড়া bucket-এ ফাইল থাকলে delete হবে না।

---

### ধাপ ৫ — IAM কমান্ড প্র্যাকটিস করো

**কেন করছি?**
CI/CD pipeline-এ অনেকসময় programmatically user তৈরি করতে হয় বা permission চেক করতে হয়। এই কমান্ডগুলো জানা থাকলে script লেখা সহজ হয়।

#### ৫.১ — Current user-এর তথ্য দেখো

```bash
aws iam get-user
```

**প্রত্যাশিত output:**
```json
{
    "User": {
        "Path": "/",
        "UserName": "root",
        "UserId": "AKIAXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::000000000000:root",
        "CreateDate": "2026-07-01T00:00:00+00:00"
    }
}
```

#### ৫.২ — সব IAM User দেখো

```bash
aws iam list-users
```

**প্রত্যাশিত output:**
```json
{
    "Users": []
}
```

#### ৫.৩ — নতুন user তৈরি করো

```bash
aws iam create-user --user-name cli-test-user
```

**প্রত্যাশিত output:**
```json
{
    "User": {
        "Path": "/",
        "UserName": "cli-test-user",
        "UserId": "AIXXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::000000000000:user/cli-test-user",
        "CreateDate": "2026-07-01T00:00:00+00:00"
    }
}
```

#### ৫.৪ — সব Group দেখো

```bash
aws iam list-groups
```

**প্রত্যাশিত output:**
```json
{
    "Groups": []
}
```

---

### ধাপ ৬ — EC2 কমান্ড প্র্যাকটিস করো

**কেন করছি?**
EC2 instance manage করা DevOps-এর daily কাজ। Server list দেখা, start/stop করা — সব CLI দিয়ে করা যায়।

#### ৬.১ — EC2 Instance-এর তালিকা দেখো

```bash
aws ec2 describe-instances
```

**প্রত্যাশিত output:**
```json
{
    "Reservations": []
}
```

#### ৬.২ — Available Region দেখো

```bash
aws ec2 describe-regions --output table
```

**প্রত্যাশিত output:**
```
------------------------------------------------------
|                   DescribeRegions                  |
+-------------------+--------------------------------+
|   RegionName      |   Endpoint                     |
+-------------------+--------------------------------+
|   ap-south-1      |   ec2.ap-south-1.amazonaws.com |
|   us-east-1       |   ec2.us-east-1.amazonaws.com  |
|   eu-west-1       |   ec2.eu-west-1.amazonaws.com  |
+-------------------+--------------------------------+
```

#### ৬.৩ — Security Group দেখো

```bash
aws ec2 describe-security-groups --output table
```

**প্রত্যাশিত output:**
```
------------------------------------------------------------
|               DescribeSecurityGroups                     |
+----------------------------+-----------------------------+
|  GroupId                   |  GroupName                  |
+----------------------------+-----------------------------+
|  sg-xxxxxxxxxxxxxxxx       |  default                    |
+----------------------------+-----------------------------+
```

---

### ধাপ ৬.৪ — EC2 Instance তৈরি করো (Launch)

**কেন করছি?**
EC2 instance তৈরি করা মানে cloud-এ একটি virtual server চালু করা। DevOps-এ server provisioning সবসময় CLI বা script দিয়ে করা হয়। এখানে পুরো workflow — key pair, security group, instance launch — একসাথে practice করব।

> **মনে রেখো:** Floci-এ EC2 real VM না — API simulate করে। Instance ID, IP দেখাবে কিন্তু SSH করা যাবে না। Command-এর syntax এবং workflow শেখাই উদ্দেশ্য।

#### ৬.৪.১ — Key Pair তৈরি করো

SSH connection-এর জন্য key pair লাগে।

```bash
aws ec2 create-key-pair --key-name my-key
```

**প্রত্যাশিত output:**
```json
{
    "KeyFingerprint": "xx:xx:xx:xx:xx:xx",
    "KeyName": "my-key",
    "KeyPairId": "key-xxxxxxxxxxxxxxxxx"
}
```

**যাচাই করো:**
```bash
aws ec2 describe-key-pairs
```

**প্রত্যাশিত output:**
```json
{
    "KeyPairs": [
        {
            "KeyName": "my-key",
            "KeyPairId": "key-xxxxxxxxxxxxxxxxx"
        }
    ]
}
```

#### ৬.৪.২ — Security Group তৈরি করো

Security group মানে firewall — কোন port থেকে traffic আসতে পারবে তা নিয়ন্ত্রণ করে।

```bash
aws ec2 create-security-group \
  --group-name my-sg \
  --description "My security group"
```

**প্রত্যাশিত output:**
```json
{
    "GroupId": "sg-xxxxxxxxxxxxxxxxx"
}
```

SSH (port 22) এবং HTTP (port 80) খুলে দাও:

```bash
aws ec2 authorize-security-group-ingress \
  --group-name my-sg \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-name my-sg \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

**প্রত্যাশিত output (দুটোর জন্যই):**
```json
{
    "Return": true,
    "SecurityGroupRules": [...]
}
```

#### ৬.৪.৩ — EC2 Instance Launch করো

```bash
aws ec2 run-instances \
  --image-id ami-12345678 \
  --instance-type t2.micro \
  --key-name my-key \
  --security-groups my-sg \
  --count 1
```

**প্রত্যাশিত output:**
```json
{
    "Instances": [
        {
            "InstanceId": "i-xxxxxxxxxxxxxxxxx",
            "InstanceType": "t2.micro",
            "ImageId": "ami-12345678",
            "State": {
                "Code": 0,
                "Name": "pending"
            },
            "KeyName": "my-key",
            "SecurityGroups": [
                { "GroupName": "my-sg" }
            ],
            "PublicIpAddress": "54.x.x.x",
            "PrivateIpAddress": "172.x.x.x"
        }
    ]
}
```

> **`InstanceId` note করো** — পরের সব কমান্ডে লাগবে।

#### ৬.৪.৪ — Instance-এর অবস্থা দেখো

```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType,PublicIpAddress]' \
  --output table
```

**প্রত্যাশিত output:**
```
--------------------------------------------------------------
|                     DescribeInstances                      |
+---------------------+---------+-----------+---------------+
|  i-xxxxxxxxxxxxxxx  | running | t2.micro  | 54.x.x.x     |
+---------------------+---------+-----------+---------------+
```

#### ৬.৪.৫ — Instance Stop, Start, Terminate করো

```bash
# বন্ধ করো
aws ec2 stop-instances --instance-ids i-xxxxxxxxxxxxxxxxx
```

**প্রত্যাশিত output:**
```json
{
    "StoppingInstances": [
        {
            "InstanceId": "i-xxxxxxxxxxxxxxxxx",
            "CurrentState": { "Name": "stopping" },
            "PreviousState": { "Name": "running" }
        }
    ]
}
```

```bash
# আবার চালু করো
aws ec2 start-instances --instance-ids i-xxxxxxxxxxxxxxxxx
```

**প্রত্যাশিত output:**
```json
{
    "StartingInstances": [
        {
            "InstanceId": "i-xxxxxxxxxxxxxxxxx",
            "CurrentState": { "Name": "pending" },
            "PreviousState": { "Name": "stopped" }
        }
    ]
}
```

```bash
# সম্পূর্ণ মুছে ফেলো
aws ec2 terminate-instances --instance-ids i-xxxxxxxxxxxxxxxxx
```

**প্রত্যাশিত output:**
```json
{
    "TerminatingInstances": [
        {
            "InstanceId": "i-xxxxxxxxxxxxxxxxx",
            "CurrentState": { "Name": "shutting-down" },
            "PreviousState": { "Name": "running" }
        }
    ]
}
```

---

### ধাপ ৭ — Output Format পরিবর্তন করো

**কেন করছি?**
Script-এ JSON দরকার, মানুষের পড়ার জন্য table সুবিধাজনক। `--output` flag দিয়ে যেকোনো সময় format পরিবর্তন করা যায়।

```bash
# JSON format (default)
aws iam list-users --output json

# Table format (সুন্দর দেখায়)
aws iam list-users --output table

# Text format (script-এ ব্যবহারের জন্য)
aws iam list-users --output text
```

#### Query দিয়ে specific তথ্য বের করো

```bash
aws iam list-users --query 'Users[*].UserName' --output text
```

**প্রত্যাশিত output:**
```
cli-test-user
```

---

### ধাপ ৮ — Help কমান্ড ব্যবহার করো

**কেন করছি?**
AWS-এর ১০০+ service-এর হাজারো কমান্ড মুখস্থ করা সম্ভব না। `help` জানলে যেকোনো কমান্ড নিজেই বের করা যায়।

```bash
# সব service দেখো
aws help

# একটি service-এর সব কমান্ড দেখো
aws s3 help

# একটি specific কমান্ডের বিস্তারিত দেখো
aws s3 cp help
```

---

### ধাপ ৯ — পরিষ্কার করো

**কেন করছি?**
Practice শেষে তৈরি করা resource মুছে রাখা ভালো অভ্যাস।

```bash
# IAM user মুছো
aws iam delete-user --user-name cli-test-user

# যাচাই করো
aws iam list-users
```

**প্রত্যাশিত output:**
```json
{
    "Users": []
}
```

---

## সাধারণ Error এবং সমাধান

| Error | কারণ | সমাধান |
|-------|------|--------|
| `Unable to locate credentials` | Env variable সেট নেই | `eval $(floci env)` আবার রান করো |
| `Could not connect to the endpoint URL` | Floci চলছে না | `floci start --persist ./floci-data` |
| `Could not connect to the endpoint URL: "http://localhost:4566/"` (real AWS-এর সাথে কাজ করার সময়) | Floci-এর env variable এখনো সেট আছে | `unset AWS_ENDPOINT_URL AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_DEFAULT_REGION` চালাও, অথবা নতুন Git Bash window খোলো |
| `InvalidClientTokenId` | ভুল Access Key | `echo $AWS_ACCESS_KEY_ID` যাচাই করো |
| `AuthFailure` | Secret Key ভুল | `eval $(floci env)` আবার রান করো |
| `NoSuchBucket` | Bucket নেই | আগে `aws s3 mb s3://bucket-name` করো |
| `BucketNotEmpty` | Bucket-এ ফাইল আছে | `--force` flag যোগ করো |

---

## দ্রুত তথ্যসূত্র — AWS CLI কমান্ড চিট শিট

### Setup
| কমান্ড | কী করে |
|--------|--------|
| `aws --version` | CLI version দেখো |
| `eval $(floci env)` | Floci-এর জন্য configure করো |
| `aws sts get-caller-identity` | বর্তমান identity যাচাই করো |

### S3
| কমান্ড | কী করে |
|--------|--------|
| `aws s3 ls` | সব bucket দেখো |
| `aws s3 mb s3://name` | Bucket তৈরি করো |
| `aws s3 cp file.txt s3://bucket/` | ফাইল upload করো |
| `aws s3 cp s3://bucket/file.txt .` | ফাইল download করো |
| `aws s3 ls s3://bucket/` | Bucket-এর ভেতর দেখো |
| `aws s3 sync folder/ s3://bucket/` | Folder sync করো |
| `aws s3 rm s3://bucket/file.txt` | ফাইল মুছো |
| `aws s3 rb s3://bucket --force` | Bucket মুছো |

### IAM
| কমান্ড | কী করে |
|--------|--------|
| `aws iam list-users` | সব user দেখো |
| `aws iam list-groups` | সব group দেখো |
| `aws iam list-roles` | সব role দেখো |
| `aws iam get-user` | Current user-এর তথ্য |
| `aws iam create-user --user-name <n>` | User তৈরি করো |

### EC2
| কমান্ড | কী করে |
|--------|--------|
| `aws ec2 describe-instances` | সব instance দেখো |
| `aws ec2 describe-regions` | সব region দেখো |
| `aws ec2 describe-security-groups` | Security group দেখো |

### Output Options
| Flag | কী করে |
|------|--------|
| `--output json` | JSON format (default) |
| `--output table` | Table format |
| `--output text` | Text format |
| `--query 'Key[*].Value'` | নির্দিষ্ট field বের করো |
| `--region us-east-1` | নির্দিষ্ট region ব্যবহার করো |

---

## আজকে যা শিখলে

```
AWS CLI
├── Theory
│   ├── CLI কী এবং কীভাবে কাজ করে
│   ├── Real AWS-এ aws configure (Access Key দিয়ে)
│   └── Floci-এ eval $(floci env) (credential ছাড়া)
└── Hands-On (Floci দিয়ে)
    ├── sts get-caller-identity → identity যাচাই
    ├── S3 → mb, ls, cp, sync, rb
    ├── IAM → list-users, create-user, delete-user
    ├── EC2 → describe-instances, describe-regions
    └── --output, --query → output format control
```

---

## বাড়ির কাজ

১. একটি S3 bucket তৈরি করো, ৩টি ভিন্ন ধরনের ফাইল upload করো এবং `--output table` দিয়ে দেখো
২. `aws iam list-users --query 'Users[*].UserName' --output text` কমান্ডটা নিয়ে গবেষণা করো — `--query` কীভাবে কাজ করে?
৩. `aws ec2 describe-instances --output table` এবং `--output json` — দুটোর পার্থক্য দেখো

---

## রিসোর্স

- Floci অফিসিয়াল: [https://floci.io](https://floci.io)
- Floci AWS সার্ভিস তালিকা: [https://floci.io/aws](https://floci.io/aws)
- AWS CLI Reference: [https://docs.aws.amazon.com/cli/latest/reference/](https://docs.aws.amazon.com/cli/latest/reference/)
- AWS CLI Install Guide: [https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
