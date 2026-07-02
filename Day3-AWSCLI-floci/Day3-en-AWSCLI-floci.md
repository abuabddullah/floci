# Day 3: AWS CLI — Command Line Interface
### Hands-on with Floci (Git Bash)

> All commands must be run in **Git Bash**.
> **Docker Desktop** must be running in the background before you run any command.

---

## What You Will Learn Today

- What AWS CLI is and why it matters in DevOps
- Installing and verifying AWS CLI
- How `aws configure` works in real AWS (theory)
- Configuring CLI with Floci — no credentials needed
- 15+ practical commands for S3, EC2, and IAM
- Common errors and how to fix them

---

## Part 1 — Theory

### What is AWS CLI?

**AWS CLI = AWS Command Line Interface**

AWS CLI is a tool that lets you manage all AWS services directly from your terminal — without using the web console.

Instead of clicking through a browser:
```bash
aws ec2 describe-instances
aws s3 ls
aws iam list-users
```

One command does what would take 5–10 clicks in the browser.

---

### Why AWS CLI Matters (DevOps Perspective)

| Situation | With Console | With CLI |
|-----------|-------------|---------|
| Create 100 S3 buckets | 100 clicks | One loop script |
| Auto-deploy at 2 AM | Wake someone up | CI/CD pipeline does it |
| Restart production server | VPN → Console → click | `aws ec2 reboot-instances --instance-ids i-xxx` |
| Daily backups | Manually every day | Cron job + CLI script |

**DevOps tools that use CLI heavily:**
- **Jenkins** → uploads build artifacts with `aws s3 cp`
- **Terraform** → uses AWS CLI credentials to create infrastructure
- **GitHub Actions** → logs into ECR with `aws ecr get-login-password` to push Docker images

---

### How AWS CLI Works

```
You           AWS CLI       AWS API        AWS Service
 │               │               │               │
 │──command──▶   │               │               │
 │               │──HTTP req──▶  │               │
 │               │               │──process──▶   │
 │               │               │◀──response──  │
 │               │◀──JSON────    │               │
 │◀──output──    │               │               │
```

When you type `aws s3 ls`, the CLI converts it into an HTTPS request to the AWS API. AWS replies in JSON, and the CLI displays it in a readable format.

---

### How to Configure CLI in Real AWS (Theory)

In real AWS, you need to tell the CLI who you are and where you want to work. This is done with `aws configure`.

```bash
aws configure
```

It asks four things:

| Prompt | What to Enter | Where to Find It |
|--------|--------------|-----------------|
| `AWS Access Key ID` | `AKIAXXXXXXXXXXXXXXXX` | IAM → Users → Security credentials |
| `AWS Secret Access Key` | `wJalrXUtnFEMI/...` | Only visible once when created |
| `Default region name` | `ap-southeast-1` | Your nearest region |
| `Default output format` | `json` | json / table / text |

This saves credentials to two files:
```
~/.aws/credentials   ← Access Key ID and Secret Access Key
~/.aws/config        ← Region and output format
```

**Common AWS Regions:**
```
ap-south-1       ← Mumbai (closest to Bangladesh)
ap-southeast-1   ← Singapore
us-east-1        ← Virginia (most popular)
eu-west-1        ← Ireland
```

> **You don't need this with Floci.** Floci runs locally — no real credentials required. `eval $(floci env)` handles everything.

---

### What is an IAM Access Key?

In real AWS, CLI authentication requires an IAM user **Access Key**.

An Access Key has two parts:
- **Access Key ID** → like a username (can be shared)
- **Secret Access Key** → like a password (shown only once — save it!)

Steps to create an Access Key (Real AWS):
```
AWS Console → IAM → Users → [select user]
→ Security credentials → Create access key
→ Select "Command Line Interface (CLI)"
→ Create → Download .csv file
```

