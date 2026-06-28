# Floci CLI Setup & AWS S3 Local Testing Guide (Windows)

কমান্ড রান করার আগে পিসিতে অবশ্যই Docker Desktop ব্যাকগ্রাউন্ডে চালু (Running) থাকতে হবে।
------------------------------
## 🛠️ ধাপ ১: PowerShell রান করুন

* Windows Start Menu থেকে PowerShell সার্চ করুন।
* ডান ক্লিক করে "Run as Administrator" হিসেবে ওপেন করুন [6c84b4005771].

------------------------------
## 📥 ধাপ ২: Floci CLI ইনস্টল করুন
নিচের কমান্ডটি দিয়ে ফ্লোসি ইনস্টল করুন:
```
irm https://floci.io/install.ps1 | iex
```
(ইনস্টলেশন শেষ হলে চলমান পাওয়ারশেল উইন্ডোটি বন্ধ (Close) করে দিন এবং পুনরায় নতুন একটি পাওয়ারশেল উইন্ডো Administrator হিসেবে ওপেন করুন।)
------------------------------
## 🟢 ধাপ ৩: Floci স্টার্ট ও এরর সহ রান করা
আপনার টার্মিনাল অনুযায়ী, ডকার ইমেজ ডাউনলোড এবং কন্টেইনার সচল করতে এই হুবহু কমান্ডটি দিন:
```
floci start; floci env | Invoke-Expression
```

* নোট: এই কমান্ডটি দিলে উইন্ডোজে export সংক্রান্ত কিছু লাল রঙের এরর (CommandNotFoundException) দেখাবে। এগুলো সম্পূর্ণ স্বাভাবিক, এগুলোকে ইগনোর করে পরের ধাপে চলে যান। কন্টেইনারটি ব্যাকগ্রাউন্ডে ঠিকই চালু হয়ে যাবে।

## 🟢 ধাপ ৩.2: Floci status
আপনার টার্মিনাল অনুযায়ী,  স্ট্যাটাস দেখতে এই হুবহু কমান্ডটি দিন:
```
floci status
```


------------------------------
## ⚙️ ধাপ ৪: Windows এনভায়রনমেন্ট ভেরিয়েবল সেট করুন
উইন্ডোজে লোকাল AWS কানেকশনের জন্য সরাসরি নিচের ৪টি লাইন একসাথে কপি করে পাওয়ারশেলে পেস্ট করুন এবং Enter চাপুন:
```
$env:AWS_ENDPOINT_URL="http://localhost.floci.io:4566"
$env:AWS_ACCESS_KEY_ID="test"
$env:AWS_SECRET_ACCESS_KEY="test"
$env:AWS_DEFAULT_REGION="us-east-1"
```
------------------------------
## 🔍 ধাপ ৫: ভেরিয়েবল ও কানেকশন চেক করা
ভেরিয়েবলগুলো ঠিকঠাক কাজ করছে কিনা দেখতে এই দুটি কমান্ড রান করুন:
```
# Check a single variable
echo $env:AWS_ENDPOINT_URL
# Check another variable
echo $env:AWS_ACCESS_KEY_ID
```
আউটপুট: স্ক্রিনে http://localhost.floci.io:4566 এবং test লেখা দুটি দেখতে পাবেন।
------------------------------
## 🧪 ধাপ ৬: বাকেট তৈরি ও ফাইল আপলোড টেস্ট (S3 Testing)
এখন লোকাল ক্লাউড ডিরেক্টরি সম্পূর্ণ প্রস্তুত। নিচের কমান্ডগুলো একে একে রান করে বাকেট ও ফাইল হ্যান্ডলিং টেস্ট করুন:
১. লোকাল বাকেট তৈরি করা:
```     
aws s3 mb s3://my-bucket
```

(আউটপুট দেখাবে: make_bucket: my-bucket)
২. ফাইল তৈরি করা:
``` 
"Why pay for S3 when floci is free? 🎉" | Out-File hello-floci.txt
```

৩. বাকেটে ফাইল আপলোড করা:
```
aws s3 cp hello-floci.txt s3://my-bucket/
```
(আউটপুট দেখাবে: upload: .\hello-floci.txt to s3://my-bucket/hello-floci.txt)

৪. বাকেট থেকে ফাইল ডাউনলোড ও রিড করা:
```
aws s3 cp s3://my-bucket/hello-floci.txt hello-back.txt; Get-Content hello-back.txt
```
ফাইনাল আউটপুট:
```
Why pay for S3 when floci is free? 🎉
```
------------------------------

