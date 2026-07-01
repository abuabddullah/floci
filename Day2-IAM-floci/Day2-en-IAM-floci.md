# Day 2: AWS IAM — Identity and Access Management
### Hands-on with Floci (Local AWS Emulator)

> **What is Floci?**
> Floci is a free, open-source AWS emulator. It runs on your PC at `http://localhost:4566`.
> You can practice every AWS command without a real AWS account.

---

## What You Will Learn Today

- What IAM is and why it matters
- IAM Users, Groups, Policies, and Roles — theory + hands-on
- Create real IAM resources using AWS CLI + Floci
- Attach policies, add users to groups, create roles
- Assume a role using STS (real DevOps workflow)

---

## Part 1 — Theory

### What is IAM?

**IAM = Identity and Access Management**

IAM is the AWS service that controls:
- **Who** can access your AWS resources
- **What** they are allowed to do

Without IAM — anyone with access could delete your servers, empty your S3 buckets, or destroy your databases.

---

### IAM Core Components

#### 1. IAM Users
A **user** represents a person or an application.

Examples:
```
dev-user        ← a developer
admin-user      ← a system administrator
ci-bot          ← a CI/CD pipeline automation account
```

Each user gets their own credentials (access key or password).

---

#### 2. IAM Groups
A **group** is a collection of users that share the same permissions.

Examples:
```
Developers      ← all developers
Admins          ← all administrators
ReadOnlyUsers   ← people who can only view resources
```

Assign permissions to the group → every user in the group inherits them.
This is much easier than setting permissions for each user individually.

---

#### 3. IAM Policies
A **policy** defines what actions are allowed or denied.

Policies are written in **JSON format**.

Example — EC2 Read-Only Policy:
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

This policy allows viewing EC2 instances but not creating or deleting them.

---

#### 4. IAM Roles
A **role** is like a temporary identity that AWS services can assume.

Example use cases:
- An EC2 instance needs to read from S3 → attach an S3 role to EC2
- A Lambda function needs to write to DynamoDB → attach a DynamoDB role to Lambda
- A developer needs temporary admin access → they assume an admin role

Roles do **not** have permanent credentials. They issue **temporary tokens** via STS.

---

### IAM Best Practices

| Rule | Why |
|------|-----|
| Never use the root account for daily tasks | Root has unlimited power — one mistake can destroy everything |
| Create individual IAM users | Each person should have their own credentials |
| Use groups to manage permissions | Easier than managing each user separately |
| Follow least privilege principle | Give only the minimum permissions required |
| Enable MFA (Multi-Factor Authentication) | Extra layer of security for important accounts |
| Rotate access keys regularly | Old keys are a security risk |

---

## Part 2 — Hands-On with Floci

### Prerequisites

Make sure these are installed (from Day 1):
- Docker Desktop (running)
- AWS CLI v2
- Floci CLI

---

### Step 0 — Start Floci

Open a terminal and run:

```powershell
floci start --persist ./floci-data
```

Expected output:
```
✓ Floci is running on http://localhost:4566
```

> **Why `--persist ./floci-data`?** Without this flag, all IAM users, groups, roles, and policies are erased when Floci stops. This flag saves everything in the `floci-data` folder so your work survives between sessions.

---

### Step 0.1 — Set Environment Variables

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

Or manually:
```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
```

> **Note:** In Floci, the AWS account ID is `000000000000` (twelve zeros). You will see this in ARNs.

---

### Step 1 — Create IAM Users

Create three users representing different team members:

```bash
aws iam create-user --user-name dev-user
aws iam create-user --user-name admin-user
aws iam create-user --user-name readonly-user
```

Expected output for each:
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

**Verify — list all users:**
```bash
aws iam list-users
```

---

### Step 2 — Create IAM Groups

Create three groups for different teams:

```bash
aws iam create-group --group-name Developers
aws iam create-group --group-name Admins
aws iam create-group --group-name ReadOnlyUsers
```

