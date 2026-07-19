# Day 9: RDS — AWS Relational Database Service

### Hands-On Learning with Floci (Git Bash)

> All commands must be run in **Git Bash**.
> **Docker Desktop** must be running before executing any commands.

📌 **Connect / Social Media:**
[LinkedIn](https://www.linkedin.com/in/asifaowadud) · [YouTube](https://www.youtube.com/@OOAAOW?sub_confirmation=1) · [Telegram](https://t.me/ooaaow) · [Web Lab](https://oao-devops-lab.vercel.app/) · [Facebook](https://www.facebook.com/OOAAOW/)

---

## Part 1 — Theory

### What is RDS?

> **AWS RDS (Relational Database Service)** is AWS's fully managed relational database service — AWS launches the database server, configures it, takes backups, patches it, and monitors it.

You only work with schema and data — server management is AWS's responsibility.

| What RDS Gives You | Details                                           |
| ------------------ | ------------------------------------------------- |
| Database Engine    | PostgreSQL, MySQL, MariaDB, Oracle, SQL Server    |
| Automated Backup   | Daily snapshot + transaction log                  |
| Patch Management   | AWS patches the OS and database engine            |
| High Availability  | Multi-AZ: standby replica with automatic failover |
| Scalability        | Storage and compute can be scaled independently   |
| Monitoring         | CloudWatch metrics built-in                       |

---

### Why RDS Matters (DevOps Perspective)

| Problem (without RDS)                  | Solution (with RDS)                            |
| -------------------------------------- | ---------------------------------------------- |
| Database server crash → data loss      | Multi-AZ: standby becomes active automatically |
| Forgot to take backup → disaster       | Automated daily backup + point-in-time restore |
| Delayed security patch → vulnerability | AWS patches automatically                      |
| Need to hire a DBA                     | AWS managed — no DBA required                  |
| Scaling requires downtime              | Change storage with one command                |

**Real-world context:** OOAAOWMetrics.Web is a Next.js analytics dashboard. It uses PostgreSQL via Prisma ORM. In production, this database runs on RDS PostgreSQL.

---

### Core RDS Components

| Component           | What it does                                     | Example                                     |
| ------------------- | ------------------------------------------------ | ------------------------------------------- |
| **DB Instance**     | The actual running database server               | `db.t3.micro` PostgreSQL 16                 |
| **DB Subnet Group** | Defines which VPC subnets RDS can use            | private-subnet-1a, private-subnet-1b        |
| **Security Group**  | Controls which traffic reaches port 5432         | Allow port 5432 from EC2                    |
| **Parameter Group** | PostgreSQL configuration (max_connections, etc.) | default.postgres16                          |
| **Snapshot**        | Point-in-time backup                             | Automated snapshot at 3 AM daily            |
| **Endpoint**        | The address your app connects to                 | `mydb.abc.us-east-1.rds.amazonaws.com:5432` |

---

### Supported Database Engines

| Engine         | RDS Name       | When to use                                       |
| -------------- | -------------- | ------------------------------------------------- |
| **PostgreSQL** | `postgres`     | Modern apps, JSON support, open source preference |
| **MySQL**      | `mysql`        | Traditional web apps, WordPress                   |
| **MariaDB**    | `mariadb`      | MySQL-compatible, community fork                  |
| **Oracle**     | `oracle-ee`    | Enterprise legacy systems                         |
| **SQL Server** | `sqlserver-se` | .NET applications, Microsoft stack                |

**Our choice: PostgreSQL** — because OOAAOWMetrics.Web uses Prisma + PostgreSQL.

---

### OOAAOWMetrics.Web — Real Project Context

OOAAOWMetrics.Web is a SaaS analytics dashboard that displays Google Analytics 4 data. Its Prisma schema has these tables:

```mermaid
graph LR
    TENANT[Tenant] --> USER[User]
    TENANT --> PRODUCT[Product]
    PRODUCT --> GA4[GA4Credential]
    USER --> ACCOUNT[Account]
    USER --> SESSION[Session]
```

| Table               | What it stores                           |
| ------------------- | ---------------------------------------- |
| `Tenant`            | Workspace or organization (multi-tenant) |
| `User`              | Login user — role: OWNER, ADMIN, MEMBER  |
| `Product`           | Analytics product (Web, Android, iOS)    |
| `GA4Credential`     | Google Analytics OAuth token             |
| `Account`           | NextAuth OAuth account link              |
| `Session`           | Login session                            |
| `VerificationToken` | Email verification token                 |

Today we'll create all these tables in RDS PostgreSQL using `prisma db push`.

---

### Multi-AZ Architecture

```mermaid
graph TB
    APP[Application - EC2] --> PRIMARY[RDS Primary - AZ 1a]
    PRIMARY -->|sync| STANDBY[RDS Standby - AZ 1b]
    PRIMARY -->|failover on crash| STANDBY
    STANDBY -->|becomes new primary| APP
```

**How it works:**

- Every write to the Primary is synchronously replicated to the Standby
- If the Primary crashes, AWS promotes the Standby automatically
- The application endpoint stays the same — zero manual intervention

**On Free Tier:** Multi-AZ is disabled (single AZ) to save cost.

---

### RDS vs Self-managed PostgreSQL on EC2

| Aspect          | RDS PostgreSQL       | PostgreSQL on EC2             |
| --------------- | -------------------- | ----------------------------- |
| Setup           | One command          | SSH → install → configure     |
| Backup          | Automatic            | Manual (write a cron job)     |
| Patching        | AWS handles it       | You handle it                 |
| Failover        | Automatic (Multi-AZ) | Manual                        |
| Monitoring      | CloudWatch built-in  | Set up yourself               |
| Cost (t3.micro) | ~$15-25 per month    | ~$8-10 per month (EC2 only)   |
| Best for        | Production           | Learning or cost optimization |

---

### Floci RDS Support

| Command / Feature               | Floci | Details                                         |
| ------------------------------- | ----- | ----------------------------------------------- |
| `aws rds create-db-instance`    | ✅    | Starts a real PostgreSQL engine                 |
| `aws rds describe-db-instances` | ✅    | Returns endpoint and port                       |
| `aws rds delete-db-instance`    | ✅    | Terminates the instance                         |
| `psql` connection               | ✅    | Real database — actual SQL runs                 |
| `prisma db push`                | ✅    | Prisma schema migration works against Floci RDS |
| Snapshots                       | ⚠️    | Works for PostgreSQL — confirm with testing     |
| Multi-AZ                        | ⚠️    | Config accepted but actual standby does not run |
| EC2 install (Part 4)            | ❌    | Floci does not run real VMs                     |

> ℹ️ **Important:** Floci's RDS runs a **real PostgreSQL database engine**. But the port is dynamically assigned (not 4566) — you must retrieve it with `describe-db-instances`.

---

## Part 2 — Hands-On with Floci

---

### Step 0 — Start Floci

**Why:** Floci must be running before any AWS CLI commands will work.

```bash
floci start --persist ./floci-data
eval $(floci env)
echo $AWS_ENDPOINT_URL
```

**Expected output:**

```
http://localhost:4566
```

---

### Step 1 — Create a Security Group

**Why:** To connect to RDS PostgreSQL, port 5432 must be open. A Security Group is a virtual firewall that controls which traffic can reach the database.

```bash
aws ec2 create-security-group \
  --group-name OOAAOW-db-sg \
  --description "OOAAOW RDS PostgreSQL Security Group"
```

**Expected output:**

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

**Expected output:**

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

**Expected output:**

```json
{
  "Return": true
}
```

---

### Step 2 — Create the RDS PostgreSQL Instance

**Why:** This is the core task — creating a PostgreSQL database server. In Floci, `create-db-instance` actually starts a real PostgreSQL process. `OOAAOW-db` is the instance identifier (the server's name), `OOAAOW` is the database name.

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

**Expected output:**

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

> ℹ️ Status `"creating"` means the PostgreSQL engine is starting. Wait 10–30 seconds.

---

### Step 3 — Check Status and Get the Endpoint Port

**Why:** Floci's RDS database runs on a dynamically assigned port (not 4566). You need this port to connect with `psql`.

```bash
aws rds describe-db-instances \
  --db-instance-identifier OOAAOW-db \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Endpoint:Endpoint}'
```

**Expected output:**

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

**Expected output:**

```
DB Port: 4510
```

> ℹ️ Your port number may differ (4510, 4511, etc.). Using the `$DB_PORT` variable means all commands below will automatically use the correct port.

---

### Step 4 — Connect with psql

**Why:** Verify the database was created and run SQL commands directly.

> ℹ️ **No install needed.** Docker Desktop is already running for Floci — use the `postgres:15-alpine` image to run `psql`. From inside a Docker container, the host's ports are reachable via `host.docker.internal` (not `localhost`).

```bash
docker run --rm -it \
  -e PGPASSWORD=OOAAOW2026 \
  postgres:15-alpine \
  psql -h host.docker.internal -p $DB_PORT -U postgres -d OOAAOW
```

**Expected output:**

```
psql (17.x)
Type "help" for help.

OOAAOW=#
```

**The `OOAAOW=#` prompt means you're connected successfully.**

Run some commands inside psql:

```sql
-- List all databases
\l

-- List tables in this database (empty for now)
\dt

-- Check PostgreSQL version
SELECT version();

-- Exit psql
\q
```

**Expected output (version):**

```
                                   version
---------------------------------------------------------------------------
 PostgreSQL 17.x on x86_64-pc-linux-musl, compiled by gcc ...
(1 row)
```

---

### Step 5 — Migrate OOAAOWMetrics.Web Prisma Schema

**Why:** All of OOAAOWMetrics.Web's tables (Tenant, User, Product, GA4Credential, etc.) will be created automatically. In real production, this is exactly how `prisma db push` creates the database structure.

```bash
# Go to the OOAAOWMetrics.Web project
cd /d/OOAAOW/OOAAOWMetrics.Web

# Set DATABASE_URL to point to Floci RDS
export DATABASE_URL="postgresql://postgres:OOAAOW2026@localhost:${DB_PORT}/OOAAOW"
echo $DATABASE_URL
```

**Expected output:**

```
postgresql://postgres:OOAAOW2026@localhost:4510/OOAAOW
```

```bash
# Sync the Prisma schema to the database
npx prisma db push
```

**Expected output:**

```
Environment variables loaded from .env
Prisma schema loaded from prisma/schema.prisma
Datasource "db": PostgreSQL database "OOAAOW", schema "public" at "localhost:4510"

🚀  Your database is now in sync with your Prisma schema. Done in 1.23s
```

**Verify — check tables with psql:**

```bash
docker run --rm \
  -e PGPASSWORD=OOAAOW2026 \
  postgres:15-alpine \
  psql -h host.docker.internal -p $DB_PORT -U postgres \
  -d OOAAOW \
  -c "\dt"
```

**Expected output:**

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

### Step 6 — Insert and Query Sample Data

**Why:** Confirm that the database and tables work correctly by inserting actual data and querying it back.

```bash
docker run --rm -it \
  -e PGPASSWORD=OOAAOW2026 \
  postgres:15-alpine \
  psql -h host.docker.internal -p $DB_PORT -U postgres -d OOAAOW
```

```sql
-- Create a Tenant
INSERT INTO "Tenant" (id, name, slug, plan, "createdAt", "updatedAt")
VALUES (
  'tenant_001',
  'OOAAOW',
  'OOAAOW',
  'PRO',
  NOW(),
  NOW()
);

-- Create a User
INSERT INTO "User" (id, email, name, role, "tenantId", "createdAt")
VALUES (
  'user_001',
  'admin@OOAAOW.com',
  'Asif Abdullah',
  'OWNER',
  'tenant_001',
  NOW()
);

-- Query the data
SELECT u.name, u.email, u.role, t.name AS tenant
FROM "User" u
JOIN "Tenant" t ON u."tenantId" = t.id;

\q
```

**Expected output:**

```
     name      |        email        | role  |  tenant
---------------+---------------------+-------+---------
 Asif Abdullah | admin@OOAAOW.com  | OWNER | OOAAOW
(1 row)
```

---

### Step 7 — Cleanup

**Why:** Delete resources after the session. On Real AWS, a running RDS instance accrues charges continuously.

```bash
aws rds delete-db-instance \
  --db-instance-identifier OOAAOW-db \
  --skip-final-snapshot
```

**Expected output:**

```json
{
  "DBInstance": {
    "DBInstanceIdentifier": "OOAAOW-db",
    "DBInstanceStatus": "deleting"
  }
}
```

**Verify:**

```bash
aws rds describe-db-instances
```

**Expected output:**

```json
{
  "DBInstances": []
}
```

---

## Part 3 — Real AWS RDS PostgreSQL

> **When to start:** Come here after completing all Floci Part 2 steps (Security Group, RDS create, psql connect, Prisma migrate) successfully, with a Real AWS Free Tier account.

---

### Step 1 — Create a DB Subnet Group

**Why:** RDS must be placed inside a VPC subnet. A Subnet Group defines which subnets RDS can use. At least 2 subnets in different Availability Zones are required.

```bash
# Get the default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)
echo $VPC_ID

# List available subnets
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].[SubnetId,AvailabilityZone]' \
  --output table
```

```bash
# Create the DB Subnet Group (provide two subnet IDs from the list above)
aws rds create-db-subnet-group \
  --db-subnet-group-name OOAAOW-subnet-group \
  --db-subnet-group-description "OOAAOW RDS Subnet Group" \
  --subnet-ids subnet-XXXXXXXX subnet-YYYYYYYY
```

**Expected output (Real AWS):**

```json
{
  "DBSubnetGroup": {
    "DBSubnetGroupName": "OOAAOW-subnet-group",
    "SubnetGroupStatus": "Complete"
  }
}
```

---

### Step 2 — Create a Security Group

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

### Step 3 — Create the RDS PostgreSQL Instance (Free Tier)

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

> ⏱ This takes 10–15 minutes. Check status with:

```bash
aws rds describe-db-instances \
  --db-instance-identifier OOAAOW-prod-db \
  --query 'DBInstances[0].DBInstanceStatus' \
  --output text
```

Proceed when you see `available`.

---

### Step 4 — Get the Endpoint and Connect

```bash
DB_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier OOAAOW-prod-db \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)
echo $DB_ENDPOINT
```

**Expected output (Real AWS):**

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

### Step 5 — Run Prisma Migrate

```bash
cd /d/OOAAOW/OOAAOWMetrics.Web

export DATABASE_URL="postgresql://postgres:YOUR_SECURE_PASSWORD@${DB_ENDPOINT}:5432/OOAAOW"

npx prisma db push
```

---

### Step 6 — Update DATABASE_URL in the Application

**Update your `.env` file:**

```
DATABASE_URL="postgresql://postgres:YOUR_SECURE_PASSWORD@OOAAOW-prod-db.abc123.us-east-1.rds.amazonaws.com:5432/OOAAOW"
```

In production deployment, this value is injected as an environment variable — in a Docker Compose file or ECS Task Definition.

---

## Part 4 — Self-managed PostgreSQL on EC2 (Reference)

> ⚠️ **This part is not possible in Floci.** Floci does not run real VMs, so SSH and software installation are not supported.
> Do these steps on Real AWS — on a real Ubuntu EC2 instance.

**When to choose EC2 over RDS:**

- Reducing cost (cheaper than RDS)
- Learning or development environment
- Need full PostgreSQL configuration control

---

### Step 1 — Launch an Ubuntu EC2 Instance

From the Real AWS console:

- AMI: Ubuntu 22.04 LTS
- Instance type: t2.micro (Free Tier)
- Key pair: create or use an existing one
- Security Group: SSH (22) + Custom TCP (5432) from your IP

---

### Step 2 — SSH In and Install PostgreSQL

```bash
# SSH into the EC2 instance
ssh -i my-key.pem ubuntu@YOUR_EC2_PUBLIC_IP

# Update and install
sudo apt update
sudo apt install postgresql postgresql-contrib -y

# Check status
sudo systemctl status postgresql
```

**Expected output (Real AWS):**

```
● postgresql.service - PostgreSQL RDBMS
     Active: active (running)
```

---

### Step 3 — Configure for Remote Access

**Why:** PostgreSQL only accepts local connections by default. To connect from your local PC or another EC2 instance, you must update the config.

```bash
# Edit the PostgreSQL config file
sudo nano /etc/postgresql/*/main/postgresql.conf
```

Find this line and change it:

```
# Find:
#listen_addresses = 'localhost'

# Change to:
listen_addresses = '*'
```

```bash
# Edit the client authentication config
sudo nano /etc/postgresql/*/main/pg_hba.conf
```

Add this line at the end of the file:

```
host    all             all             0.0.0.0/0               md5
```

```bash
# Restart PostgreSQL
sudo systemctl restart postgresql
```

---

### Step 4 — Create a User and Test Remote Connection

```bash
# Log in as the postgres system user
sudo -u postgres psql
```

```sql
-- Create a remote user
CREATE USER devops WITH PASSWORD 'devops2026';

-- Create the database
CREATE DATABASE OOAAOW OWNER devops;

-- Grant all privileges
GRANT ALL PRIVILEGES ON DATABASE OOAAOW TO devops;

\q
```

**Connect from your local PC:**

```bash
PGPASSWORD=devops2026 psql \
  -h YOUR_EC2_PUBLIC_IP \
  -p 5432 \
  -U devops \
  -d OOAAOW
```

---

## Quick Reference — RDS Command Cheat Sheet

| Command                                                                                                   | What it does            |
| --------------------------------------------------------------------------------------------------------- | ----------------------- |
| `aws rds create-db-instance --db-instance-identifier NAME --engine postgres ...`                          | Create an RDS instance  |
| `aws rds describe-db-instances --db-instance-identifier NAME`                                             | View instance details   |
| `aws rds describe-db-instances --query 'DBInstances[0].Endpoint'`                                         | Get endpoint and port   |
| `aws rds delete-db-instance --db-instance-identifier NAME --skip-final-snapshot`                          | Delete an instance      |
| `aws rds create-db-snapshot --db-instance-identifier NAME --db-snapshot-identifier SNAP`                  | Take a manual snapshot  |
| `aws rds restore-db-instance-from-db-snapshot --db-instance-identifier NEW --db-snapshot-identifier SNAP` | Restore from snapshot   |
| `aws rds modify-db-instance --db-instance-identifier NAME --allocated-storage 50 --apply-immediately`     | Increase storage        |
| `aws rds describe-db-engine-versions --engine postgres --query 'DBEngineVersions[*].EngineVersion'`       | List available versions |
| `aws rds create-db-subnet-group --db-subnet-group-name NAME --subnet-ids ...`                             | Create subnet group     |

---

## Real AWS Console Flow (Reference)

**Summary:**
`RDS Dashboard → Create Database → PostgreSQL → Free Tier → Configure → Create → Wait 10-15 min → Copy Endpoint → psql connect → prisma db push → Tables ready`

<details>
<summary>📊 Click to view detailed visual diagram</summary>

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

## What You Built Today

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

| Built                      | Floci | Real AWS  |
| -------------------------- | ----- | --------- |
| Security Group (port 5432) | ✅    | ✅        |
| RDS PostgreSQL instance    | ✅    | ✅        |
| psql connection            | ✅    | ✅        |
| Prisma db push             | ✅    | ✅        |
| Multi-AZ                   | ❌    | ✅ (paid) |
| PostgreSQL install on EC2  | ❌    | ✅        |

---

## Homework

1. In Floci, create an RDS instance and run `prisma db push` to create all 7 tables. Then insert 3 sample Products and query them back with `SELECT`.
2. On Real AWS, create a Free Tier RDS PostgreSQL instance. Get the endpoint. Connect with `psql`.
3. In Floci, run `aws rds describe-db-instances --query 'DBInstances[0].Endpoint.Port' --output text` — what port number do you see?

---

## Resources

- Floci: https://floci.io
- Floci AWS services: https://floci.io/aws
- AWS RDS docs: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/
- AWS RDS CLI reference: https://docs.aws.amazon.com/cli/latest/reference/rds/
- Prisma PostgreSQL docs: https://www.prisma.io/docs/concepts/database-connectors/postgresql
