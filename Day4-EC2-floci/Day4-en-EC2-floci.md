# Day 4: Amazon EC2 — Elastic Compute Cloud
### Hands-on with Floci (Local AWS Emulator) — Git Bash

> All CLI commands must be run in **Git Bash**.
> **Docker Desktop** must be running before any command.

---

## What You Will Learn Today

- What EC2 is and why virtual servers matter in the cloud
- Key Pairs and how SSH authentication works
- Security Groups — cloud firewall
- EC2 Instance Types and Pricing Models
- Launch, stop, start, and terminate EC2 instances using Floci CLI
- User Data — automatically install software at instance launch
- Launch Templates — save configuration and reuse it
- How to SSH into a real AWS EC2 instance
- Common EC2 mistakes and how to fix them

---

## Part 1 — Theory

### What is EC2?

**EC2 = Elastic Compute Cloud**

Amazon EC2 is the AWS service that creates virtual servers in the cloud.

"Elastic" means: scale up when you need more power, scale down or stop when you don't.

EC2 can run:
- Websites and web applications
- API servers
- Docker containers
- Machine learning models
- Jenkins, Terraform, Kubernetes nodes

---

### Why EC2? (DevOps Perspective)

| Physical Server | AWS EC2 |
|----------------|---------|
| Once bought, you're locked in | Create when needed, stop when not |
| If hardware fails, you're stuck | AWS handles all maintenance |
| Adding capacity takes days | New server in 5 minutes |
| Tied to one location | Available 24/7 globally |

**EC2 in DevOps:**
- Running Jenkins CI/CD servers
- Deploying production applications
- Docker/Kubernetes cluster nodes
- Database servers (alternative to RDS)

---

### Key Pairs

A **Key Pair** is the authentication mechanism for SSH access to EC2 instances.

Two parts:
- **Public Key** → stored in AWS (placed inside the EC2 instance)
- **Private Key (.pem file)** → stored on your computer only

How it works:
```
You                          EC2 Instance
  │                                │
  │──ssh -i my-key.pem──▶          │
  │   (using private key)          │
  │                                │── matches with public key
  │◀─── connection established ─── │
```

> **Warning:** If you lose the `.pem` file, you cannot SSH in again. You must create a new key pair.

---

### Security Groups

A **Security Group** is EC2's **virtual firewall**.

Controls:
- **Inbound rules** → what traffic can enter from outside
- **Outbound rules** → what traffic can leave from inside

Common inbound rules:

| Type | Protocol | Port | Purpose | Source |
|------|----------|------|---------|--------|
| SSH | TCP | 22 | Server access | My IP |
| HTTP | TCP | 80 | Website | 0.0.0.0/0 |
| HTTPS | TCP | 443 | Secure website | 0.0.0.0/0 |
| Custom TCP | TCP | 8080 | Jenkins | My IP |

> **Rule:** Always set SSH port 22 to `My IP`, never `0.0.0.0/0` — open SSH to everyone and hackers will find you.

---

### EC2 Instance Types

An instance type defines the hardware configuration of your server.

| Type | vCPU | RAM | Use Case |
|------|------|-----|----------|
| `t2.micro` | 1 | 1 GB | Free tier, testing |
| `t2.small` | 1 | 2 GB | Light apps |
| `t2.large` | 2 | 8 GB | Medium apps |
| `m5.large` | 2 | 8 GB | Balanced, production |
| `c5.large` | 2 | 4 GB | CPU intensive (Jenkins, compile) |
| `r5.large` | 2 | 16 GB | Memory intensive (DB, cache) |

**Understanding the naming pattern:**
```
t  2  .  micro
│  │       └── size: nano, micro, small, medium, large, xlarge
│  └────────── generation: 2, 3, 4, 5...
└───────────── family: t=general, m=balanced, c=compute, r=memory
```

---

### EC2 Pricing Models

#### 1. On-Demand
- Pay per second or per hour
- No commitment
- **When to use:** Testing, unpredictable workloads

#### 2. Reserved Instances
- 1-year or 3-year commitment
- Up to 72% savings vs On-Demand
- **When to use:** Production servers running 24/7