**Verify — list all groups:**
```bash
aws iam list-groups
```

---

### Step 3 — Create a Custom IAM Policy

We will create a custom policy that allows only S3 read access.

First, create the policy document file. Save this as `s3-readonly-policy.json`:

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

Now create the policy in Floci:

```bash
aws iam create-policy \
  --policy-name S3ReadOnlyPolicy \
  --policy-document file://s3-readonly-policy.json
```

Expected output:
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

**Verify — list custom policies:**
```bash
aws iam list-policies --scope Local
```

---

### Step 4 — Attach Policies to Groups

Attach the custom S3 policy to the `ReadOnlyUsers` group:

```bash
aws iam attach-group-policy \
  --group-name ReadOnlyUsers \
  --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy
```

Attach AWS managed EC2 full access to `Developers` group:

```bash
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
```

Attach AWS managed AdministratorAccess to `Admins` group:

```bash
aws iam attach-group-policy \
  --group-name Admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

**Verify — list policies attached to a group:**
```bash
aws iam list-attached-group-policies --group-name Developers
aws iam list-attached-group-policies --group-name ReadOnlyUsers
```

---

### Step 5 — Add Users to Groups

Now assign each user to the appropriate group:

```bash
aws iam add-user-to-group --group-name Developers    --user-name dev-user
aws iam add-user-to-group --group-name Admins        --user-name admin-user
aws iam add-user-to-group --group-name ReadOnlyUsers --user-name readonly-user
```

**Verify — list users in a group:**
```bash
aws iam get-group --group-name Developers
aws iam get-group --group-name Admins
```

**Verify — check which groups a user belongs to:**
```bash
aws iam list-groups-for-user --user-name dev-user
```

---

### Step 6 — Attach Policy Directly to a User

Sometimes you need to give one specific user an extra permission outside their group.

Attach S3 read access directly to `dev-user`:

```bash
aws iam attach-user-policy \
  --user-name dev-user \
  --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy
```

**Verify — list policies attached directly to a user:**
```bash
aws iam list-attached-user-policies --user-name dev-user
```

---

### Step 7 — Create Access Keys for a User

Access keys allow a user to authenticate via CLI or SDK (instead of a password).

```bash
aws iam create-access-key --user-name dev-user
```

Expected output:
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

**List access keys for a user:**
```bash
aws iam list-access-keys --user-name dev-user
```

---

### Step 8 — Create an IAM Role

Roles are used by AWS services. Let's create a role that allows EC2 to access S3.

First, create the trust policy document. Save as `ec2-trust-policy.json`:

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

Create the role:

```bash
aws iam create-role \
  --role-name EC2-S3-Access-Role \
  --assume-role-policy-document file://ec2-trust-policy.json
```

Expected output:
```json
{
    "Role": {
        "Path": "/",
        "RoleName": "EC2-S3-Access-Role",
        "RoleId": "AROAXXXXXXXXXXXXXXXXXX",
        "Arn": "arn:aws:iam::000000000000:role/EC2-S3-Access-Role",
        "CreateDate": "2026-07-01T00:00:00+00:00",
        "AssumeRolePolicyDocument": { ... }
    }
}
```

Now attach the S3 read policy to this role:

```bash
aws iam attach-role-policy \
  --role-name EC2-S3-Access-Role \
  --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy
```

**Verify — list roles:**
```bash
aws iam list-roles
```

**Verify — list policies attached to a role:**
```bash
aws iam list-attached-role-policies --role-name EC2-S3-Access-Role
```

---

### Step 9 — Assume a Role (STS)

This is a real DevOps workflow — temporarily assume a role to gain its permissions.

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::000000000000:role/EC2-S3-Access-Role \
  --role-session-name my-test-session
```

Expected output:
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

The temporary credentials (`AccessKeyId`, `SecretAccessKey`, `SessionToken`) can now be used to perform actions as this role.

---

