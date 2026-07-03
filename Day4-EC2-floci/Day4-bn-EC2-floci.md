# দিন ৪: Amazon EC2 — Elastic Compute Cloud
### Floci দিয়ে হাতে-কলমে শেখো (Git Bash)

> সব CLI কমান্ড **Git Bash**-এ রান করতে হবে।
> কমান্ড রান করার আগে **Docker Desktop** অবশ্যই চালু থাকতে হবে।

---

## আজকে যা যা শিখবে

- EC2 কী এবং কেন cloud-এ virtual server দরকার
- Key Pair কী এবং SSH কীভাবে কাজ করে
- Security Group — cloud firewall
- EC2 Instance Types এবং Pricing Models
- Floci দিয়ে CLI-তে EC2 instance launch, stop, start, terminate
- User Data দিয়ে instance launch হওয়ার সময়ই web server install করা
- Launch Template — configuration save করে বারবার ব্যবহার করা
- Real AWS-এ SSH করে ভেতরে ঢোকার পদ্ধতি
- Common mistakes এবং সমাধান

---

## পার্ট ১ — তত্ত্ব (Theory)

### EC2 কী?

**EC2 = Elastic Compute Cloud**

Amazon EC2 হলো AWS-এর সেই সার্ভিস যেটা দিয়ে cloud-এ virtual server তৈরি করা যায়।

"Elastic" মানে — যখন দরকার বড় করো, যখন দরকার নেই ছোট করো বা বন্ধ করো।

EC2 দিয়ে চালানো যায়:
- ওয়েবসাইট এবং web application
- API server
- Docker container
- Machine learning model
- Jenkins, Terraform, Kubernetes node

---

### EC2 কেন দরকার? (DevOps দৃষ্টিকোণ)

| নিজের server | AWS EC2 |
|-------------|---------|
| একবার কিনলে আটকে গেলে | যখন লাগবে তখন তৈরি, না লাগলে বন্ধ |
| Hardware নষ্ট হলে সমস্যা | AWS নিজে maintain করে |
| Capacity বাড়াতে দিন লাগে | ৫ মিনিটে নতুন server |
| অফিস বন্ধ থাকলে চলে না | ২৪/৭ available |

**DevOps-এ EC2 ব্যবহার:**
- Jenkins server চালানো
- Production application deploy করা
- Docker/Kubernetes node
- Database server (RDS-এর বিকল্প)

---

### Key Pair কী?

**Key Pair** হলো EC2 instance-এ SSH করে ঢোকার চাবি।

দুটো অংশ:
- **Public Key** → AWS-এ জমা থাকে (EC2 instance-এ রাখা হয়)
- **Private Key (.pem file)** → তোমার computer-এ থাকে

কীভাবে কাজ করে:
```
তুমি                    EC2 Instance
  │                          │
  │──ssh -i my-key.pem──▶    │
  │   (private key দিয়ে)     │
  │                          │── public key দিয়ে match করে
  │◀─── connection open ───  │
```

> **সতর্কতা:** `.pem` file হারিয়ে গেলে আর SSH করা যাবে না। নতুন key pair তৈরি করতে হবে।

---

### Security Group কী?

**Security Group** হলো EC2-এর **virtual firewall**।

এটা নিয়ন্ত্রণ করে:
- **Inbound rules** → বাইরে থেকে কোন traffic আসতে পারবে
- **Outbound rules** → ভেতর থেকে কোথায় traffic যেতে পারবে

সাধারণ inbound rules:

| Type | Protocol | Port | Purpose | Source |
|------|----------|------|---------|--------|
| SSH | TCP | 22 | Server-এ ঢোকা | My IP |
| HTTP | TCP | 80 | Website | 0.0.0.0/0 |
| HTTPS | TCP | 443 | Secure website | 0.0.0.0/0 |
| Custom TCP | TCP | 8080 | Jenkins | My IP |

> **নিয়ম:** SSH port (22) সবসময় `My IP` দাও, `0.0.0.0/0` (সবার জন্য) দিলে hacker ঢুকতে পারে।

---

### EC2 Instance Types

Instance type মানে server-এর hardware configuration।

| Type | vCPU | RAM | ব্যবহার |
|------|------|-----|---------|
| `t2.micro` | 1 | 1 GB | Free tier, testing |
| `t2.small` | 1 | 2 GB | Light apps |
| `t2.large` | 2 | 8 GB | Medium apps |
| `m5.large` | 2 | 8 GB | Balanced, production |
| `c5.large` | 2 | 4 GB | CPU intensive (Jenkins, compile) |
| `r5.large` | 2 | 16 GB | Memory intensive (DB, cache) |