#### 3. Spot Instances
- AWS unused capacity — cheapest (up to 90% savings)
- AWS can terminate at any time with 2-minute notice
- **When to use:** Batch processing, data analysis, non-critical jobs

#### 4. Free Tier
- `t2.micro` — 750 hours free per month
- First 12 months only
- **When to use:** Learning and experimentation

---

### AMI (Amazon Machine Image)

An AMI is the **blueprint** for your EC2 instance — which operating system and pre-installed software it starts with.

Common AMIs:
```
Amazon Linux 2          ← AWS's own, most popular
Ubuntu 22.04 LTS        ← Developers' favorite
Windows Server 2022     ← Windows workloads
Red Hat Enterprise      ← Enterprise production
```

---

### User Data

**User Data** is a shell script that runs **automatically when an EC2 instance boots for the first time**.

```
Instance launch
     │
     ▼
OS boots
     │
     ▼
User Data script runs  ← nginx/docker/app installs here
     │
     ▼
Instance ready
```

**When to use it:**
- Automatically install a web server (nginx, apache)
- Deploy an application
- Create configuration files
- Start Docker

> **Floci note:** Floci accepts `--user-data` at the API level, but the script does not execute — there is no real VM. On real AWS, this is standard production practice.

---

### Launch Templates

A **Launch Template** is a **saved EC2 configuration**.

Create it once with your AMI, instance type, key pair, security group, and user data. After that, launching a new instance takes one short command — everything comes from the template.

**When you need it:**
- Auto Scaling Groups (many instances with identical config)
- CI/CD pipelines for automated deployment
- Ensuring the whole team uses consistent instance settings

---

### Best Practices

| Rule | Why |
|------|-----|
| Set SSH to `My IP`, not `0.0.0.0/0` | Open SSH = hackers will get in |
| Keep `.pem` file secure (`chmod 400`) | Wrong permissions → SSH refuses to connect |
| Use Reserved Instances for production | Significant cost savings |
| Stop instances when not in use | Stopped instances don't charge for compute |
| Backup data before terminating | Terminate is permanent — data is gone |
| Tag every instance | Easier to identify what's running and why |
| Use User Data to automate setup | Manual SSH installs are error-prone |
| Use Launch Templates | Keeps configuration consistent across the team |

---

## Part 2 — Hands-On with Floci (CLI)

> **Note:** Floci simulates the EC2 **API** — no real VM is running.
> You can create instances but cannot SSH into them.
> Goal: learn CLI commands and the complete EC2 workflow.

---

### Step 0 — Start Floci

**Why:** No `aws ec2` command will work until Floci is running.

```bash
floci start --persist ./floci-data
eval $(floci env)
```

**Verify:**
```bash
echo $AWS_ENDPOINT_URL
```

**Expected output:**
```
http://localhost:4566
```

---

### Step 1 — Create a Key Pair

**Why:** EC2 instances require a key pair for SSH access. Creating one tells AWS which "key" will unlock this server.

```bash
aws ec2 create-key-pair --key-name my-ec2-key
```

**Expected output:**
```json
{
    "KeyFingerprint": "xx:xx:xx:xx:xx:xx:xx:xx",
    "KeyName": "my-ec2-key",
    "KeyPairId": "key-xxxxxxxxxxxxxxxxx",
    "KeyMaterial": "-----BEGIN RSA PRIVATE KEY-----\n..."
}
```

> On real AWS, save the `KeyMaterial` content to a `.pem` file.

**Verify:**
```bash
aws ec2 describe-key-pairs
```

**Expected output:**
```json
{
    "KeyPairs": [
        {
            "KeyPairId": "key-xxxxxxxxxxxxxxxxx",
            "KeyName": "my-ec2-key",
            "KeyFingerprint": "xx:xx:xx:xx:xx:xx"
        }
    ]
}
```

---

### Step 2 — Create a Security Group

**Why:** Without a security group, no traffic can reach your instance. We need to open port 22 for SSH and port 80 for web traffic.