### Step 10 — Get Current Identity

Check who you are currently authenticated as:

```bash
aws sts get-caller-identity
```

Expected output:
```json
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

---

### Step 11 — Clean Up (Optional)

Remove all the resources you created:

```bash
# Remove users from groups
aws iam remove-user-from-group --group-name Developers    --user-name dev-user
aws iam remove-user-from-group --group-name Admins        --user-name admin-user
aws iam remove-user-from-group --group-name ReadOnlyUsers --user-name readonly-user

# Detach policies from groups
aws iam detach-group-policy --group-name Developers    --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
aws iam detach-group-policy --group-name Admins        --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam detach-group-policy --group-name ReadOnlyUsers --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy

# Delete groups
aws iam delete-group --group-name Developers
aws iam delete-group --group-name Admins
aws iam delete-group --group-name ReadOnlyUsers

# Detach and delete policy from user
aws iam detach-user-policy --user-name dev-user --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy

# Delete access keys before deleting user
aws iam list-access-keys --user-name dev-user
# Copy the AccessKeyId from above output, then:
aws iam delete-access-key --user-name dev-user --access-key-id <AccessKeyId>

# Delete users
aws iam delete-user --user-name dev-user
aws iam delete-user --user-name admin-user
aws iam delete-user --user-name readonly-user

# Detach policy from role
aws iam detach-role-policy --role-name EC2-S3-Access-Role --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy

# Delete role
aws iam delete-role --role-name EC2-S3-Access-Role

# Delete custom policy
aws iam delete-policy --policy-arn arn:aws:iam::000000000000:policy/S3ReadOnlyPolicy
```

---

## Quick Reference — IAM Commands

| What to do | Command |
|------------|---------|
| Create user | `aws iam create-user --user-name <name>` |
| List users | `aws iam list-users` |
| Create group | `aws iam create-group --group-name <name>` |
| List groups | `aws iam list-groups` |
| Add user to group | `aws iam add-user-to-group --group-name <g> --user-name <u>` |
| Create policy | `aws iam create-policy --policy-name <n> --policy-document file://policy.json` |
| Attach policy to group | `aws iam attach-group-policy --group-name <g> --policy-arn <arn>` |
| Attach policy to user | `aws iam attach-user-policy --user-name <u> --policy-arn <arn>` |
| Create role | `aws iam create-role --role-name <n> --assume-role-policy-document file://trust.json` |
| Attach policy to role | `aws iam attach-role-policy --role-name <n> --policy-arn <arn>` |
| Assume role | `aws sts assume-role --role-arn <arn> --role-session-name <session>` |
| Get current identity | `aws sts get-caller-identity` |

---

## What You Practiced Today

- [x] Created IAM Users (`dev-user`, `admin-user`, `readonly-user`)
- [x] Created IAM Groups (`Developers`, `Admins`, `ReadOnlyUsers`)
- [x] Created a custom IAM Policy (S3 Read-Only)
- [x] Attached managed and custom policies to groups
- [x] Added users to groups
- [x] Attached a policy directly to a user
- [x] Created access keys for a user
- [x] Created an IAM Role with a trust policy
- [x] Assumed a role using STS
- [x] Verified identity with `get-caller-identity`

---

## Homework

1. Create a new user `qa-user` and a group `QA` with only S3 full access
2. Create a custom policy that allows Lambda read-only access
3. Create a role that allows Lambda to write to DynamoDB

---

## Resources

- Floci Official: [https://floci.io](https://floci.io)
- Floci AWS Services: [https://floci.io/aws](https://floci.io/aws)
- AWS IAM CLI Reference: [https://docs.aws.amazon.com/cli/latest/reference/iam/](https://docs.aws.amazon.com/cli/latest/reference/iam/)
- AWS IAM User Guide: [https://docs.aws.amazon.com/IAM/latest/UserGuide/](https://docs.aws.amazon.com/IAM/latest/UserGuide/)
