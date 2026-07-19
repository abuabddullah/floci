# Day 9: RDS — AWS Relational Database Service

### Floci দিয়ে হাতে-কলমে শেখো (Git Bash)

> সব কমান্ড **Git Bash**-এ রান করতে হবে।
> কমান্ড রান করার আগে **Docker Desktop** অবশ্যই চালু থাকতে হবে।

📌 **যোগাযোগ / সোশ্যাল মিডিয়া:**
[LinkedIn](https://www.linkedin.com/in/asifaowadud) · [YouTube](https://www.youtube.com/@OOAAOW?sub_confirmation=1) · [Telegram](https://t.me/ooaaow) · [Web Lab](https://oao-devops-lab.vercel.app/) · [Facebook](https://www.facebook.com/OOAAOW/)

---

## পার্ট ১ — তত্ত্ব

### RDS কী?

> **AWS RDS (Relational Database Service)** হলো AWS-এর fully managed relational database service — যেখানে AWS নিজে database server চালু করে, configure করে, backup নেয়, patch করে, এবং monitor করে।

তুমি শুধু schema এবং data নিয়ে কাজ করো — server management AWS-এর দায়িত্ব।

| RDS-এ কী পাও      | বিস্তারিত                                            |
| ----------------- | ---------------------------------------------------- |
| Database Engine   | PostgreSQL, MySQL, MariaDB, Oracle, SQL Server       |
| Automated Backup  | প্রতিদিন snapshot + transaction log                  |
| Patch Management  | AWS নিজে OS এবং database patch করে                   |
| High Availability | Multi-AZ: standby replica automatically failover করে |
| Scalability       | Storage এবং compute আলাদাভাবে scale করা যায়         |
| Monitoring        | CloudWatch metrics built-in                          |

---

### DevOps-এ RDS কেন দরকার?

| সমস্যা (RDS ছাড়া)                          | সমাধান (RDS দিয়ে)                             |
| ------------------------------------------- | ---------------------------------------------- |
| Database server crash হলে data হারিয়ে যায় | Multi-AZ: standby automatically active হয়     |
| Backup নিতে ভুলে গেলে disaster              | Automated daily backup + point-in-time restore |
| Security patch দিতে দেরি হলে vulnerability  | AWS নিজে patch করে                             |
| আলাদা DBA hire করতে হয়                     | AWS managed — DBA লাগে না                      |
| Scale করতে downtime লাগে                    | একটা change-এ storage বাড়ানো যায়             |

**Real-world context:** OOAAOWMetrics.Web একটা Next.js analytics dashboard। এটা PostgreSQL database ব্যবহার করে Prisma ORM দিয়ে। Production-এ এই database-টা RDS PostgreSQL-এ থাকবে।

---

### RDS-এর মূল উপাদান

| উপাদান              | কাজ                                              | উদাহরণ                                      |
| ------------------- | ------------------------------------------------ | ------------------------------------------- |
| **DB Instance**     | Database server যেটা আসলে চলে                    | `db.t3.micro` PostgreSQL 16                 |
| **DB Subnet Group** | কোন VPC subnet-এ RDS থাকবে সেটা define করে       | private-subnet-1a, private-subnet-1b        |
| **Security Group**  | Port 5432 কে allow করবে সেটা নিয়ন্ত্রণ করে      | EC2 থেকে port 5432 allow                    |
| **Parameter Group** | PostgreSQL configuration (max_connections, etc.) | default.postgres16                          |
| **Snapshot**        | Point-in-time backup                             | রোজ রাত ৩টায় automated snapshot            |
| **Endpoint**        | Application connect করার address                 | `mydb.abc.us-east-1.rds.amazonaws.com:5432` |

---

### Supported Database Engines

| Engine         | RDS Name       | কখন ব্যবহার করবে                            |
| -------------- | -------------- | ------------------------------------------- |
| **PostgreSQL** | `postgres`     | Modern app, JSON support, open source পছন্দ |
| **MySQL**      | `mysql`        | Traditional web app, WordPress              |
| **MariaDB**    | `mariadb`      | MySQL-compatible, community fork            |
| **Oracle**     | `oracle-ee`    | Enterprise legacy system                    |
| **SQL Server** | `sqlserver-se` | .NET application, Microsoft stack           |

**আমাদের choice: PostgreSQL** — কারণ OOAAOWMetrics.Web-এ Prisma + PostgreSQL ব্যবহার হচ্ছে।

---

### OOAAOWMetrics.Web — Real Project Context

OOAAOWMetrics.Web হলো একটা SaaS analytics dashboard যেটা Google Analytics 4 data দেখায়। এর Prisma schema-তে এই tables আছে:

```mermaid
graph LR
    TENANT[Tenant] --> USER[User]
    TENANT --> PRODUCT[Product]
    PRODUCT --> GA4[GA4Credential]
    USER --> ACCOUNT[Account]
    USER --> SESSION[Session]
```

| Table               | কী রাখে                                  |
| ------------------- | ---------------------------------------- |
| `Tenant`            | Workspace বা organization (multi-tenant) |
| `User`              | Login user — role: OWNER, ADMIN, MEMBER  |
| `Product`           | Analytics product (Web, Android, iOS)    |
| `GA4Credential`     | Google Analytics OAuth token             |
| `Account`           | NextAuth OAuth account link              |
| `Session`           | Login session                            |
| `VerificationToken` | Email verification token                 |

এই tables গুলো আজকে আমরা RDS PostgreSQL-এ তৈরি করব — `prisma db push` দিয়ে।

---

### Multi-AZ Architecture

```mermaid
graph TB
    APP[Application - EC2] --> PRIMARY[RDS Primary - AZ 1a]
    PRIMARY -->|sync| STANDBY[RDS Standby - AZ 1b]
    PRIMARY -->|failover on crash| STANDBY
    STANDBY -->|becomes new primary| APP
```

**কীভাবে কাজ করে:**

- Primary DB-তে write হলে Standby automatically sync হয়
- Primary crash করলে AWS automatically Standby-কে Primary বানিয়ে দেয়
- Application endpoint একই থাকে — zero manual intervention

**Free Tier-এ:** Multi-AZ disabled (single AZ) — cost বাঁচাতে।

---

### RDS vs Self-managed PostgreSQL on EC2

| বিষয়           | RDS PostgreSQL       | EC2-তে PostgreSQL             |
| --------------- | -------------------- | ----------------------------- |
| Setup           | ১ command            | SSH → install → configure     |
| Backup          | Automatic            | Manual (cron job লিখতে হবে)   |
| Patch           | AWS করে              | তুমি করবে                     |
| Failover        | Automatic (Multi-AZ) | Manual                        |
| Monitoring      | CloudWatch built-in  | নিজে setup করতে হবে           |
| Cost (t3.micro) | ~$15-25 per month    | ~$8-10 per month (EC2 only)   |
| কখন ব্যবহার     | Production           | Learning বা cost optimization |

---

### Floci RDS Support

| Command / Feature               | Floci | বিস্তারিত                                          |
| ------------------------------- | ----- | -------------------------------------------------- |
| `aws rds create-db-instance`    | ✅    | Real PostgreSQL engine start হয়                   |
| `aws rds describe-db-instances` | ✅    | Endpoint এবং port পাওয়া যায়                      |
| `aws rds delete-db-instance`    | ✅    | Instance terminate হয়                             |
| `psql` connection               | ✅    | Real database — actual SQL run করা যায়            |
| `prisma db push`                | ✅    | Floci RDS-এ Prisma schema migrate করা যায়         |
| Snapshots                       | ⚠️    | PostgreSQL-এ কাজ করে — user testing-এ confirm করো  |
| Multi-AZ                        | ⚠️    | Config accept হয় কিন্তু actual standby run হয় না |
| EC2 install (Part 4)            | ❌    | Floci real VM চালায় না                            |

> ℹ️ **Important:** Floci-তে RDS একটা **real PostgreSQL database** চালায়। কিন্তু port dynamically assign হয় (4566 নয়) — `describe-db-instances` দিয়ে port বের করতে হবে।

---

## পার্ট ২ — Floci হাতে-কলমে

---

### ধাপ ০ — Floci চালু করো

**কেন করছি?** Floci চালু না থাকলে কোনো AWS CLI command কাজ করবে না।

```bash
floci start --persist ./floci-data
eval $(floci env)
echo $AWS_ENDPOINT_URL
```

**প্রত্যাশিত output:**

```
http://localhost:4566
```

---

### ধাপ ১ — Security Group তৈরি করো

**কেন করছি?** RDS PostgreSQL-এ connect করতে port 5432 open করতে হবে। Security Group হলো virtual firewall — এটা নিয়ন্ত্রণ করে কোথা থেকে database-এ connection আসতে পারবে।

```bash
aws ec2 create-security-group \
  --group-name OOAAOW-db-sg \
  --description "OOAAOW RDS PostgreSQL Security Group"
```

**প্রত্যাশিত output:**

```json
{
  "GroupId": "sg-xxxxxxxxxxxxxxxxx"
}
```

```bash
DB_SG_ID=$(aws ec2 describe-security-groups \
  --group-names OOAAOW-db-sg \
  --query 'SecurityGroups[0].GroupId' \
  --output text)
echo $DB_SG_ID
```

**প্রত্যাশিত output:**

```
sg-xxxxxxxxxxxxxxxxx
```

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $DB_SG_ID \
  --protocol tcp \
  --port 5432 \
  --cidr 0.0.0.0/0
```

**প্রত্যাশিত output:**

```json
{
  "Return": true
}
```

---

### ধাপ ২ — RDS PostgreSQL Instance তৈরি করো

**কেন করছি?** এটাই মূল কাজ — একটা PostgreSQL database server তৈরি করা। `create-db-instance` command-টা Floci-তে একটা real PostgreSQL process start করে। `OOAAOW-db` হলো instance-এর identifier (server-এর নাম), `OOAAOW` হলো database-এর নাম।

```bash
aws rds create-db-instance \
  --db-instance-identifier OOAAOW-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 17 \
  --master-username postgres \
  --master-user-password OOAAOW2026 \
  --allocated-storage 20 \
  --db-name OOAAOW \
  --no-multi-az \
  --no-publicly-accessible \
  --vpc-security-group-ids $DB_SG_ID
```

**প্রত্যাশিত output:**

```json
{
  "DBInstance": {
    "DBInstanceIdentifier": "OOAAOW-db",
    "DBInstanceClass": "db.t3.micro",
    "Engine": "postgres",
    "DBInstanceStatus": "creating",
    "MasterUsername": "postgres",
    "DBName": "OOAAOW",
    "AllocatedStorage": 20,
    "EngineVersion": "17"
  }
}
```

> ℹ️ Status `"creating"` মানে PostgreSQL engine start হচ্ছে। ১০-৩০ সেকেন্ড অপেক্ষা করো।

---

### ধাপ ৩ — Instance Status এবং Endpoint Port দেখো

**কেন করছি?** Floci RDS-এ database একটা dynamically assigned port-এ চলে (4566 নয়)। `psql` connect করতে এই port-টা জানতে হবে।

```bash
aws rds describe-db-instances \
  --db-instance-identifier OOAAOW-db \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Endpoint:Endpoint}'