```bash
aws ec2 create-security-group \
  --group-name my-ec2-sg \
  --description "EC2 Security Group for web server"
```

**Expected output:**
```json
{
    "GroupId": "sg-xxxxxxxxxxxxxxxxx"
}
```

**Get the Security Group ID:**
```bash
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?GroupName==`my-ec2-sg`].GroupId' \
  --output text
```

**Expected output:**
```
sg-xxxxxxxxxxxxxxxxx
```

**Open SSH (port 22):**
```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxxxxxxxxxxx \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

**Open HTTP (port 80):**
```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxxxxxxxxxxx \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

**Expected output (for both):**
```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-xxxxxxxxxxxxxxxxx",
            "GroupId": "sg-xxxxxxxxxxxxxxxxx",
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
```

**Verify:**
```bash
aws ec2 describe-security-groups --group-ids sg-xxxxxxxxxxxxxxxxx --output table
```

**Expected output:**
```
-------------------------------------------------------
|              DescribeSecurityGroups                 |
+---------------------+-------------------------------+
|  GroupId            |  GroupName                    |
+---------------------+-------------------------------+
|  sg-xxxxxxxxxx      |  my-ec2-sg                    |
+---------------------+-------------------------------+
```

---

### Step 3 — Launch an EC2 Instance

**Why:** This is the core task — creating a virtual server in the cloud. `run-instances` tells AWS to start a new server with our specified configuration.

```bash
aws ec2 run-instances \
  --image-id ami-12345678 \
  --instance-type t2.micro \
  --key-name my-ec2-key \
  --security-group-ids sg-xxxxxxxxxxxxxxxxx \
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
            "KeyName": "my-ec2-key",
            "SecurityGroups": [
                { "GroupName": "my-ec2-sg", "GroupId": "sg-xxxxxxxxxxxxxxxxx" }
            ],
            "PublicIpAddress": "54.x.x.x",
            "PrivateIpAddress": "172.31.x.x",
            "LaunchTime": "2026-07-01T00:00:00+00:00"
        }
    ]
}
```

> **Note the `InstanceId`:** `i-xxxxxxxxxxxxxxxxx` — you will need this for all following commands.

---

### Step 3a — Launch an Instance with User Data (Startup Script)

**Why:** Having to SSH in and manually install nginx after every launch is slow and error-prone. User Data is a script that runs **automatically the moment the instance first boots**. This is the first step toward DevOps automation.

**Create the startup script:**

```bash
cat > startup.sh << 'EOF'
#!/bin/bash
yum update -y
yum install nginx -y
systemctl start nginx
systemctl enable nginx
EOF
```

**Expected output:**
```
(no output — this is normal, the file was created)
```

**Verify the file:**
```bash
cat startup.sh
```

**Expected output:**
```
#!/bin/bash
yum update -y
yum install nginx -y
systemctl start nginx
systemctl enable nginx
```

**Launch the instance with User Data:**

> **Note:** If you are running this from the `floci/` root folder, use `file://Day4-EC2-floci/startup.sh`. If you are inside the `Day4-EC2-floci/` folder, use `file://startup.sh`.

```bash
aws ec2 run-instances \
  --image-id ami-12345678 \
  --instance-type t2.micro \
  --key-name my-ec2-key \
  --security-group-ids sg-xxxxxxxxxxxxxxxxx \
  --user-data file://Day4-EC2-floci/startup.sh \
  --count 1
```

**Expected output:**
```json
{
    "Instances": [
        {
            "InstanceId": "i-yyyyyyyyyyyyyyyyy",
            "InstanceType": "t2.micro",
            "ImageId": "ami-12345678",
            "State": {
                "Code": 0,
                "Name": "pending"
            },
            "PublicIpAddress": "54.x.x.x"
        }
    ]
}
```