**নামের pattern বোঝো:**
```
t  2  .  micro
│  │      └── size: nano, micro, small, medium, large, xlarge
│  └───────── generation: 2, 3, 4, 5...
└──────────── family: t=general, m=balanced, c=compute, r=memory
```

---

### EC2 Pricing Models

#### ১. On-Demand
- প্রতি সেকেন্ড বা ঘণ্টায় pay করো
- কোনো commitment নেই
- **কখন:** Testing, unpredictable workload

#### ২. Reserved Instances
- ১ বছর বা ৩ বছরের commitment
- On-demand-এর চেয়ে ৭২% পর্যন্ত সাশ্রয়
- **কখন:** Production server যেটা সবসময় চলে

#### ৩. Spot Instances
- AWS-এর unused capacity — সবচেয়ে সস্তা (৯০% পর্যন্ত সাশ্রয়)
- AWS যেকোনো সময় বন্ধ করে দিতে পারে
- **কখন:** Batch processing, data analysis, non-critical কাজ

#### ৪. Free Tier
- `t2.micro` — প্রতি মাসে ৭৫০ ঘণ্টা ফ্রি
- প্রথম ১২ মাস
- **কখন:** শেখার জন্য

---

### AMI কী? (Amazon Machine Image)

AMI হলো EC2-এর **blueprint** — কোন operating system এবং software দিয়ে server শুরু হবে।

সাধারণ AMI:
```
Amazon Linux 2          ← AWS-এর নিজস্ব, সবচেয়ে popular
Ubuntu 22.04 LTS        ← Developers-এর পছন্দ
Windows Server 2022     ← Windows workload
Red Hat Enterprise      ← Enterprise production
```

---

### User Data কী?

**User Data** হলো একটি shell script যেটা EC2 instance **প্রথমবার চালু হওয়ার সময় automatically চলে**।

```
Instance launch
     │
     ▼
OS boot হয়
     │
     ▼
User Data script চলে  ← এখানে nginx/docker/app install হয়
     │
     ▼
Instance ready
```

**কখন ব্যবহার হয়:**
- Web server (nginx, apache) automatically install করতে
- Application deploy করতে
- Configuration file তৈরি করতে
- Docker চালু করতে

> **Floci note:** Floci API-তে `--user-data` accept করে কিন্তু script execute হয় না — কারণ real VM নেই। Real AWS-এ এটা production-এ ব্যবহার হয়।

---

### Launch Template কী?

**Launch Template** হলো EC2-এর **saved configuration**।

একবার template তৈরি করো, যেখানে AMI, instance type, key pair, security group, user data সব save থাকে। পরে একটি ছোট কমান্ড দিলেই সেই configuration-এ instance তৈরি হয়।

**কখন লাগে:**
- Auto Scaling Group (একই configuration-এ অনেক instance)
- CI/CD pipeline-এ automated deployment
- Team-এ সবাই যেন একই configuration ব্যবহার করে

---

### Best Practices

| নিয়ম | কেন |
|------|-----|
| SSH-এ `My IP` দাও, `0.0.0.0/0` না | Security — সবাই access করতে পারবে না |
| `.pem` file secure রাখো (`chmod 400`) | File permission ঠিক না থাকলে SSH কাজ করে না |
| Production-এ Reserved Instance ব্যবহার করো | খরচ কমে |
| Instance বন্ধ থাকলে Stop করো | Stop করলে compute charge বন্ধ হয় |
| Terminate করার আগে data backup নাও | Terminate করলে সব মুছে যায় |
| Tag দাও প্রতিটি instance-এ | কোনটা কীসের জন্য সহজে বোঝা যায় |
| User Data দিয়ে setup automate করো | Manual SSH করে install করা ভুলপ্রবণ |
| Launch Template ব্যবহার করো | Configuration consistent থাকে |

---

## পার্ট ২ — Floci দিয়ে হাতে-কলমে (CLI)

> **মনে রেখো:** Floci-এ EC2 **API simulation** — real VM চলে না।
> তুমি instance তৈরি করতে পারবে, কিন্তু SSH করা যাবে না।
> উদ্দেশ্য: CLI command এবং workflow শেখা।

---

### ধাপ ০ — Floci চালু করো