```

**প্রত্যাশিত output:**

```json
{
  "Status": "available",
  "Endpoint": {
    "Address": "localhost",
    "Port": 4510,
    "HostedZoneId": "Z2R2ITUGPM61AM"
  }
}
```

```bash
DB_PORT=$(aws rds describe-db-instances \
  --db-instance-identifier OOAAOW-db \
  --query 'DBInstances[0].Endpoint.Port' \
  --output text)
echo "DB Port: $DB_PORT"
```

**প্রত্যাশিত output:**

```
DB Port: 4510
```

> ℹ️ Port number তোমার installation-এ ভিন্ন হতে পারে (4510, 4511, ইত্যাদি)। `$DB_PORT` variable ব্যবহার করলে নিচের সব command automatically সঠিক port ব্যবহার করবে।

---

### ধাপ ৪ — psql দিয়ে Connect করো

**কেন করছি?** Database তৈরি হয়েছে কিনা verify করা এবং সরাসরি SQL command চালানো।

> ℹ️ **psql install লাগবে না।** Docker Desktop তো চলছেই Floci-র জন্য — `postgres:15-alpine` image দিয়ে `psql` চালানো যাবে। Docker container-এর ভেতর থেকে host-এর port ধরতে `localhost`-এর বদলে `host.docker.internal` দিতে হয়।

```bash
docker run --rm -it \
  -e PGPASSWORD=OOAAOW2026 \
  postgres:15-alpine \
  psql -h host.docker.internal -p $DB_PORT -U postgres -d OOAAOW