> **Floci vs Real AWS — User Data difference:**
>
> | | Floci (Local Practice) | Real AWS |
> |-|------------------------|----------|
> | `--user-data` flag | ✅ API accepts it, no error | ✅ API accepts it |
> | Does the script run? | ❌ No — no real VM exists | ✅ Yes — runs automatically on first boot |
> | Result | Instance shows `running`, nginx absent | nginx running after 2–3 min, `http://PUBLIC_IP` works |
> | How to check logs | Not applicable | `cat /var/log/cloud-init-output.log` |
>
> **Bottom line:** In Floci we're learning the CLI workflow — getting the command right is the goal. On real AWS, the same command will actually execute the script.

---

### Step 4 — Check Instance Status

**Why:** After launch, an instance goes from `pending` → `running`. On real AWS, you must wait for `running` before SSH will work.

```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType,PublicIpAddress,KeyName]' \
  --output table
```

**Expected output:**
```
---------------------------------------------------------------------------
|                          DescribeInstances                              |
+---------------------+---------+-----------+--------------+--------------+
|  i-xxxxxxxxxxxxxxx  | running | t2.micro  | 54.x.x.x    | my-ec2-key   |
+---------------------+---------+-----------+--------------+--------------+
```

**Check a specific instance:**
```bash
aws ec2 describe-instances --instance-ids i-xxxxxxxxxxxxxxxxx
```

---

### Step 5 — Tag the Instance

**Why:** In production environments with dozens of instances, tags help you identify what each one is for at a glance.

```bash
aws ec2 create-tags \
  --resources i-xxxxxxxxxxxxxxxxx \
  --tags Key=Name,Value=my-web-server Key=Environment,Value=dev
```

**Expected output:**
```
(no output — this is normal, it means the command succeeded)
```

**Verify the tags:**
```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxxxxxxxxxxx \
  --query 'Reservations[*].Instances[*].Tags'
```

**Expected output:**
```json
[
    [
        [
            { "Key": "Name", "Value": "my-web-server" },
            { "Key": "Environment", "Value": "dev" }
        ]
    ]
]
```

---

### Step 5a — Create a Launch Template

**Why:** Writing out every flag (AMI, instance type, key pair, security group) every time you launch an instance is repetitive. A Launch Template saves all of that once. In DevOps, Auto Scaling Groups and CI/CD pipelines rely on these templates.

**Create the template configuration file:**

```bash
cat > template-config.json << 'EOF'
{
  "ImageId": "ami-12345678",
  "InstanceType": "t2.micro",
  "KeyName": "my-ec2-key",
  "SecurityGroupIds": ["sg-xxxxxxxxxxxxxxxxx"],
  "TagSpecifications": [
    {
      "ResourceType": "instance",
      "Tags": [
        {"Key": "Name", "Value": "web-server"},
        {"Key": "Environment", "Value": "dev"}
      ]
    }
  ]
}
EOF
```

**Expected output:**
```
(no output — this is normal, the file was created)
```

**Create the Launch Template:**

> **Remember:** If running from the `floci/` root folder, the path must be `file://Day4-EC2-floci/template-config.json`. If you are inside `Day4-EC2-floci/`, use `file://template-config.json`.

```bash
aws ec2 create-launch-template \
  --launch-template-name my-web-template \
  --version-description "Web server v1" \
  --launch-template-data file://Day4-EC2-floci/template-config.json
```

**Expected output:**
```json
{
    "LaunchTemplate": {
        "LaunchTemplateId": "lt-xxxxxxxxxxxxxxxxx",
        "LaunchTemplateName": "my-web-template",
        "CreateTime": "2026-07-01T00:00:00+00:00",
        "DefaultVersionNumber": 1,
        "LatestVersionNumber": 1
    }
}
```

**Verify that the template data was saved correctly:**

```bash
aws ec2 describe-launch-template-versions \
  --launch-template-name my-web-template \
  --output json
```

**Expected output — `LaunchTemplateData` must contain `ImageId`:**
```json
{
    "LaunchTemplateVersions": [
        {
            "LaunchTemplateName": "my-web-template",
            "LaunchTemplateData": {
                "ImageId": "ami-12345678",
                "InstanceType": "t2.micro",
                "KeyName": "my-ec2-key",
                "SecurityGroupIds": ["sg-xxxxxxxxxxxxxxxxx"]
            }
        }
    ]
}
```