**কেন করছি?**
Floci না চাললে কোনো `aws ec2` কমান্ড কাজ করবে না।

```bash
floci start --persist ./floci-data
eval $(floci env)
```

**যাচাই করো:**
```bash
echo $AWS_ENDPOINT_URL
```

**প্রত্যাশিত output:**
```
http://localhost:4566
```

---

### ধাপ ১ — Key Pair তৈরি করো

**কেন করছি?**
EC2 instance-এ SSH করতে হলে key pair দরকার। Key pair তৈরি করা মানে AWS-কে বলছি "এই চাবি ব্যবহার করে আমার server-এ ঢোকা যাবে।"

```bash
aws ec2 create-key-pair --key-name my-ec2-key
```

**প্রত্যাশিত output:**
```json
{
    "KeyFingerprint": "xx:xx:xx:xx:xx:xx:xx:xx",
    "KeyName": "my-ec2-key",
    "KeyPairId": "key-xxxxxxxxxxxxxxxxx",
    "KeyMaterial": "-----BEGIN RSA PRIVATE KEY-----\n..."
}
```

> Real AWS-এ `KeyMaterial`-এর content `.pem` file হিসেবে save করতে হয়।

**যাচাই করো:**
```bash
aws ec2 describe-key-pairs
```

**প্রত্যাশিত output:**
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

### ধাপ ২ — Security Group তৈরি করো

**কেন করছি?**
Security group ছাড়া instance launch হবে কিন্তু কোনো traffic ঢুকতে পারবে না। SSH করতে port 22 এবং website দেখতে port 80 খুলতে হবে।

```bash
aws ec2 create-security-group \
  --group-name my-ec2-sg \
  --description "EC2 Security Group for web server"
```

**প্রত্যাশিত output:**
```json
{
    "GroupId": "sg-xxxxxxxxxxxxxxxxx"
}
```

**Security Group-এর ID নাও:**
```bash
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?GroupName==`my-ec2-sg`].GroupId' \
  --output text
```

**প্রত্যাশিত output:**
```
sg-xxxxxxxxxxxxxxxxx
```

**SSH (port 22) খুলো:**
```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxxxxxxxxxxx \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

**HTTP (port 80) খুলো:**
```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxxxxxxxxxxx \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

**প্রত্যাশিত output (দুটোর জন্যই):**
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

**যাচাই করো:**
```bash
aws ec2 describe-security-groups --group-ids sg-xxxxxxxxxxxxxxxxx --output table
```

**প্রত্যাশিত output:**
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

### ধাপ ৩ — EC2 Instance Launch করো

**কেন করছি?**
এটাই মূল কাজ — cloud-এ একটি virtual server তৈরি করা। `run-instances` কমান্ড দিলে AWS একটি নতুন server চালু করে।

```bash
aws ec2 run-instances \
  --image-id ami-12345678 \
  --instance-type t2.micro \
  --key-name my-ec2-key \
  --security-group-ids sg-xxxxxxxxxxxxxxxxx \
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

> **`InstanceId` note করো:** `i-xxxxxxxxxxxxxxxxx` — পরের সব কমান্ডে লাগবে।

---

### ধাপ ৩ক — User Data দিয়ে Instance Launch করো (Startup Script)

**কেন করছি?**
প্রতিবার instance launch করার পর SSH করে nginx install করা সময়সাপেক্ষ এবং ভুলপ্রবণ। User Data হলো এমন একটি script যেটা instance **প্রথমবার চালু হওয়ার সময়ই automatically চলে**। এটা DevOps automation-এর প্রথম ধাপ।

**প্রথমে startup script তৈরি করো:**

```bash
cat > startup.sh << 'EOF'
#!/bin/bash
yum update -y
yum install nginx -y
systemctl start nginx
systemctl enable nginx
EOF
```

**প্রত্যাশিত output:**
```
(কোনো output আসবে না — এটা স্বাভাবিক, মানে file তৈরি হয়েছে)
```

**File তৈরি হয়েছে কিনা যাচাই করো:**
```bash
cat startup.sh
```

**প্রত্যাশিত output:**
```
#!/bin/bash
yum update -y
yum install nginx -y
systemctl start nginx
systemctl enable nginx
```

**User Data সহ instance launch করো:**

> **মনে রেখো:** `floci/` ফোল্ডার থেকে কমান্ড দিলে path `file://Day4-EC2-floci/startup.sh` হবে। `Day4-EC2-floci/` ফোল্ডারের ভেতরে থাকলে `file://startup.sh` হবে।

