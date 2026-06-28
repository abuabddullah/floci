# দিন ১: তোমার PC-তে Floci সেটআপ করো

**Floci** হলো একটি ফ্রি, ওপেন-সোর্স AWS emulator যেটা তোমার কম্পিউটারে locally চলে।
Real AWS account বা credit card ছাড়াই সমস্ত AWS সার্ভিস practice করা যাবে — সম্পূর্ণ বিনামূল্যে, চিরকালের জন্য।

> **ভিডিও:** [YouTube লিংক — শীঘ্রই আসছে]

---

## আজকে যা শিখবে

- Floci কী এবং কেন আমরা এটা ব্যবহার করি
- Docker Desktop কীভাবে install করতে হয় (পূর্বশর্ত)
- AWS CLI v2 কীভাবে install করতে হয়
- Floci কীভাবে install ও start করতে হয়
- সবকিছু ঠিকঠাক কাজ করছে কিনা কীভাবে যাচাই করতে হয়

---

## পূর্বশর্ত (Prerequisites)

Floci install করার আগে তোমার PC-তে দুটো জিনিস লাগবে:

1. **Docker Desktop** — Floci Docker-এর ভেতরে চলে
2. **AWS CLI v2** — AWS কমান্ড দিয়ে Floci-এর সাথে কথা বলার জন্য

---

## ধাপ ১: Docker Desktop Install করো

Floci-এর Lambda, RDS এবং অন্যান্য সার্ভিস real container-এ চালানোর জন্য Docker দরকার।

### Windows

