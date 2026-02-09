# CI/CD Pipeline Setup Guide

A step-by-step guide to configure GitHub Secrets, set up an EC2 instance, and test the pipeline end-to-end.

---

## Part 1: Setting Up GitHub Secrets

1. Go to your GitHub repository
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret** and add each:

| Name | Value |
|------|-------|
| `EC2_HOST` | Your EC2 instance's public IP address (e.g., `13.53.123.45`) |
| `EC2_SSH_KEY` | The **full contents** of your `.pem` private key file (see below) |

To get the SSH key value, run on your local machine:
```bash
cat ~/.ssh/lti-backend-key.pem
```
Copy the entire output (including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----`) and paste it as the `EC2_SSH_KEY` secret value.

You can find your EC2 public IP in **AWS Console** → **EC2** → **Instances** → select your instance → **Public IPv4 address**.

---

## Part 2: Creating and Configuring the EC2 Instance

### 2.1 Create a Key Pair (for SSH Access)

1. Go to **AWS Console** → **EC2** → **Key Pairs** (left sidebar under "Network & Security")
2. Click **Create key pair**
3. Name: `lti-backend-key`
4. Key pair type: **RSA**
5. Private key file format: **.pem** (for Linux/macOS) or **.ppk** (for PuTTY on Windows)
6. Click **Create key pair** — the `.pem` file downloads automatically
7. Secure the key file on your machine:

```bash
# Move it to your SSH directory
mv ~/Downloads/lti-backend-key.pem ~/.ssh/

# Restrict permissions (required by SSH)
chmod 400 ~/.ssh/lti-backend-key.pem
```

### 2.2 Create a Security Group

1. Go to **AWS Console** → **EC2** → **Security Groups** → **Create security group**
2. Name: `lti-backend-sg`
3. Description: `Security group for LTI backend on EC2`
4. VPC: select your default VPC
5. Add **Inbound rules**:

| Type | Port | Source | Purpose |
|------|------|--------|---------|
| SSH | 22 | 0.0.0.0/0 | SSH access (from your machine and GitHub Actions runners) |
| Custom TCP | 3010 | 0.0.0.0/0 | Backend API access |

6. Leave outbound rules as default (allow all)
7. Click **Create security group**

### 2.3 Launch the EC2 Instance

1. Go to **AWS Console** → **EC2** → **Launch Instance**
2. Configure:

| Setting | Value |
|---------|-------|
| **Name** | `lti-backend` |
| **AMI** | Amazon Linux 2023 (free tier eligible) |
| **Instance type** | `t2.micro` (free tier eligible) |
| **Key pair** | `lti-backend-key` (created in step 2.2) |
| **Security group** | Select existing → `lti-backend-sg` (created in step 2.2) |

3. Click **Launch Instance**
4. **Copy the public IP address** — you'll need it for the `EC2_HOST` GitHub secret

### 2.4 Connect via SSH

Wait ~1 minute for the instance to start, then:

```bash
# Find the public IP in AWS Console → EC2 → Instances → select your instance
# Or use the AWS CLI:
aws ec2 describe-instances \
  --instance-ids i-YOUR_INSTANCE_ID \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text

# Connect via SSH
ssh -i ~/.ssh/lti-backend-key.pem ec2-user@YOUR_PUBLIC_IP
```

> **Note:** The default username is `ec2-user` for Amazon Linux and `ubuntu` for Ubuntu AMIs.

### 2.5 Install Node.js, npm, and PM2

Once connected via SSH, run the following commands to set up the runtime environment:

```bash
# Update system packages
sudo yum update -y

# Install Node.js 20
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
sudo yum install -y nodejs

# Verify installation
node --version    # should show v20.x.x
npm --version

# Install PM2 globally (process manager to keep the app running)
sudo npm install -g pm2

# Configure PM2 to start on boot
pm2 startup
# Run the command that PM2 outputs (it will look like: sudo env PATH=... pm2 startup ...)

# Create the application directory
sudo mkdir -p /opt/app/backend
sudo chown ec2-user:ec2-user /opt/app/backend
```

### 2.6 (Optional) Set Up an Elastic IP

By default, the public IP changes every time the instance stops/starts. To get a fixed IP:

1. Go to **EC2** → **Elastic IPs** → **Allocate Elastic IP address**
2. Click **Allocate**
3. Select the new IP → **Actions** → **Associate Elastic IP address**
4. Select your `lti-backend` instance → **Associate**

Now your instance has a permanent public IP for SSH and API access.

---

## Part 3: The GitHub Actions Workflow

The workflow file is already created at `.github/workflows/pipeline.yml`. Here's what each job does:

```
push to branch (non-main)
  │
  ▼
check-pr ── has open PR? ──No──▶ skip all jobs
  │
  Yes
  ▼
test ── npm ci → prisma generate → npm test
  │
  Pass
  ▼
build ── npm ci → prisma generate → tsc → upload artifact
  │
  ▼
deploy ── download artifact → aws configure → ssm send-command → pm2 restart
```

The full workflow is documented in [pipeline.md](pipeline.md).

---

## Part 4: Testing the Pipeline

### 4.1 Test Locally First

```bash
# From the project root
cd backend

# Install dependencies
npm ci

# Generate Prisma Client (needed for types, no DB required)
npx prisma generate

# Run tests — all 4 should pass
npm test

# Build — should compile without errors
npm run build

# Verify output
ls dist/index.js    # should exist
```

### 4.2 Test the Full Pipeline on GitHub

**Step 1:** Commit and push your branch

```bash
git add .github/workflows/pipeline.yml
git commit -m "Add CI/CD pipeline for backend"
git push origin pipeline-GDT
```

**Step 2:** Open a Pull Request

```bash
# Using GitHub CLI
gh pr create --title "Add CI/CD pipeline" --body "Adds automated test, build, and deploy pipeline"

# Or do it from GitHub.com:
# Go to your repo → Pull requests → New pull request
# Set base: main, compare: pipeline-GDT
```

**Step 3:** The pipeline should trigger automatically since you pushed to a branch with an open PR.

**Step 4:** Monitor the run

```bash
# Using GitHub CLI
gh run list --branch pipeline-GDT
gh run watch    # live tail of the latest run
```

Or go to your repo on GitHub → **Actions** tab to see the run.

### 4.3 What to Expect

| Job | Expected result |
|-----|----------------|
| `check-pr` | Passes — finds the open PR |
| `test` | Passes — 4 test suites, 4 tests |
| `build` | Passes — compiles TypeScript to dist/ |
| `deploy` | Passes only if `EC2_HOST` and `EC2_SSH_KEY` secrets are configured and EC2 is set up |

### 4.4 Troubleshooting

**Pipeline doesn't trigger:**
- Make sure the PR is **open** (not draft, not closed)
- Make sure you're pushing to the PR's head branch, not to `main`

**`check-pr` says no open PR:**
- The branch name must exactly match. Check with `gh pr list --head your-branch-name`

**Tests fail:**
- Run `npm test` locally to reproduce. Tests mock Prisma so no DB is needed

**Build fails:**
- Run `npm run build` locally. Check for TypeScript errors

**Deploy fails with "Connection refused" or "Timeout":**
- Verify `EC2_HOST` secret contains the correct public IP
- Verify the instance is running
- Verify port 22 is open in the security group

**Deploy fails with "Permission denied":**
- Verify `EC2_SSH_KEY` contains the full `.pem` file contents (including BEGIN/END lines)
- Verify the key matches the key pair used when launching the instance

**Deploy fails with "npm ci" or "pm2" errors:**
- SSH into the instance and verify Node.js and PM2 are installed
- Verify `/opt/app/backend` exists and is owned by `ec2-user`