> **Warning:** If the `file://` path is wrong, Floci silently creates the template with empty data — no error is shown. Running `run-instances --launch-template` then fails with `MissingParameter: ImageId`. Always verify template data after creation.

**List all templates:**

```bash
aws ec2 describe-launch-templates --output table
```

**Expected output:**
```
-------------------------------------------------------------
|              DescribeLaunchTemplates                      |
+----------------------+------------------------------------+
|  LaunchTemplateId    |  LaunchTemplateName                |
+----------------------+------------------------------------+
|  lt-xxxxxxxxxx       |  my-web-template                   |
+----------------------+------------------------------------+
```

---

### Step 5b — Launch an Instance from the Template

**Why:** With the template created, launching a new instance now takes one short command — all configuration comes from the template.

```bash
# In Floci, --image-id must be passed explicitly
aws ec2 run-instances \
  --launch-template LaunchTemplateName=my-web-template,Version=1 \
  --image-id ami-12345678 \
  --count 1
```

> **Floci vs Real AWS:** On real AWS, `--image-id` is not needed — it comes from the template automatically. In Floci, the template data is stored correctly but `ImageId` is not pulled when running `run-instances`, so it must be passed explicitly.

**Expected output:**
```json
{
    "Instances": [
        {
            "InstanceId": "i-zzzzzzzzzzzzzzzzz",
            "InstanceType": "t2.micro",
            "ImageId": "ami-12345678",
            "State": { "Name": "pending" },
            "Tags": [{"Key": "Name", "Value": "web-server"}]
        }
    ]
}
```

**Verify the instance is running:**
```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType]' \
  --output table
```

**Delete the template when done:**

```bash
aws ec2 delete-launch-template \
  --launch-template-name my-web-template
```

**Expected output:**
```json
{
    "LaunchTemplate": {
        "LaunchTemplateId": "lt-xxxxxxxxxxxxxxxxx",
        "LaunchTemplateName": "my-web-template"
    }
}
```

---

### Step 6 — Stop and Start the Instance

**Why:** Stopping an instance when not in use saves compute costs. Data is preserved; you pay only for the storage.

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
# Start the instance again
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

> **Stop vs Terminate:**
> - **Stop** → server paused, data preserved, can restart
> - **Terminate** → server destroyed permanently, data gone

---

### Step 7 — Terminate the Instance

**Why:** Good habit to terminate practice instances when done. On real AWS, a running instance charges you even if idle.

```bash
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

**Verify:**
```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' \
  --output table
```

**Expected output:**
```
+---------------------+-------------+
|  i-xxxxxxxxxxxxxxx  | terminated  |
+---------------------+-------------+
```

---

## Part 3 — SSH on Real AWS (Reference)

> These steps do not work in Floci.
> Follow this section when you have a real AWS Free Tier instance running.

---

### After Launching on Real AWS

**1. Fix the `.pem` file permissions (Linux/Mac/Git Bash):**

```bash
chmod 400 my-ec2-key.pem
```

**Why:** If permissions are too open, SSH will refuse to use the key and show `WARNING: UNPROTECTED PRIVATE KEY FILE!`

**2. SSH into the instance:**

```bash
ssh -i my-ec2-key.pem ec2-user@YOUR_PUBLIC_IP
```

**Example:**
```bash
ssh -i my-ec2-key.pem ec2-user@54.123.45.67
```

**Expected output:**
```
The authenticity of host '54.123.45.67' can't be established.
Are you sure you want to continue connecting (yes/no)? yes

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

[ec2-user@ip-172-31-x-x ~]$
```

When you see `[ec2-user@ip-...]$` you are inside the cloud server.

**3. Run a few commands inside the server:**

```bash
whoami
```
```
ec2-user
```

```bash
uptime
```
```
 00:00:00 up 0 min,  1 user,  load average: 0.00, 0.00, 0.00
```

```bash
free -h
```
```
              total        used        free
Mem:           983M         68M        915M
```

**4. Install nginx web server:**

```bash
sudo yum install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