```

**প্রত্যাশিত output:**

```
psql (17.x)
Type "help" for help.

OOAAOW=#
```

**`OOAAOW=#` prompt মানে সফলভাবে connect হয়েছ।**

psql-এর ভেতরে কিছু command চালাও:

```sql
-- Database list দেখো
\l

-- Current database-এর tables দেখো (এখন খালি)
\dt

-- PostgreSQL version দেখো
SELECT version();

-- psql থেকে বের হও
\q
```

**প্রত্যাশিত output (version):**

```
                                   version
---------------------------------------------------------------------------
 PostgreSQL 17.x on x86_64-pc-linux-musl, compiled by gcc ...
(1 row)
```

---

### ধাপ ৫ — OOAAOWMetrics.Web Prisma Schema Migrate করো

**কেন করছি?** OOAAOWMetrics.Web-এর সব tables (Tenant, User, Product, GA4Credential, ইত্যাদি) automatically তৈরি হবে। Real production-এ এভাবেই `prisma db push` চালিয়ে database structure তৈরি করা হয়।

```bash
# OOAAOWMetrics.Web project-এ যাও
cd /d/OOAAOW/OOAAOWMetrics.Web

# DATABASE_URL set করো (Floci RDS endpoint)
export DATABASE_URL="postgresql://postgres:OOAAOW2026@localhost:${DB_PORT}/OOAAOW"
echo $DATABASE_URL
```

