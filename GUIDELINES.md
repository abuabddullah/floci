# Floci AWS Course — Documentation Guidelines

এই file-টা assistant-এর জন্য। নতুন chat-এ শুধু এই file-এর path দিলেই assistant বুঝবে কীভাবে documentation লিখতে হবে।

---

## Project Structure

```
floci/
├── GUIDELINES.md                          ← এই ফাইল
├── syllabus.md                            ← 30-day syllabus
├── Day1-Setup-floci/
│   ├── Day1-en-Setup-floci.md             ← English
│   └── Day1-bn-Setup-floci.md             ← Bengali
├── Day2-IAM-floci/
│   ├── Day2-en-IAM-floci.md
│   └── Day2-bn-IAM-floci.md
├── Day3-EC2-floci/
│   ├── Day3-en-EC2-floci.md
│   └── Day3-bn-EC2-floci.md
└── ...
```

**Naming convention:** `DayX-[topic]-floci/DayX-[en/bn]-[topic]-floci.md`

---

## প্রতিটি Day-এর ফাইলে যা থাকতে হবে

### ১. Header

```markdown
# দিন X: [টপিক নাম] (Bengali file)
# Day X: [Topic Name] (English file)
### Floci দিয়ে হাতে-কলমে শেখো (Git Bash)

> সব কমান্ড **Git Bash**-এ রান করতে হবে।
> কমান্ড রান করার আগে **Docker Desktop** অবশ্যই চালু থাকতে হবে।
```

---

### ২. পার্ট ১ — তত্ত্ব (Theory)

**অবশ্যই থাকতে হবে:**
- টপিকটা কী (সংজ্ঞা)
- কেন দরকার (real-world use case)
- মূল উপাদানসমূহ (বিস্তারিত, JSON example সহ যদি প্রযোজ্য হয়)
- Best Practices table
- User vs Role বা অন্য তুলনা (যদি প্রযোজ্য হয়)

---

### ৩. পার্ট ২ — হাতে-কলমে (Hands-On)

**প্রতিটি ধাপের format:**

```markdown
### ধাপ X — [ধাপের নাম]

**কেন করছি?**
[এই ধাপটা কেন করা হচ্ছে, real-world-এ কীভাবে কাজে লাগে]

```bash
command here
```

**প্রত্যাশিত output:**
```
expected output here
```

[যদি output না আসে: `(কোনো output আসবে না — এটা স্বাভাবিক, মানে সফল হয়েছে)`]

**যাচাই করো:**
```bash
verification command
```

**প্রত্যাশিত output:**
```
verification output
```
```

---

### ৪. প্রতিটি Day-এর শুরুতে যা থাকবে (Floci setup)

```bash
# Floci চালু করো
floci start --persist ./floci-data

# Environment variable সেট করো
eval $(floci env)

# যাচাই করো
echo $AWS_ENDPOINT_URL
```

**Expected output:**
```
http://localhost:4566
```

---

## গুরুত্বপূর্ণ নিয়মাবলী

### Command Rules

| নিষিদ্ধ (PowerShell) | সঠিক (Git Bash) |
|----------------------|----------------|
| `$env:VAR="value"` | `export VAR=value` বা `eval $(floci env)` |
| `echo $env:VAR` | `echo $VAR` |
| `"text" \| Out-File file.txt` | `echo "text" > file.txt` |
| `Get-Content file.txt` | `cat file.txt` |
| `cmd1; cmd2` | `cmd1 && cmd2` |
| `irm url \| iex` | `curl -fsSL url \| sh` |

### Floci-specific Rules

- **Account ID:** সবসময় `000000000000` (বারোটি শূন্য) — real AWS-এ real number থাকত
- **Endpoint:** `http://localhost:4566`
- **Start command:** সবসময় `floci start --persist ./floci-data` — `--persist` ছাড়া data হারিয়ে যায়
- **Env setup:** `eval $(floci env)` — Git Bash window বন্ধ করলে আবার দিতে হবে

### ARN Format in Floci

```
arn:aws:iam::000000000000:user/dev-user
arn:aws:iam::000000000000:group/Developers
arn:aws:iam::000000000000:role/MyRole
arn:aws:iam::000000000000:policy/MyPolicy
arn:aws:iam::aws:policy/AmazonEC2FullAccess   ← AWS managed policy (000000000000 না)
```

### Output Rules

- প্রতিটি command-এর পরে **প্রত্যাশিত output** দিতে হবে
- Output না আসলে লিখতে হবে: `(কোনো output আসবে না — এটা স্বাভাবিক, মানে সফল হয়েছে)`
- JSON output-এ dummy value ব্যবহার করো: `AKIAXXXXXXXXXXXXXXXX`, `AROAXXXXXXXXXXXXXXXXXX`
- Date হিসেবে `2026-07-01T00:00:00+00:00` ব্যবহার করো

---

### ৫. শেষে যা থাকবে

```markdown
## দ্রুত তথ্যসূত্র — [টপিক] কমান্ড চিট শিট
(table আকারে সব কমান্ড)

## আজকে যা তৈরি করলে
(tree diagram বা checklist)

## বাড়ির কাজ
(৩টি practice task)

## রিসোর্স
- Floci: https://floci.io
- Floci AWS services: https://floci.io/aws
- AWS official docs link
```

---

## File তৈরির চেকলিস্ট

নতুন Day-এর জন্য ফাইল তৈরি করার সময় নিশ্চিত করো:

- [ ] Bengali ফাইল আছে (`DayX-bn-topic-floci.md`)
- [ ] English ফাইল আছে (`DayX-en-topic-floci.md`)
- [ ] Theory section সম্পূর্ণ (কী, কেন, উপাদান, best practices)
- [ ] প্রতিটি ধাপে "কেন করছি" আছে
- [ ] সব command Git Bash compatible
- [ ] প্রতিটি command-এর নিচে expected output আছে
- [ ] `floci start --persist ./floci-data` দিয়ে শুরু
- [ ] `eval $(floci env)` দিয়ে env setup
- [ ] Cleanup/delete section আছে
- [ ] চিট শিট table আছে
- [ ] বাড়ির কাজ আছে

---

## Syllabus Reference

`syllabus.md` দেখো। প্রতিটি Day-এর জন্য সিলেবাসে দেওয়া topic follow করো।

| Day | Topic |
|-----|-------|
| 1 | Introduction + Floci Setup |
| 2 | IAM |
| 3 | EC2 Instances |
| 4 | AWS Networking (VPC) |
| 5 | AWS Security |
| 6 | Route 53 |
| 7 | Secure VPC + EC2 Project |
| 9 | Amazon S3 |
| 10 | AWS CLI |
| 11 | CloudFormation |
| 12 | CodeCommit |
| 13 | CodePipeline |
| 14 | CodeBuild |
| 15 | CodeDeploy |
| 16 | CloudWatch |
| 17 | Lambda |
| 18 | EventBridge |
| 19 | CloudFront |
| 20 | ECR |
| 21 | ECS |
| 22 | EKS |
| 23 | Secrets Manager / SSM |
| 24 | Terraform |
| 25 | CloudTrail + Config |
| 26 | ELB |
| 30 | RDS |
