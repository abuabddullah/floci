# Day 1: Floci Setup on Your PC

**Floci** is a free, open-source AWS emulator that runs locally on your computer.
You can practice all AWS services without a real AWS account or credit card — completely free, forever.

> **Video:** [YouTube Link — Coming Soon]

---

## What You Will Learn

- What Floci is and why we use it
- How to install Docker Desktop (prerequisite)
- How to install AWS CLI v2
- How to install and start Floci
- How to verify everything is working

---

## Prerequisites

Before installing Floci, you need two things on your PC:

1. **Docker Desktop** — Floci runs inside Docker
2. **AWS CLI v2** — To interact with Floci using AWS commands

---

## Step 1: Install Docker Desktop

Docker is required for Floci to run Lambda functions, RDS, and other services in real containers.

### Windows

1. Go to: [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
2. Click **"Download for Windows"**
3. Run the installer (`Docker Desktop Installer.exe`)
4. During installation, make sure **"Use WSL 2 instead of Hyper-V"** is checked
5. After installation, **restart your PC**
6. Open Docker Desktop from the Start Menu
7. Wait until you see **"Docker Desktop is running"** in the taskbar (whale icon)

### Mac

```bash
brew install --cask docker
```
Then open Docker from Applications and wait for it to start.

### Linux (Ubuntu/Debian)

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
```

### Verify Docker Installation

Open a terminal (PowerShell on Windows / Terminal on Mac/Linux) and run:

```bash
docker --version
```

Expected output (version may differ):
```
Docker version 27.x.x, build xxxxxxx
```

```bash
docker run hello-world
```

Expected output:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

---

## Step 2: Install AWS CLI v2

AWS CLI lets you run AWS commands (like `aws s3 ls`, `aws lambda list-functions`) that Floci will handle locally.

### Windows

1. Download the installer: [https://awscli.amazonaws.com/AWSCLIV2.msi](https://awscli.amazonaws.com/AWSCLIV2.msi)
2. Run `AWSCLIV2.msi`
3. Click **Next → Next → Install → Finish**

Or via PowerShell (run as Administrator):

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

### Verify AWS CLI Installation

```bash
aws --version
```

Expected output:
```
aws-cli/2.x.x Python/3.x.x ...
```

---

## Step 3: Install Floci

### Windows (PowerShell — Run as Administrator)

```powershell
iwr https://floci.io/install.ps1 | iex
```

**Alternative — Using Scoop (if you have Scoop installed):**

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

### Verify Floci Installation

```bash
floci --version
```

---

## Step 4: Start Floci

```bash
floci start
```

Expected output:
```
✓ Floci is running on http://localhost:4566
```

Floci runs on port **4566** — this is where all your AWS API calls will go.

> **To stop Floci later:**
> ```bash
> floci stop
> ```

---

## Step 5: Configure AWS CLI to Point to Floci

When you run `aws` commands, you need to tell the AWS CLI to talk to Floci (localhost:4566) instead of the real AWS.

### Windows (PowerShell — set for current session)

```powershell
$env:AWS_ENDPOINT_URL     = "http://localhost:4566"
$env:AWS_DEFAULT_REGION   = "us-east-1"
$env:AWS_ACCESS_KEY_ID    = "test"
$env:AWS_SECRET_ACCESS_KEY = "test"
```

### Mac / Linux / Git Bash (set for current session)

```bash
eval $(floci env)
```

This automatically sets all required environment variables.

Or manually:

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
```

> **Note:** `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` can be any dummy value (like `test`) because Floci does not require real credentials.

---

## Step 6: Verify Everything is Working

Run these commands to confirm Floci is working correctly.

### Check Floci health

```bash
floci status
```

Expected output:
```
✓ Floci is healthy
```

### Test S3

```bash
aws s3 mb s3://my-first-bucket
aws s3 ls
```

Expected output:
```
make_bucket: my-first-bucket
2026-06-28 00:00:00 my-first-bucket
```

### Test DynamoDB

```bash
aws dynamodb list-tables
```

Expected output:
```json
{
    "TableNames": []
}
```

### Test SQS

```bash
aws sqs create-queue --queue-name my-test-queue
aws sqs list-queues
```

Expected output:
```json
{
    "QueueUrls": [
        "http://localhost:4566/000000000000/my-test-queue"
    ]
}
```

If all three tests pass — **your Floci setup is complete!**

---

## Useful Floci Commands

| Command | What it does |
|---------|-------------|
| `floci start` | Start the Floci emulator |
| `floci stop` | Stop the emulator |
| `floci status` | Check if Floci is healthy |
| `floci logs --follow` | Watch real-time logs |
| `floci doctor` | Run diagnostics to find problems |
| `floci start --persist ./data` | Save your data between restarts |
| `floci snapshot save` | Take a snapshot of current state |
| `floci snapshot restore` | Restore a saved snapshot |

---

## Alternative: Run Floci with Docker (without CLI install)

If you prefer not to install the Floci CLI, you can run it directly with Docker:

```bash
docker run -d --name floci \
  -p 4566:4566 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  floci/floci:latest
```

Or with Docker Compose — create a file named `compose.yaml`:

```yaml
services:
  floci:
    image: floci/floci:latest
    ports:
      - "4566:4566"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

Then run:

```bash
docker compose up -d
```

---

## Troubleshooting

### Problem: Port 4566 is already in use

```bash
floci start --port 4567
```

Then update your environment variable:

```bash
export AWS_ENDPOINT_URL=http://localhost:4567
```

### Problem: Docker is not running

Make sure Docker Desktop is open and the whale icon is in your taskbar (Windows/Mac). On Linux:

```bash
sudo systemctl start docker
```

### Problem: `floci` command not found after install

Close and reopen your terminal (PowerShell/Terminal). The PATH needs to refresh.

On Windows, you may need to restart PowerShell or add Floci to PATH manually:

```powershell
$env:PATH += ";$env:USERPROFILE\.floci\bin"
```

### Problem: AWS CLI says "Unable to connect to endpoint"

Make sure you have set the environment variables from **Step 5** and that `floci start` is running.

```bash
floci status
```

---

## What's Next

- **Day 2:** IAM — Create users, groups, roles, and policies using Floci
- **Day 3:** EC2 — Launch virtual servers locally with Floci

---

## Resources

- Floci Official Website: [https://floci.io](https://floci.io)
- Floci AWS Services List: [https://floci.io/aws](https://floci.io/aws)
- GitHub Repository: [https://github.com/floci-io/floci](https://github.com/floci-io/floci)
- AWS CLI v2 Download: [https://aws.amazon.com/cli/](https://aws.amazon.com/cli/)
- Docker Desktop Download: [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