**প্রত্যাশিত output:**

```
postgresql://postgres:OOAAOW2026@localhost:4510/OOAAOW
```

```bash
# Prisma schema sync করো
npx prisma db push
```

**প্রত্যাশিত output:**

```
Environment variables loaded from .env
Prisma schema loaded from prisma/schema.prisma
Datasource "db": PostgreSQL database "OOAAOW", schema "public" at "localhost:4510"

🚀  Your database is now in sync with your Prisma schema. Done in 1.23s
```

**যাচাই করো — psql দিয়ে tables দেখো:**

```bash
docker run --rm \
  -e PGPASSWORD=OOAAOW2026 \
  postgres:15-alpine \
  psql -h host.docker.internal -p $DB_PORT -U postgres \
  -d OOAAOW \
  -c "\dt"
```

**প্রত্যাশিত output:**

```
             List of relations
 Schema |        Name        | Type  |  Owner
--------+--------------------+-------+----------
 public | Account            | table | postgres
 public | GA4Credential      | table | postgres
 public | Product            | table | postgres
 public | Session            | table | postgres
 public | Tenant             | table | postgres
 public | User               | table | postgres
 public | VerificationToken  | table | postgres
(7 rows)
```

---

### ধাপ ৬ — Sample Data Insert এবং Query করো

**কেন করছি?** Database এবং tables কাজ করছে কিনা verify করতে actual data insert এবং query করা।

```bash
docker run --rm -it \
  -e PGPASSWORD=OOAAOW2026 \
  postgres:15-alpine \
  psql -h host.docker.internal -p $DB_PORT -U postgres -d OOAAOW
```

```sql
-- Tenant তৈরি করো
INSERT INTO "Tenant" (id, name, slug, plan, "createdAt", "updatedAt")
VALUES (
  'tenant_001',
  'OOAAOW',
  'OOAAOW',
  'PRO',
  NOW(),
  NOW()
);

-- User তৈরি করো
INSERT INTO "User" (id, email, name, role, "tenantId", "createdAt")
VALUES (
  'user_001',
  'admin@OOAAOW.com',
  'Asif Abdullah',
  'OWNER',
  'tenant_001',
  NOW()
);

-- Data দেখো
SELECT u.name, u.email, u.role, t.name AS tenant
FROM "User" u
JOIN "Tenant" t ON u."tenantId" = t.id;

\q
```