```bash
aws ec2 run-instances \
  --image-id ami-12345678 \
  --instance-type t2.micro \
  --key-name my-ec2-key \
  --security-group-ids sg-xxxxxxxxxxxxxxxxx \
  --user-data file://Day4-EC2-floci/startup.sh \
  --count 1
```

**প্রত্যাশিত output:**
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

> **Floci vs Real AWS — User Data-তে পার্থক্য:**
>
> | | Floci (Local Practice) | Real AWS |
> |-|------------------------|----------|
> | `--user-data` flag | ✅ API accept করে, কোনো error নেই | ✅ API accept করে |
> | Script execute হয়? | ❌ না — real VM নেই | ✅ হ্যাঁ — first boot-এ automatically চলে |
> | Result | Instance `running` দেখাবে, nginx নেই | ২-৩ মিনিট পর nginx চালু, `http://PUBLIC_IP` কাজ করে |
> | Log দেখার উপায় | প্রযোজ্য নয় | `cat /var/log/cloud-init-output.log` |
>
> **সারকথা:** Floci-তে আমরা CLI workflow শিখছি — command ঠিকমতো দিতে পারলেই হলো। Real AWS-এ same command দিলে script সত্যিই চলবে।

---

### ধাপ ৪ — Instance-এর অবস্থা দেখো

**কেন করছি?**
Instance তৈরি হওয়ার পর সে `pending` → `running` state-এ যায়। Real AWS-এ SSH করার আগে `running` হওয়া পর্যন্ত অপেক্ষা করতে হয়।

```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType,PublicIpAddress,KeyName]' \
  --output table
```

**প্রত্যাশিত output:**
```
---------------------------------------------------------------------------
|                          DescribeInstances                              |
+---------------------+---------+-----------+--------------+--------------+
|  i-xxxxxxxxxxxxxxx  | running | t2.micro  | 54.x.x.x    | my-ec2-key   |
+---------------------+---------+-----------+--------------+--------------+
```

**Specific instance দেখতে:**
```bash
aws ec2 describe-instances --instance-ids i-xxxxxxxxxxxxxxxxx
```

---

### ধাপ ৫ — Instance-এ Tag দাও

**কেন করছি?**
Real production-এ অনেক instance থাকে। Tag দিলে কোনটা কীসের জন্য সহজে চেনা যায়।

```bash
aws ec2 create-tags \
  --resources i-xxxxxxxxxxxxxxxxx \
  --tags Key=Name,Value=my-web-server Key=Environment,Value=dev
```

**প্রত্যাশিত output:**
```
(কোনো output আসবে না — এটা স্বাভাবিক, মানে সফল হয়েছে)
```

**Tag দেখো:**
```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxxxxxxxxxxx \
  --query 'Reservations[*].Instances[*].Tags'
```

**প্রত্যাশিত output:**
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

### ধাপ ৫ক — Launch Template তৈরি করো

**কেন করছি?**
প্রতিবার instance তৈরিতে একই AMI, instance type, key pair, security group লেখা repetitive। Launch Template-এ এই configuration একবার save করো — পরে একটি ছোট কমান্ডেই instance তৈরি হয়। DevOps-এ Auto Scaling, CI/CD pipeline-এ এটাই ব্যবহার হয়।

**Template configuration file তৈরি করো:**

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

**প্রত্যাশিত output:**
```
(কোনো output আসবে না — এটা স্বাভাবিক, মানে file তৈরি হয়েছে)
```

**Launch Template তৈরি করো:**

> **মনে রেখো:** `floci/` ফোল্ডার থেকে কমান্ড দিলে path `file://Day4-EC2-floci/template-config.json` হবে। `Day4-EC2-floci/` ফোল্ডারের ভেতরে থাকলে `file://template-config.json` হবে।

```bash
aws ec2 create-launch-template \
  --launch-template-name my-web-template \
  --version-description "Web server v1" \
  --launch-template-data file://Day4-EC2-floci/template-config.json
```

**প্রত্যাশিত output:**
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

**Template-এর data ঠিকমতো save হয়েছে কিনা যাচাই করো:**

```bash
aws ec2 describe-launch-template-versions \
  --launch-template-name my-web-template \
  --output json
```

**প্রত্যাশিত output-এ `LaunchTemplateData`-তে `ImageId` থাকতে হবে:**
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