**Best Practices:**
- Never create an Access Key for the root account
- If you lose the Secret Key, create a new one (it can't be retrieved)
- Rotate keys regularly
- Never hardcode keys in code or commit them to GitHub

---

### Output Format Comparison

Same command, three formats:

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

Switch format with: `aws s3 ls --output table`

---

### CLI Command Structure

```
aws  [service]  [operation]  [options]
 │      │           │            │
 │      │           │            └── --bucket-name, --region, --output
 │      │           └────────────── describe-instances, create-bucket
 │      └────────────────────────── s3, ec2, iam, lambda
 └───────────────────────────────── always starts with "aws"
```

Examples:
```bash
aws    s3      ls        --output table
aws    ec2     describe-instances
aws    iam     list-users
aws    lambda  list-functions  --region us-east-1
```

---

### Best Practices

| Rule | Why |
|------|-----|
| Never create an Access Key for root | Losing root means losing everything |
| Give CLI users least privilege | Minimum permissions only |
| Never share your Secret Key | It's the key to your AWS account |
| Keep `~/.aws/` folder secure | Credentials are stored there |
| Use EC2 Roles in production, not Access Keys | Roles are safer than static keys |
| Use `--dry-run` flag for testing | Test a command without executing it |

---

## Part 2 — Hands-On with Floci

---

### Step 0 — Verify AWS CLI Installation

**Why?**
Make sure AWS CLI is installed before anything else.

```bash
aws --version
```

**Expected output:**
```
aws-cli/2.x.x Python/3.x.x ...
```

**If not installed — Linux/Git Bash:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Windows:** Download and install [AWSCLIV2.msi](https://awscli.amazonaws.com/AWSCLIV2.msi)

---

### Step 1 — Start Floci

**Why?**
Without Floci running, no `aws` commands will work.

```bash
floci start --persist ./floci-data
```

**Expected output:**
```
Pulling floci/floci:latest ...
Starting Floci container...
✓ Floci is running on http://localhost:4566
Data will be persisted to: ./floci-data
```

---

### Step 2 — Configure CLI (Floci Method)

**Why?**
In real AWS, you'd run `aws configure` and enter real credentials. With Floci, `eval $(floci env)` sets everything automatically — no real credentials needed.

```bash
eval $(floci env)
```

**Expected output:**
```
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
```

**Verify:**
```bash
echo $AWS_ENDPOINT_URL
echo $AWS_DEFAULT_REGION
```

**Expected output:**
```
http://localhost:4566
us-east-1
```

> **⚠️ Important — If you want to connect to real AWS:**
> Once you run `eval $(floci env)`, these environment variables **stay set** in that Git Bash window until you close it. If you later try to work with real AWS in the same session, the CLI will still send requests to `localhost:4566` and you'll see this error:
> ```
> Could not connect to the endpoint URL: "http://localhost:4566/"
> ```
> **Fix —** unset these variables before switching to real AWS:
> ```bash
> unset AWS_ENDPOINT_URL
> unset AWS_ACCESS_KEY_ID
> unset AWS_SECRET_ACCESS_KEY
> unset AWS_DEFAULT_REGION
> ```
> Then run `aws configure` with your real credentials. The safest approach is to always use a **separate, fresh Git Bash window** for real AWS work — one where `floci env` was never run.

---

### Step 3 — Verify Your Identity

**Why?**
In real AWS, `aws sts get-caller-identity` confirms you're in the right account. In DevOps, this is always the first check — to prevent accidentally working in the wrong account.

```bash
aws sts get-caller-identity
```

**Expected output:**
```json
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

> `Account: 000000000000` confirms you're in Floci. In real AWS, you'd see your actual 12-digit account number.

---

### Step 4 — Practice S3 Commands

**Why?**
S3 is the most-used AWS service. S3 CLI commands are used daily in DevOps — artifact storage, log backups, static website hosting.

#### 4.1 — Create a Bucket

```bash
aws s3 mb s3://my-demo-bucket
```

**Expected output:**
```
make_bucket: my-demo-bucket
```

#### 4.2 — List All Buckets

```bash
aws s3 ls
```

**Expected output:**
```
2026-07-01 00:00:00 my-demo-bucket
```

#### 4.3 — Create a File

```bash
echo "Hello from AWS CLI!" > demo.txt
cat demo.txt
```

**Expected output:**
```
Hello from AWS CLI!
```

#### 4.4 — Upload File to S3

```bash
aws s3 cp demo.txt s3://my-demo-bucket/
```

**Expected output:**
```
upload: ./demo.txt to s3://my-demo-bucket/demo.txt
```

#### 4.5 — List Files Inside Bucket

```bash
aws s3 ls s3://my-demo-bucket/
```

**Expected output:**
```
2026-07-01 00:00:00         19 demo.txt
```

#### 4.6 — Download File from S3

```bash
aws s3 cp s3://my-demo-bucket/demo.txt downloaded.txt && cat downloaded.txt
```

**Expected output:**
```
download: s3://my-demo-bucket/demo.txt to ./downloaded.txt
Hello from AWS CLI!
```

#### 4.7 — Sync a Folder to S3

```bash
mkdir my-folder
echo "file 1" > my-folder/file1.txt
echo "file 2" > my-folder/file2.txt
aws s3 sync my-folder/ s3://my-demo-bucket/my-folder/
```

**Expected output:**
```
upload: my-folder/file1.txt to s3://my-demo-bucket/my-folder/file1.txt
upload: my-folder/file2.txt to s3://my-demo-bucket/my-folder/file2.txt
```

#### 4.8 — Delete a Bucket

```bash
aws s3 rb s3://my-demo-bucket --force
```

**Expected output:**
```
delete: s3://my-demo-bucket/demo.txt
delete: s3://my-demo-bucket/my-folder/file1.txt
delete: s3://my-demo-bucket/my-folder/file2.txt
remove_bucket: my-demo-bucket
```

> Without `--force`, the command fails if the bucket has files inside.

---

### Step 5 — Practice IAM Commands

**Why?**
CI/CD pipelines often create users or check permissions programmatically. Knowing these commands makes automation scripting much easier.

#### 5.1 — Get Current User Info

```bash
aws iam get-user
```

**Expected output:**
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

#### 5.2 — List All Users

```bash
aws iam list-users
```

**Expected output:**
```json
{
    "Users": []
}
```

#### 5.3 — Create a User

```bash
aws iam create-user --user-name cli-test-user
```

**Expected output:**
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

#### 5.4 — List All Groups

```bash
aws iam list-groups
```

**Expected output:**
```json
{
    "Groups": []
}
```

---

### Step 6 — Practice EC2 Commands

**Why?**
Managing EC2 instances is a core DevOps responsibility. Listing servers, checking status — all done via CLI in real pipelines.

#### 6.1 — List EC2 Instances

```bash
aws ec2 describe-instances
```

**Expected output:**
```json
{
    "Reservations": []
}
```

#### 6.2 — List Available Regions

```bash
aws ec2 describe-regions --output table
```

**Expected output:**
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

#### 6.3 — List Security Groups

```bash
aws ec2 describe-security-groups --output table
```

**Expected output:**
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

### Step 6.4 — Create and Launch an EC2 Instance

**Why?**
Creating an EC2 instance means launching a virtual server in the cloud. In DevOps, server provisioning is always done via CLI or scripts. Here we practice the full workflow — key pair, security group, instance launch.

> **Remember:** EC2 in Floci is simulated — not a real VM. You'll get an instance ID and IP address, but you cannot SSH into it. The goal is to learn the command syntax and workflow.

#### 6.4.1 — Create a Key Pair

A key pair is required for SSH connections.

```bash
aws ec2 create-key-pair --key-name my-key
```

**Expected output:**
```json
{
    "KeyFingerprint": "xx:xx:xx:xx:xx:xx",
    "KeyName": "my-key",
    "KeyPairId": "key-xxxxxxxxxxxxxxxxx"
}
```

**Verify:**
```bash
aws ec2 describe-key-pairs
```

**Expected output:**
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

#### 6.4.2 — Create a Security Group

A security group acts as a firewall — it controls which ports allow traffic.

```bash
aws ec2 create-security-group \
  --group-name my-sg \
  --description "My security group"
```

**Expected output:**
```json
{
    "GroupId": "sg-xxxxxxxxxxxxxxxxx"
}
```

Open SSH (port 22) and HTTP (port 80):

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

**Expected output (for both):**
```json
{
    "Return": true,
    "SecurityGroupRules": [...]
}
```

#### 6.4.3 — Launch an EC2 Instance

```bash
aws ec2 run-instances \
  --image-id ami-12345678 \
  --instance-type t2.micro \
  --key-name my-key \
  --security-groups my-sg \
  --count 1
```

**Expected output:**
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

> **Note the `InstanceId`** — you'll need it for all subsequent commands.

#### 6.4.4 — Check Instance Status

```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType,PublicIpAddress]' \
  --output table
```

**Expected output:**
```
--------------------------------------------------------------
|                     DescribeInstances                      |
+---------------------+---------+-----------+---------------+
|  i-xxxxxxxxxxxxxxx  | running | t2.micro  | 54.x.x.x     |
+---------------------+---------+-----------+---------------+
```

#### 6.4.5 — Stop, Start, and Terminate an Instance

```bash
# Stop the instance
aws ec2 stop-instances --instance-ids i-xxxxxxxxxxxxxxxxx
```

**Expected output:**
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
# Start it again
aws ec2 start-instances --instance-ids i-xxxxxxxxxxxxxxxxx
```

**Expected output:**
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
# Permanently delete it
aws ec2 terminate-instances --instance-ids i-xxxxxxxxxxxxxxxxx
```

**Expected output:**
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

### Step 7 — Control Output Format

**Why?**
Scripts need JSON, humans prefer tables. The `--output` flag lets you switch format any time.

```bash
# JSON (default)
aws iam list-users --output json

# Table (human-readable)
aws iam list-users --output table

# Text (for shell scripts)
aws iam list-users --output text
```

#### Extract specific fields with --query

```bash
aws iam list-users --query 'Users[*].UserName' --output text
```

**Expected output:**
```
cli-test-user
```

---

### Step 8 — Use the Help Command

**Why?**
You can't memorize thousands of commands across 100+ services. Knowing how to use `help` means you can figure out any command yourself.

```bash
# List all available services
aws help

# List all commands for a service
aws s3 help

# Get detailed help for a specific command
aws s3 cp help
```

---

### Step 9 — Clean Up

**Why?**
Good practice to clean up resources after practice.

```bash
aws iam delete-user --user-name cli-test-user
aws iam list-users
```

**Expected output:**
```json
{
    "Users": []
}
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Unable to locate credentials` | Env vars not set | Run `eval $(floci env)` again |
| `Could not connect to the endpoint URL` | Floci not running | Run `floci start --persist ./floci-data` |
| `Could not connect to the endpoint URL: "http://localhost:4566/"` (while working with real AWS) | Floci's env variables are still set | Run `unset AWS_ENDPOINT_URL AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_DEFAULT_REGION`, or open a fresh Git Bash window |
| `InvalidClientTokenId` | Wrong Access Key | Check `echo $AWS_ACCESS_KEY_ID` |
| `AuthFailure` | Wrong Secret Key | Run `eval $(floci env)` again |
| `NoSuchBucket` | Bucket doesn't exist | Create it first with `aws s3 mb s3://name` |
| `BucketNotEmpty` | Bucket has files | Add `--force` flag |

---

## Quick Reference — AWS CLI Cheat Sheet

### Setup
| Command | What it does |
|---------|-------------|
| `aws --version` | Check CLI version |
| `eval $(floci env)` | Configure for Floci |
| `aws sts get-caller-identity` | Verify current identity |

### S3
| Command | What it does |
|---------|-------------|
| `aws s3 ls` | List all buckets |
| `aws s3 mb s3://name` | Create a bucket |
| `aws s3 cp file.txt s3://bucket/` | Upload a file |
| `aws s3 cp s3://bucket/file.txt .` | Download a file |
| `aws s3 ls s3://bucket/` | List files in a bucket |
| `aws s3 sync folder/ s3://bucket/` | Sync a folder |
| `aws s3 rm s3://bucket/file.txt` | Delete a file |
| `aws s3 rb s3://bucket --force` | Delete a bucket |

### IAM
| Command | What it does |
|---------|-------------|
| `aws iam list-users` | List all users |
| `aws iam list-groups` | List all groups |
| `aws iam list-roles` | List all roles |
| `aws iam get-user` | Current user info |
| `aws iam create-user --user-name <n>` | Create a user |

### EC2
| Command | What it does |
|---------|-------------|
| `aws ec2 describe-instances` | List all instances |
| `aws ec2 describe-regions` | List all regions |
| `aws ec2 describe-security-groups` | List security groups |

### Output Options
| Flag | What it does |
|------|-------------|
| `--output json` | JSON format (default) |
| `--output table` | Table format |
| `--output text` | Text format |
| `--query 'Key[*].Value'` | Extract specific fields |
| `--region us-east-1` | Use a specific region |

---

## What You Practiced Today

```
AWS CLI
├── Theory
│   ├── What CLI is and how it works
│   ├── Real AWS: aws configure (with Access Key)
│   └── Floci: eval $(floci env) (no credentials needed)
└── Hands-On (with Floci)
    ├── sts get-caller-identity → verify identity
    ├── S3 → mb, ls, cp, sync, rb
    ├── IAM → list-users, create-user, delete-user
    ├── EC2 → describe-instances, describe-regions
    └── --output, --query → format control
```

---

## Homework

1. Create an S3 bucket, upload 3 different types of files, and view them with `--output table`
2. Research how `--query` works: try `aws iam list-users --query 'Users[*].UserName' --output text`
3. Compare `aws ec2 describe-instances --output table` vs `--output json` — when would you use each?

---

## Resources

- Floci Official: [https://floci.io](https://floci.io)
- Floci AWS Services: [https://floci.io/aws](https://floci.io/aws)
- AWS CLI Reference: [https://docs.aws.amazon.com/cli/latest/reference/](https://docs.aws.amazon.com/cli/latest/reference/)
- AWS CLI Install Guide: [https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