**প্রত্যাশিত output:**

```
     name      |        email        | role  |  tenant
---------------+---------------------+-------+---------
 Asif Abdullah | admin@OOAAOW.com  | OWNER | OOAAOW
(1 row)
```

---

### ধাপ ৭ — Cleanup

**কেন করছি?** Session শেষে resource delete করা। Real AWS-এ RDS running থাকলে charge হয়।

```bash
aws rds delete-db-instance \
  --db-instance-identifier OOAAOW-db \
  --skip-final-snapshot
```

**প্রত্যাশিত output:**

```json
{
  "DBInstance": {
    "DBInstanceIdentifier": "OOAAOW-db",
    "DBInstanceStatus": "deleting"
  }
}
```

**যাচাই করো:**

```bash
aws rds describe-db-instances
```

**প্রত্যাশিত output:**

```json
{
  "DBInstances": []
}
```

---

## পার্ট ৩ — Real AWS RDS PostgreSQL

> **কখন করবে:** Floci-তে পার্ট ২-এর সব ধাপ (Security Group, RDS create, psql connect, Prisma migrate) সফলভাবে শেষ করার পর, Real AWS Free Tier account নিয়ে এখানে আসো।

---

### ধাপ ১ — DB Subnet Group তৈরি করো

**কেন করছি?** RDS-কে অবশ্যই একটা VPC subnet-এ রাখতে হবে। Subnet Group define করে কোন subnets-এ RDS থাকতে পারবে। কমপক্ষে ২টা different Availability Zone লাগবে।

```bash
# Default VPC-এর ID বের করো
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)
echo $VPC_ID

# Subnets বের করো
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].[SubnetId,AvailabilityZone]' \
  --output table
```

```bash
# DB Subnet Group তৈরি করো (দুটো subnet ID দাও)
aws rds create-db-subnet-group \
  --db-subnet-group-name OOAAOW-subnet-group \
  --db-subnet-group-description "OOAAOW RDS Subnet Group" \
  --subnet-ids subnet-XXXXXXXX subnet-YYYYYYYY
```

**প্রত্যাশিত output (Real AWS):**

```json
{
  "DBSubnetGroup": {
    "DBSubnetGroupName": "OOAAOW-subnet-group",
    "DBSubnetGroupDescription": "OOAAOW RDS Subnet Group",
    "SubnetGroupStatus": "Complete"
  }
}
```

---

### ধাপ ২ — Security Group তৈরি করো

```bash
DB_SG_ID=$(aws ec2 create-security-group \
  --group-name OOAAOW-rds-sg \
  --description "OOAAOW RDS PostgreSQL" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $DB_SG_ID \
  --protocol tcp \
  --port 5432 \
  --cidr 0.0.0.0/0
```

---

### ধাপ ৩ — RDS PostgreSQL Instance তৈরি করো (Free Tier)

```bash
aws rds create-db-instance \
  --db-instance-identifier OOAAOW-prod-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 16 \
  --master-username postgres \
  --master-user-password YOUR_SECURE_PASSWORD \
  --allocated-storage 20 \
  --db-name OOAAOW \
  --no-multi-az \
  --publicly-accessible \
  --vpc-security-group-ids $DB_SG_ID \
  --db-subnet-group-name OOAAOW-subnet-group \
  --backup-retention-period 7
```

> ⏱ ১০-১৫ মিনিট সময় লাগবে। নিচের command দিয়ে status check করো:

```bash
aws rds describe-db-instances \
  --db-instance-identifier OOAAOW-prod-db \
  --query 'DBInstances[0].DBInstanceStatus' \
  --output text
```

`available` দেখালে পরের ধাপে যাও।

---

### ধাপ ৪ — Endpoint পাও এবং Connect করো

```bash
DB_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier OOAAOW-prod-db \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)
echo $DB_ENDPOINT
```

**প্রত্যাশিত output (Real AWS):**

```
OOAAOW-prod-db.abc123xyz.us-east-1.rds.amazonaws.com
```

