# CI/CD Pipeline Setup Guide

A step-by-step guide to configure AWS credentials, set up an EC2 instance with SSM, and test the pipeline end-to-end.

---

## Part 1: Setting Up AWS Credentials

### 1.1 Create an IAM User for GitHub Actions

1. Go to **AWS Console** → **IAM** → **Users** → **Create user**
2. Name: `github-actions-deployer`
3. Select **Attach policies directly** → click **Create policy** with this JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SSMDeploy",
      "Effect": "Allow",
      "Action": [
        "ssm:SendCommand",
        "ssm:GetCommandInvocation"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3DeployArtifacts",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::lti-backend-deploys/*"
    }
  ]
}
```

4. Name the policy `GitHubActionsSSMAndS3Deploy` and attach it to the user
5. Go to the user → **Security credentials** → **Create access key**
6. Select **Third-party service** as the use case
7. Copy the **Access Key ID** and **Secret Access Key** (you won't see the secret again)

### 1.2 Create an S3 Bucket for Deploy Artifacts

1. Go to **AWS Console** → **S3** → **Create bucket**
2. Bucket name: `lti-backend-deploys`
3. Region: `eu-north-1` (same as your EC2 instance)
4. Keep all other defaults (private, no versioning needed)
5. Click **Create bucket**

> **Optional:** Add a lifecycle rule to automatically delete objects older than 1 day (S3 → bucket → Management → Create lifecycle rule → Expire current versions after 1 day).

### 1.3 Add Secrets to GitHub

1. Go to your GitHub repository
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret** and add each:

| Name | Value |
|------|-------|
| `AWS_ACCESS_ID` | The Access Key ID from step 1.1 |
| `AWS_ACCESS_KEY` | The Secret Access Key from step 1.1 |
| `EC2_INSTANCE` | Your EC2 instance ID (e.g., `i-0abcdef1234567890`) |

You can find your EC2 instance ID in **AWS Console** → **EC2** → **Instances** — it's the `i-` prefixed value in the Instance ID column.

---

## Part 2: Creating and Configuring the EC2 Instance

### 2.1 Create an IAM Role for the EC2 Instance

The instance needs an IAM role so that AWS Systems Manager (SSM) can communicate with it and the instance can download deploy artifacts from S3. This must be done **before** launching the instance.

1. Go to **AWS Console** → **IAM** → **Roles** → **Create role**
2. Select trusted entity: **AWS service**
3. Use case: **EC2** → click **Next**
4. Search for and check: `AmazonSSMManagedInstanceCore` → click **Next**
5. Role name: `EC2-SSM-Role`
6. Click **Create role**
7. Open the newly created role → **Add permissions** → **Create inline policy** with this JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3DeployDownload",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::lti-backend-deploys/*"
    }
  ]
}
```

8. Name the inline policy `S3DeployDownload`

### 2.2 Create a Key Pair (for SSH Access)

A key pair is useful for initial instance setup and debugging. The CI/CD pipeline itself does not use SSH.

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

### 2.3 Create a Security Group

1. Go to **AWS Console** → **EC2** → **Security Groups** → **Create security group**
2. Name: `lti-backend-sg`
3. Description: `Security group for LTI backend on EC2`
4. VPC: select your default VPC
5. Add **Inbound rules**:

| Type | Port | Source | Purpose |
|------|------|--------|---------|
| Custom TCP | 3010 | 0.0.0.0/0 | Backend API access |

> **Note:** No SSH port (22) is needed for the pipeline — SSM communicates via outbound HTTPS (port 443), which is allowed by default. You may optionally add an SSH rule restricted to your own IP for manual debugging.

6. Leave outbound rules as default (allow all)
7. Click **Create security group**

### 2.4 Launch the EC2 Instance

1. Go to **AWS Console** → **EC2** → **Launch Instance**
2. Configure:

| Setting | Value |
|---------|-------|
| **Name** | `lti-backend` |
| **AMI** | Amazon Linux 2023 (free tier eligible) |
| **Instance type** | `t2.micro` (free tier eligible) |
| **Key pair** | `lti-backend-key` (created in step 2.2) |
| **Security group** | Select existing → `lti-backend-sg` (created in step 2.3) |
| **IAM instance profile** | `EC2-SSM-Role` (under **Advanced details**, created in step 2.1) |

3. Click **Launch Instance**
4. **Copy the Instance ID** (e.g., `i-0abcdef1234567890`) — you'll need it for the `EC2_INSTANCE` GitHub secret

### 2.5 Connect via SSH (for Initial Setup)

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

### 2.6 Install Node.js, npm, and PM2

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

### 2.7 Install and Configure PostgreSQL

The backend requires a PostgreSQL database. Install it directly on the instance:

First, copy the `.env` file from your local project root to EC2:

```bash
# From your local machine
scp -i ~/.ssh/lti-backend-key.pem .env ec2-user@YOUR_PUBLIC_IP:/opt/app/backend/.env
```

> **Important:** The local `.env` uses variable interpolation in `DATABASE_URL` (e.g., `${DB_USER}`). This works for docker-compose but **not for Prisma**, which reads `.env` via dotenv without interpolation. After copying, fix it on EC2:

```bash
# On EC2 — resolve DATABASE_URL to its actual value
cd /opt/app/backend
export $(grep -E '^(DB_USER|DB_PASSWORD|DB_NAME|DB_PORT)=' .env | xargs)
sed -i "s|^DATABASE_URL=.*|DATABASE_URL=\"postgresql://${DB_USER}:${DB_PASSWORD}@localhost:${DB_PORT}/${DB_NAME}\"|" .env
```

Then on EC2, install PostgreSQL and create the database using the values from `.env`:

```bash
# Install PostgreSQL 15
sudo dnf install postgresql15-server postgresql15 -y

# Initialize and start the database
sudo postgresql-setup --initdb
sudo systemctl enable postgresql
sudo systemctl start postgresql

# Load the DB credentials from .env
export $(grep -E '^(DB_USER|DB_PASSWORD|DB_NAME)=' /opt/app/backend/.env | xargs)

# Create the database user and database
sudo -u postgres psql -c "CREATE USER \"$DB_USER\" WITH PASSWORD '$DB_PASSWORD';"
sudo -u postgres psql -c "CREATE DATABASE \"$DB_NAME\" OWNER \"$DB_USER\";"

# Enable password authentication for host connections
sudo sed -i 's/ident/md5/g' /var/lib/pgsql/data/pg_hba.conf
sudo systemctl restart postgresql

# Run migrations and seed
cd /opt/app/backend
npx prisma migrate deploy
npx prisma db seed

# Restart the app
pm2 restart backend
```

> **Note:** The `DATABASE_URL` is read from the `.env` file at runtime. This is a one-time setup — subsequent deployments via the pipeline only restart the app, they don't touch the database.

### 2.8 Verify SSM Agent is Running

SSM Agent comes pre-installed on Amazon Linux 2023. Verify it's active:

```bash
sudo systemctl status amazon-ssm-agent
```

You should see `active (running)`. If not:

```bash
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

### 2.9 Verify SSM Connectivity from AWS Console

1. Go to **AWS Console** → **Systems Manager** → **Fleet Manager**
2. Your instance should appear with status **Online**
3. If it doesn't appear after a few minutes, check:
   - The `EC2-SSM-Role` IAM instance profile is attached (EC2 → Instance → Actions → Security → Modify IAM role)
   - The SSM Agent is running (step 2.7)
   - The instance has internet access (default VPC provides this)
   - The security group allows outbound HTTPS (port 443) — default outbound rules allow all

### 2.10 (Optional) Set Up an Elastic IP

By default, the public IP changes every time the instance stops/starts. To get a fixed IP:

1. Go to **EC2** → **Elastic IPs** → **Allocate Elastic IP address**
2. Click **Allocate**
3. Select the new IP → **Actions** → **Associate Elastic IP address**
4. Select your `lti-backend` instance → **Associate**

Now your instance has a permanent public IP for API access.

---

## Part 3: The GitHub Actions Workflow

The workflow file is already created at `.github/workflows/ci.yaml`. Here's what each job does:

```
pull request opened/updated (targeting main)
  │
  ▼
test ── npm ci → prisma generate → npm test
  │
  Pass
  ▼
build ── npm ci → prisma generate → tsc → upload artifact
  │
  ▼
deploy ── download artifact → tar + upload to S3 → SSM send-command → EC2 downloads from S3, deploys, restarts PM2
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
git add .github/workflows/ci.yaml
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
| `test` | Passes — 4 test suites, 4 tests |
| `build` | Passes — compiles TypeScript to dist/ |
| `deploy` | Passes only if AWS secrets are configured and EC2 is set up with SSM |

### 4.4 Troubleshooting

**Pipeline doesn't trigger:**
- Make sure the PR is **open** (not draft, not closed)
- Make sure you're pushing to the PR's head branch, not to `main`

**Tests fail:**
- Run `npm test` locally to reproduce. Tests mock Prisma so no DB is needed

**Build fails:**
- Run `npm run build` locally. Check for TypeScript errors

**Deploy fails with "Instance not found":**
- Verify `EC2_INSTANCE` secret contains just the instance ID (e.g., `i-0abcdef1234567890`)
- Verify the instance is running

**Deploy fails with "SSM agent not available":**
- Check the IAM instance profile has `AmazonSSMManagedInstanceCore`
- Verify SSM Agent is running: `sudo systemctl status amazon-ssm-agent`
- Verify the instance appears in **Systems Manager → Fleet Manager**

**Deploy fails with "Access Denied":**
- Verify `AWS_ACCESS_ID` and `AWS_ACCESS_KEY` are correct
- Verify the IAM user has `ssm:SendCommand` and `ssm:GetCommandInvocation` permissions
- Verify the `aws-region` in the workflow matches your EC2 instance's region

**Deploy fails downloading from S3:**
- Verify the EC2 instance role has `s3:GetObject` permission on `arn:aws:s3:::lti-backend-deploys/*`
- Verify the S3 bucket `lti-backend-deploys` exists in the same region (`eu-north-1`)
- SSH into the instance and test: `aws s3 ls s3://lti-backend-deploys/`

**Deploy fails with "npm ci" or "pm2" errors:**
- SSH into the instance and verify Node.js and PM2 are installed
- Verify `/opt/app/backend` exists and is owned by `ec2-user`