> **সতর্কতা:** `file://` path ভুল হলে Floci error না দিয়ে empty data দিয়েই template বানিয়ে দেয়। তখন `run-instances --launch-template` করলে `MissingParameter: ImageId` error আসে। Template data verify করা তাই জরুরি।

**সব template দেখো:**

```bash
aws ec2 describe-launch-templates --output table
```

**প্রত্যাশিত output:**
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

### ধাপ ৫খ — Launch Template থেকে Instance তৈরি করো

**কেন করছি?**
Template তৈরির পর এখন একটি ছোট কমান্ডেই instance launch হবে — সব configuration template থেকে আসবে, আলাদা করে লিখতে হবে না।

```bash
# Floci-তে --image-id explicitly দিতে হয়
aws ec2 run-instances \
  --launch-template LaunchTemplateName=my-web-template,Version=1 \
  --image-id ami-12345678 \
  --count 1
```

> **Floci vs Real AWS:** Real AWS-এ `--image-id` লাগে না — template থেকে automatically আসে। Floci-তে template data store হলেও `run-instances`-এর সময় সেটা থেকে `ImageId` pull করে না, তাই explicitly দিতে হয়।

**প্রত্যাশিত output:**
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

**যাচাই করো — instance চলছে কিনা:**
```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType]' \
  --output table
```

**Template মুছে দাও (cleanup):**

```bash
aws ec2 delete-launch-template \
  --launch-template-name my-web-template
```

**প্রত্যাশিত output:**
```json
{
    "LaunchTemplate": {
        "LaunchTemplateId": "lt-xxxxxxxxxxxxxxxxx",
        "LaunchTemplateName": "my-web-template"
    }
}
```

---

### ধাপ ৬ — Instance Stop ও Start করো

**কেন করছি?**
রাতে বা weekend-এ server বন্ধ রাখলে compute cost বাঁচে। Stop করলে data থাকে কিন্তু charge বন্ধ হয়।

```bash
# Instance বন্ধ করো
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
# Instance আবার চালু করো
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

> **Stop vs Terminate:**
> - **Stop** → server বন্ধ, data থাকে, আবার চালু করা যায়
> - **Terminate** → server সম্পূর্ণ মুছে যায়, ফিরিয়ে আনা যায় না

---

### ধাপ ৭ — Instance Terminate করো

**কেন করছি?**
Practice শেষে instance মুছে রাখা ভালো অভ্যাস। Real AWS-এ terminate না করলে charge চলতে থাকে।

```bash
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

**যাচাই করো:**
```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' \
  --output table
```

**প্রত্যাশিত output:**
```
+---------------------+-------------+
|  i-xxxxxxxxxxxxxxx  | terminated  |
+---------------------+-------------+
```

---

## পার্ট ৩ — Real AWS-এ SSH করা (Reference)

> এই section Floci-তে কাজ করবে না।
> Real AWS Free Tier account-এ instance launch করলে এই steps follow করো।

---

### Real AWS-এ Instance Launch করার পর

**১. `.pem` file-এর permission ঠিক করো (Linux/Mac/Git Bash):**

```bash
chmod 400 my-ec2-key.pem
```

**কেন:** Permission ঠিক না থাকলে SSH বলবে `WARNING: UNPROTECTED PRIVATE KEY FILE!` এবং connect হবে না।

**২. SSH দিয়ে ঢোকো:**

```bash
ssh -i my-ec2-key.pem ec2-user@YOUR_PUBLIC_IP
```

**উদাহরণ:**
```bash
ssh -i my-ec2-key.pem ec2-user@54.123.45.67
```

**প্রত্যাশিত output:**
```
The authenticity of host '54.123.45.67' can't be established.
Are you sure you want to continue connecting (yes/no)? yes

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

[ec2-user@ip-172-31-x-x ~]$
```

`[ec2-user@ip-...]$` দেখলে বুঝবে তুমি এখন cloud server-এর ভেতরে।

**৩. Server-এ কিছু কমান্ড চালাও:**

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

**৪. Nginx web server install করো (User Data ছাড়া manually):**