**5. View it in a browser:**

```
http://YOUR_PUBLIC_IP
```

If you see the nginx default page, your web server is running successfully.

**6. Exit the server:**

```bash
exit
```

---

## Common Mistakes and Fixes

| Mistake | What Happens | Fix |
|---------|-------------|-----|
| SSH port 22 blocked | `Connection timed out` | Add port 22 to security group inbound rules |
| Wrong `.pem` file | `Permission denied (publickey)` | Use the correct key file that matches the key pair |
| File permissions too open | `WARNING: UNPROTECTED PRIVATE KEY FILE` | Run `chmod 400 my-key.pem` |
| Instance is stopped | SSH doesn't connect | Start the instance first |
| Wrong username | `Permission denied` | Amazon Linux: `ec2-user`, Ubuntu: `ubuntu` |
| Public IP changed | SSH to old IP fails | Stop/Start changes the IP — get the new one |

---

## Quick Reference — EC2 CLI Cheat Sheet

| Command | What It Does |
|---------|-------------|
| `aws ec2 create-key-pair --key-name <n>` | Create a key pair |
| `aws ec2 describe-key-pairs` | List all key pairs |
| `aws ec2 create-security-group --group-name <n> --description <d>` | Create security group |
| `aws ec2 authorize-security-group-ingress --group-name <g> --protocol tcp --port <p> --cidr <c>` | Open a port |
| `aws ec2 run-instances --image-id <ami> --instance-type <t> --key-name <k> --security-group-ids <sg-id>` | Launch an instance |
| `aws ec2 describe-instances` | List all instances |
| `aws ec2 describe-instances --query '...' --output table` | Table view |
| `aws ec2 create-tags --resources <id> --tags Key=Name,Value=<v>` | Add a tag |
| `aws ec2 stop-instances --instance-ids <id>` | Stop an instance |
| `aws ec2 start-instances --instance-ids <id>` | Start an instance |
| `aws ec2 terminate-instances --instance-ids <id>` | Terminate an instance |
| `aws ec2 run-instances ... --user-data file://script.sh` | Launch with startup script |
| `aws ec2 create-launch-template --launch-template-name <n> --launch-template-data file://config.json` | Create a template |
| `aws ec2 run-instances --launch-template LaunchTemplateName=<n>,Version=1` | Launch from template |
| `aws ec2 describe-launch-templates` | List all templates |
| `aws ec2 delete-launch-template --launch-template-name <n>` | Delete a template |
| `ssh -i key.pem ec2-user@IP` | SSH into an instance (Real AWS) |

---

## What You Built Today

```
EC2 Setup (Floci CLI)
├── Key Pair         → my-ec2-key
├── Security Group   → my-ec2-sg (port 22, 80 open)
├── EC2 Instance     → t2.micro (basic launch)
├── EC2 Instance     → t2.micro (with User Data — nginx auto-install)
│   └── startup.sh   → yum install nginx + systemctl start
├── Launch Template  → my-web-template
│   ├── AMI, type, key, sg all saved
│   └── One command to launch any instance
├── Instance operations
│   ├── Tag: Name=my-web-server, Environment=dev
│   ├── Stop  → data preserved, compute charge stops
│   └── Terminate → permanently deleted
└── Real AWS Reference
    ├── chmod 400 → fix .pem permissions
    ├── ssh -i key.pem ec2-user@IP
    └── nginx install + browser access
```

---

## Homework

1. Create two separate security groups — one for a web server (port 80, 443), one for a database (port 3306)
2. Write a User Data script that installs nginx and creates a custom `index.html` with the text "Hello from EC2!"
3. Create a Launch Template that includes your User Data script, then launch an instance from it

---

## Resources

- Floci Official: [https://floci.io](https://floci.io)
- Floci AWS Services: [https://floci.io/aws](https://floci.io/aws)
- AWS EC2 CLI Reference: [https://docs.aws.amazon.com/cli/latest/reference/ec2/](https://docs.aws.amazon.com/cli/latest/reference/ec2/)
- AWS EC2 User Guide: [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/)