1. এই লিংকে যাও: [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
2. **"Download for Windows"** বাটনে ক্লিক করো
3. Installer চালাও (`Docker Desktop Installer.exe`)
4. Installation-এর সময় **"Use WSL 2 instead of Hyper-V"** চেক করা আছে কিনা নিশ্চিত করো
5. Installation শেষ হলে **PC restart করো**
6. Start Menu থেকে Docker Desktop খোলো
7. Taskbar-এ whale আইকন দেখলে এবং **"Docker Desktop is running"** লেখা দেখলে বুঝবে চালু হয়ে গেছে

### Mac

```bash
brew install --cask docker
```
তারপর Applications থেকে Docker খোলো এবং চালু হওয়ার জন্য অপেক্ষা করো।

### Linux (Ubuntu/Debian)

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
```

### Docker Installation যাচাই করো

Terminal খোলো (Windows-এ PowerShell, Mac/Linux-এ Terminal) এবং রান করো:

```bash
docker --version
```

প্রত্যাশিত output (version নম্বর আলাদা হতে পারে):
```
Docker version 27.x.x, build xxxxxxx
```

```bash
docker run hello-world
```

প্রত্যাশিত output:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

---

## ধাপ ২: AWS CLI v2 Install করো

AWS CLI দিয়ে তুমি AWS কমান্ড চালাতে পারবে (যেমন `aws s3 ls`, `aws lambda list-functions`) এবং Floci সেগুলো locally handle করবে।

### Windows

1. Installer download করো: [https://awscli.amazonaws.com/AWSCLIV2.msi](https://awscli.amazonaws.com/AWSCLIV2.msi)
2. `AWSCLIV2.msi` ফাইলটি চালাও
3. **Next → Next → Install → Finish** ক্লিক করো

অথবা PowerShell-এ (Administrator হিসেবে চালাও):

```powershell
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi /quiet
```

### Mac

```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

### Linux (Ubuntu/Debian)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### AWS CLI Installation যাচাই করো

```bash
aws --version
```

প্রত্যাশিত output:
```
aws-cli/2.x.x Python/3.x.x ...
```

---

## ধাপ ৩: Floci Install করো

### Windows (PowerShell — Administrator হিসেবে চালাও)

```powershell
iwr https://floci.io/install.ps1 | iex
```

**বিকল্প — Scoop ব্যবহার করে (যদি Scoop থাকে):**

```powershell
scoop bucket add floci https://github.com/floci-io/scoop-floci
scoop install floci
```

### Mac (Homebrew)

```bash
brew install floci-io/floci/floci
```

### Linux / Mac / Windows Git Bash (curl)

```bash
curl -fsSL https://floci.io/install.sh | sh
```

### Floci Installation যাচাই করো

```bash
floci --version
```

---

## ধাপ ৪: Floci চালু করো

```bash
floci start
```

প্রত্যাশিত output:
```
✓ Floci is running on http://localhost:4566
```

Floci পোর্ট **4566**-এ চলে — তোমার সমস্ত AWS API call এখানে আসবে।

> **পরে Floci বন্ধ করতে চাইলে:**
> ```bash
> floci stop
> ```

---

## ধাপ ৫: AWS CLI-কে Floci-এর দিকে Point করো

`aws` কমান্ড চালানোর সময় AWS CLI-কে বলতে হবে যে সে real AWS-এ না গিয়ে Floci (localhost:4566)-এ যাবে।

### Windows (PowerShell — এই session-এর জন্য)

```powershell
$env:AWS_ENDPOINT_URL      = "http://localhost:4566"
$env:AWS_DEFAULT_REGION    = "us-east-1"
$env:AWS_ACCESS_KEY_ID     = "test"
$env:AWS_SECRET_ACCESS_KEY = "test"
```

### Mac / Linux / Git Bash (এই session-এর জন্য)

```bash
eval $(floci env)
```

এই একটা কমান্ডেই সব environment variable সেট হয়ে যাবে।

অথবা manually:

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
```

> **মনে রেখো:** `AWS_ACCESS_KEY_ID` এবং `AWS_SECRET_ACCESS_KEY`-এর মান যেকোনো কিছু দেওয়া যাবে (যেমন `test`)। Floci-এর real credential দরকার নেই।

---

## ধাপ ৬: সবকিছু ঠিকঠাক কাজ করছে কিনা যাচাই করো

নিচের কমান্ডগুলো রান করে নিশ্চিত হও যে Floci সঠিকভাবে কাজ করছে।

### Floci-এর স্বাস্থ্য পরীক্ষা করো

```bash
floci status
```

প্রত্যাশিত output:
```
✓ Floci is healthy
```

### S3 পরীক্ষা করো

```bash
aws s3 mb s3://my-first-bucket
aws s3 ls
```

প্রত্যাশিত output:
```
make_bucket: my-first-bucket
2026-06-28 00:00:00 my-first-bucket
```

### DynamoDB পরীক্ষা করো

```bash
aws dynamodb list-tables
```

প্রত্যাশিত output:
```json
{
    "TableNames": []
}
```

### SQS পরীক্ষা করো

```bash
aws sqs create-queue --queue-name my-test-queue
aws sqs list-queues
```

প্রত্যাশিত output:
```json
{
    "QueueUrls": [
        "http://localhost:4566/000000000000/my-test-queue"
    ]
}
```

তিনটি test-ই পাস করলে — **তোমার Floci সেটআপ সম্পন্ন!**

---

## Floci-এর দরকারী কমান্ডসমূহ

| কমান্ড | কী করে |
|--------|--------|
| `floci start` | Floci emulator চালু করে |
| `floci stop` | Emulator বন্ধ করে |
| `floci status` | Floci ঠিকঠাক চলছে কিনা দেখায় |
| `floci logs --follow` | Real-time log দেখায় |
| `floci doctor` | সমস্যা খুঁজে বের করে |
| `floci start --persist ./data` | Restart-এর পরেও data রেখে দেয় |
| `floci snapshot save` | বর্তমান অবস্থার snapshot নেয় |
| `floci snapshot restore` | সংরক্ষিত snapshot ফিরিয়ে আনে |

---

## বিকল্প পদ্ধতি: Docker দিয়ে Floci চালাও (CLI ছাড়া)

Floci CLI install না করতে চাইলে সরাসরি Docker দিয়েও চালানো যায়:

```bash
docker run -d --name floci \
  -p 4566:4566 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  floci/floci:latest
```

অথবা Docker Compose দিয়ে — `compose.yaml` নামে একটি ফাইল তৈরি করো:

```yaml
services:
  floci:
    image: floci/floci:latest
    ports:
      - "4566:4566"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

তারপর রান করো:

```bash
docker compose up -d
```

---

## সমস্যা সমাধান (Troubleshooting)

### সমস্যা: পোর্ট 4566 ইতিমধ্যে ব্যবহৃত হচ্ছে

```bash
floci start --port 4567
```

তারপর environment variable আপডেট করো:

```bash
export AWS_ENDPOINT_URL=http://localhost:4567
```

### সমস্যা: Docker চলছে না

Docker Desktop খোলা আছে কিনা এবং taskbar-এ whale আইকন দেখা যাচ্ছে কিনা নিশ্চিত করো (Windows/Mac)। Linux-এ:

```bash
sudo systemctl start docker
```

### সমস্যা: Install-এর পরেও `floci` কমান্ড পাওয়া যাচ্ছে না

Terminal বন্ধ করে আবার খোলো (PowerShell/Terminal)। PATH আপডেট হতে নতুন session দরকার।

Windows-এ manually PATH যোগ করতে হলে:

```powershell
$env:PATH += ";$env:USERPROFILE\.floci\bin"
```

### সমস্যা: AWS CLI বলছে "Unable to connect to endpoint"

নিশ্চিত করো যে **ধাপ ৫**-এর environment variable সেট করা হয়েছে এবং `floci start` চলছে।

```bash
floci status
```

---

## পরবর্তী পদক্ষেপ

- **দিন ২:** IAM — Floci ব্যবহার করে user, group, role এবং policy তৈরি করো
- **দিন ৩:** EC2 — Floci দিয়ে locally virtual server চালাও

---

## রিসোর্স

- Floci অফিসিয়াল ওয়েবসাইট: [https://floci.io](https://floci.io)
- Floci AWS সার্ভিস তালিকা: [https://floci.io/aws](https://floci.io/aws)
- GitHub Repository: [https://github.com/floci-io/floci](https://github.com/floci-io/floci)
- AWS CLI v2 Download: [https://aws.amazon.com/cli/](https://aws.amazon.com/cli/)
- Docker Desktop Download: [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