```bash
sudo yum install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

**৫. Browser-এ দেখো:**

```
http://YOUR_PUBLIC_IP
```

Nginx default page দেখালে web server সফলভাবে চলছে।

**৬. Server থেকে বের হয়ে আসো:**

```bash
exit
```

---

## সাধারণ Mistake এবং সমাধান

| Mistake | কী হয় | সমাধান |
|---------|--------|--------|
| SSH port 22 বন্ধ | `Connection timed out` | Security group-এ port 22 খুলো |
| ভুল `.pem` file | `Permission denied (publickey)` | সঠিক key file ব্যবহার করো |
| File permission ঠিক নেই | `WARNING: UNPROTECTED PRIVATE KEY FILE` | `chmod 400 my-key.pem` |
| Instance stop করা | SSH connect হয় না | Instance start করো |
| ভুল username | `Permission denied` | Amazon Linux: `ec2-user`, Ubuntu: `ubuntu` |
| Public IP পরিবর্তন | SSH কাজ করে না | Stop/Start করলে IP পরিবর্তন হয় — নতুন IP নাও |
| User Data script কাজ করেনি | nginx নেই | `cat /var/log/cloud-init-output.log` দেখো |

---

## দ্রুত তথ্যসূত্র — EC2 CLI চিট শিট

| কমান্ড | কী করে |
|--------|--------|
| `aws ec2 create-key-pair --key-name <n>` | Key pair তৈরি |
| `aws ec2 describe-key-pairs` | সব key pair দেখো |
| `aws ec2 create-security-group --group-name <n> --description <d>` | Security group তৈরি |
| `aws ec2 authorize-security-group-ingress --group-name <g> --protocol tcp --port <p> --cidr <c>` | Port খুলো |
| `aws ec2 run-instances --image-id <ami> --instance-type <t> --key-name <k> --security-group-ids <sg-id>` | Instance launch |
| `aws ec2 run-instances ... --user-data file://script.sh` | User Data সহ launch |
| `aws ec2 create-launch-template --launch-template-name <n> --launch-template-data file://config.json` | Template তৈরি |
| `aws ec2 run-instances --launch-template LaunchTemplateName=<n>,Version=1` | Template থেকে launch |
| `aws ec2 describe-launch-templates` | সব template দেখো |
| `aws ec2 delete-launch-template --launch-template-name <n>` | Template মুছো |
| `aws ec2 describe-instances` | সব instance দেখো |
| `aws ec2 describe-instances --query '...' --output table` | Table format-এ দেখো |
| `aws ec2 create-tags --resources <id> --tags Key=Name,Value=<v>` | Tag দাও |
| `aws ec2 stop-instances --instance-ids <id>` | Instance বন্ধ করো |
| `aws ec2 start-instances --instance-ids <id>` | Instance চালু করো |
| `aws ec2 terminate-instances --instance-ids <id>` | Instance মুছো |
| `ssh -i key.pem ec2-user@IP` | SSH দিয়ে ঢোকো (Real AWS) |

---

## আজকে যা তৈরি করলে

```
EC2 Setup (Floci CLI)
├── Key Pair         → my-ec2-key
├── Security Group   → my-ec2-sg (port 22, 80 open)
├── EC2 Instance     → t2.micro (basic launch)
├── EC2 Instance     → t2.micro (User Data সহ — nginx auto-install)
│   └── startup.sh   → yum install nginx + systemctl start
├── Launch Template  → my-web-template
│   ├── AMI, type, key, sg সব save করা
│   └── একটি কমান্ডে instance তৈরি
├── Instance operations
│   ├── Tag: Name=my-web-server, Environment=dev
│   ├── Stop  → data থাকে, charge বন্ধ
│   └── Terminate → সব মুছে যায়
└── Real AWS Reference
    ├── chmod 400 → .pem permission
    ├── ssh -i key.pem ec2-user@IP
    └── nginx install + browser access
```

---

## বাড়ির কাজ

১. দুটো আলাদা security group তৈরি করো — একটা web server-এর জন্য (port 80, 443), একটা database-এর জন্য (port 3306)
২. User Data script-এ nginx-এর পাশাপাশি একটি custom `index.html` তৈরি করো যাতে "Hello from EC2!" লেখা থাকে
৩. Launch Template-এ User Data যোগ করো এবং সেই template থেকে instance launch করো

---

## রিসোর্স

- Floci অফিসিয়াল: [https://floci.io](https://floci.io)
- Floci AWS সার্ভিস তালিকা: [https://floci.io/aws](https://floci.io/aws)
- AWS EC2 CLI Reference: [https://docs.aws.amazon.com/cli/latest/reference/ec2/](https://docs.aws.amazon.com/cli/latest/reference/ec2/)
- AWS EC2 User Guide: [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/)
