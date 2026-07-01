# Floci CLI Setup & AWS S3 Local Testing Guide (Windows)

কমান্ড রান করার আগে পিসিতে অবশ্যই Docker Desktop ব্যাকগ্রাউন্ডে চালু (Running) থাকতে হবে।

---

## ধাপ ১: PowerShell রান করুন

- Windows Start Menu থেকে PowerShell সার্চ করুন।
- ডান ক্লিক করে "Run as Administrator" হিসেবে ওপেন করুন।

---

## ধাপ ২: Floci CLI ইনস্টল করুন

নিচের কমান্ডটি দিয়ে Floci ইনস্টল করুন:

```powershell
irm https://floci.io/install.ps1 | iex
```

ইনস্টলেশন শেষ হলে চলমান PowerShell উইন্ডোটি বন্ধ করে দিন এবং পুনরায় নতুন একটি PowerShell উইন্ডো Administrator হিসেবে ওপেন করুন।

---

## ধাপ ৩: Floci স্টার্ট করুন

ডকার ইমেজ ডাউনলোড এবং কন্টেইনার সচল করতে এই কমান্ডটি দিন:

```powershell
floci start --persist ./floci-data; floci env | Invoke-Expression
```

> **`--persist ./floci-data` কেন দিলাম?**
> Floci default-এ memory-তে চলে — বন্ধ করলেই সব data (S3 bucket, IAM user, ইত্যাদি) মুছে যায়।
> এই flag দিলে `floci-data` folder-এ সব data save থাকে, পরের session-এও পাওয়া যাবে।

**নোট:** এই কমান্ডটি দিলে Windows-এ export সংক্রান্ত কিছু লাল রঙের error (CommandNotFoundException) দেখাবে। এগুলো সম্পূর্ণ স্বাভাবিক — ignore করে পরের ধাপে চলে যান। Container ব্যাকগ্রাউন্ডে ঠিকই চালু হয়ে যাবে।

---

## ধাপ ৩.২: Floci status যাচাই করুন

```powershell
floci status
```

প্রত্যাশিত output:

```
Floci is healthy
```

---

## ধাপ ৪: Environment Variable সেট করুন

Windows-এ local AWS connection-এর জন্য নিচের ৪টি লাইন একসাথে কপি করে PowerShell-এ paste করুন এবং Enter চাপুন:

```bash
export AWS_ENDPOINT_URL="http://localhost:4566"
export AWS_ACCESS_KEY_ID="test"
export AWS_SECRET_ACCESS_KEY="test"
export AWS_DEFAULT_REGION="us-east-1"
```

> **গুরুত্বপূর্ণ:** PowerShell window বন্ধ করলে এই variable গুলো হারিয়ে যাবে। নতুন session-এ এসে আবার set করতে হবে।

---

## ধাপ ৫: Variable ও Connection যাচাই করুন

```bash
echo $env:AWS_ENDPOINT_URL
echo $env:AWS_ACCESS_KEY_ID
```

প্রত্যাশিত output:

```
http://localhost:4566
test
```

---

## ধাপ ৬: S3 Testing — Bucket তৈরি ও ফাইল আপলোড

এখন local cloud সম্পূর্ণ প্রস্তুত। নিচের কমান্ডগুলো একে একে রান করুন:

**১. Local bucket তৈরি করুন:**

```bash
aws s3 mb s3://my-bucket
```

আউটপুট: `make_bucket: my-bucket`

**২. ফাইল তৈরি করুন:**

```bash
echo "Why pay for S3 when floci is free?" > hello-floci.txt
```

**৩. Bucket-এ ফাইল আপলোড করুন:**

```bash
aws s3 cp Day1-Setup-floci/hello-floci.txt s3://my-bucket/
```

আউটপুট: `upload: .\hello-floci.txt to s3://my-bucket/hello-floci.txt`

**৪. Bucket থেকে ফাইল ডাউনলোড ও read করুন:**

```bash
aws s3 cp s3://my-bucket/hello-floci.txt Day1-Setup-floci/hello-back.txt; cat Day1-Setup-floci/hello-back.txt
```

আউটপুট:

```
Why pay for S3 when floci is free?
```

সব ঠিকঠাক আসলে — **তোমার Floci সেটআপ সম্পন্ন!**

---

## দরকারী কমান্ডসমূহ

| কমান্ড                               | কী করে                        |
| ------------------------------------ | ----------------------------- |
| `floci start --persist ./floci-data` | Data save রেখে Floci চালু করে |
| `floci stop`                         | Floci বন্ধ করে                |
| `floci status`                       | Floci চলছে কিনা দেখায়        |
| `floci logs --follow`                | Real-time log দেখায়          |

---

## পরবর্তী পদক্ষেপ

- **দিন ২:** IAM — Floci দিয়ে user, group, role এবং policy তৈরি করো