```bash
PGPASSWORD=YOUR_SECURE_PASSWORD psql \
  -h $DB_ENDPOINT \
  -p 5432 \
  -U postgres \
  -d OOAAOW
```

---

### ধাপ ৫ — Prisma Migrate চালাও

```bash
cd /d/OOAAOW/OOAAOWMetrics.Web

export DATABASE_URL="postgresql://postgres:YOUR_SECURE_PASSWORD@${DB_ENDPOINT}:5432/OOAAOW"

npx prisma db push
```

---

### ধাপ ৬ — Application-এ DATABASE_URL Update করো

**`.env` file update করো:**

```
DATABASE_URL="postgresql://postgres:YOUR_SECURE_PASSWORD@OOAAOW-prod-db.abc123.us-east-1.rds.amazonaws.com:5432/OOAAOW"
```

Production deployment-এ এই value টা environment variable হিসেবে inject হবে — Docker Compose-এ বা ECS Task Definition-এ।

---

## পার্ট ৪ — Self-managed PostgreSQL on EC2 (রেফারেন্স)

> ⚠️ **Floci-তে এই পার্ট সম্ভব না।** Floci real VM চালায় না, তাই SSH এবং software installation হয় না।
> Real AWS-এ একটা Ubuntu EC2 launch করে এই ধাপগুলো করো।

**কখন EC2-তে PostgreSQL বেছে নেবে:**

- Cost কমাতে হবে (RDS-এর চেয়ে সস্তা)
- Learning বা development environment
- Full PostgreSQL configuration control দরকার

---

### ধাপ ১ — Ubuntu EC2 Launch করো

Real AWS console থেকে launch করো:

- AMI: Ubuntu 22.04 LTS
- Instance type: t2.micro (Free Tier)
- Key pair: বানিয়ে রাখো বা existing use করো
- Security Group: SSH (22) + Custom TCP (5432) — তোমার IP from

---

### ধাপ ২ — SSH করো এবং PostgreSQL Install করো

```bash
# EC2-তে SSH করো
ssh -i my-key.pem ubuntu@YOUR_EC2_PUBLIC_IP

# Update এবং install
sudo apt update
sudo apt install postgresql postgresql-contrib -y

# Status চেক করো
sudo systemctl status postgresql
```

**প্রত্যাশিত output (Real AWS):**

```
● postgresql.service - PostgreSQL RDBMS
     Active: active (running)
```

---

### ধাপ ৩ — Remote Access Configure করো

**কেন করছি?** PostgreSQL default-এ শুধু localhost থেকে connection নেয়। Remote (local PC বা অন্য EC2) থেকে connect করতে config পরিবর্তন করতে হবে।

```bash
# PostgreSQL config file খোঁজো এবং edit করো
sudo nano /etc/postgresql/*/main/postgresql.conf
```

এই line খোঁজো এবং পরিবর্তন করো:

```
# এই line টা পাও:
#listen_addresses = 'localhost'

# এভাবে বদলাও:
listen_addresses = '*'
```

```bash
# Client authentication config edit করো
sudo nano /etc/postgresql/*/main/pg_hba.conf
```

File-এর শেষে এই line add করো:

```
host    all             all             0.0.0.0/0               md5
```

```bash
# PostgreSQL restart করো
sudo systemctl restart postgresql
```

---

### ধাপ ৪ — User তৈরি করো

```bash
# postgres system user হিসেবে psql-এ login করো
sudo -u postgres psql
```

```sql
-- Remote user তৈরি করো
CREATE USER devops WITH PASSWORD 'devops2026';

-- Database তৈরি করো
CREATE DATABASE OOAAOW OWNER devops;

-- Permission দাও
GRANT ALL PRIVILEGES ON DATABASE OOAAOW TO devops;

\q
```

**Local PC থেকে connect করো:**

```bash
PGPASSWORD=devops2026 psql \
  -h YOUR_EC2_PUBLIC_IP \
  -p 5432 \
  -U devops \
  -d OOAAOW
```

---

## দ্রুত তথ্যসূত্র — RDS Command Cheat Sheet

| Command                                                                                                   | কাজ                       |
| --------------------------------------------------------------------------------------------------------- | ------------------------- |
| `aws rds create-db-instance --db-instance-identifier NAME --engine postgres ...`                          | RDS instance তৈরি         |
| `aws rds describe-db-instances --db-instance-identifier NAME`                                             | Instance details দেখো     |
| `aws rds describe-db-instances --query 'DBInstances[0].Endpoint'`                                         | Endpoint এবং port পাও     |
| `aws rds delete-db-instance --db-instance-identifier NAME --skip-final-snapshot`                          | Instance delete করো       |
| `aws rds create-db-snapshot --db-instance-identifier NAME --db-snapshot-identifier SNAP`                  | Manual snapshot নাও       |
| `aws rds restore-db-instance-from-db-snapshot --db-instance-identifier NEW --db-snapshot-identifier SNAP` | Snapshot থেকে restore করো |
| `aws rds modify-db-instance --db-instance-identifier NAME --allocated-storage 50 --apply-immediately`     | Storage বাড়াও            |
| `aws rds describe-db-engine-versions --engine postgres --query 'DBEngineVersions[*].EngineVersion'`       | Available versions দেখো   |
| `aws rds create-db-subnet-group --db-subnet-group-name NAME --subnet-ids ...`                             | Subnet group তৈরি         |

---

## Real AWS Console-এ ফ্লো (রেফারেন্স)

**সংক্ষেপে:**
`RDS Dashboard → Create Database → PostgreSQL → Free Tier → Configure → Create → Wait 10-15 min → Endpoint copy → psql connect → prisma db push → Tables ready`

<details>
<summary>📊 বিস্তারিত ভিজ্যুয়াল ডায়াগ্রাম দেখতে ক্লিক করো</summary>

```mermaid
flowchart TD
    A[RDS Dashboard] --> B[Create Database]
    B --> C[Standard Create - PostgreSQL 16]
    C --> D[Free Tier Template]
    D --> E[DB identifier: OOAAOW-prod-db]
    E --> F[Master username: postgres - set password]
    F --> G[db.t3.micro - 20GB - no Multi-AZ]
    G --> H[VPC - Subnet Group - Security Group port 5432]
    H --> I[Create Database - wait 10-15 min]
    I --> J[Status: available]
    J --> K[Endpoint copied from console]
    K --> L[psql connected successfully]
    L --> M[prisma db push runs]
    M --> N[All 7 tables created - app ready]
```

</details>

---

## আজকে যা তৈরি করলে

```
Day9-RDS-floci/
├── Security Group (OOAAOW-db-sg) — port 5432
├── RDS PostgreSQL Instance (OOAAOW-db)
│   └── Database: OOAAOW
│       ├── Tenant table
│       ├── User table
│       ├── Product table
│       ├── GA4Credential table
│       ├── Account table
│       ├── Session table
│       └── VerificationToken table
└── Sample data: Tenant + User inserted and queried
```

| তৈরি                       | Floci | Real AWS  |
| -------------------------- | ----- | --------- |
| Security Group (port 5432) | ✅    | ✅        |
| RDS PostgreSQL instance    | ✅    | ✅        |
| psql connection            | ✅    | ✅        |
| Prisma db push             | ✅    | ✅        |
| Multi-AZ                   | ❌    | ✅ (paid) |
| EC2-তে PostgreSQL install  | ❌    | ✅        |

---

## বাড়ির কাজ

1. Floci-তে RDS তৈরি করো এবং `prisma db push` চালিয়ে সব tables তৈরি করো। তারপর `Product` table-এ ৩টা sample product insert করো এবং `SELECT` দিয়ে দেখাও।
2. Real AWS-এ RDS Free Tier PostgreSQL create করো। Endpoint পাও। `psql` দিয়ে connect করো।
3. Floci-তে `aws rds describe-db-instances --query 'DBInstances[0].Endpoint.Port' --output text` চালিয়ে port নোট করো — কত দেখাচ্ছে?

---

## রিসোর্স

- Floci: https://floci.io
- Floci AWS services: https://floci.io/aws
- AWS RDS docs: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/
- AWS RDS CLI reference: https://docs.aws.amazon.com/cli/latest/reference/rds/
- Prisma PostgreSQL docs: https://www.prisma.io/docs/concepts/database-connectors/postgresql
