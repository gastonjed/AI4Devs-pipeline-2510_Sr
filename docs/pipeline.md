# CI/CD Pipeline Documentation

## Overview

The pipeline is defined in `.github/workflows/ci.yaml` and runs on every pull request targeting `main`. It triggers on PR open, synchronize (new pushes), and reopen events — working for both same-repo and fork-based PRs.

It consists of three sequential jobs:

```
test → build → deploy
```

---

## Job 1: `test` — Run Backend Tests

**Purpose:** Run the Jest test suite for the backend to catch regressions before building.

**Steps:**
1. Checkout code
2. Setup Node.js 20 with npm cache
3. `npm ci` — clean install of dependencies
4. `npx prisma generate` — generate Prisma Client types
5. `npm test` — run Jest

**Why `npx prisma generate` is needed even without a database:**
The tests import from `@prisma/client` for type mocking. Without generating the client, TypeScript compilation (done by ts-jest) fails because the generated types don't exist.

---

## Job 2: `build` — Compile TypeScript

**Purpose:** Compile the TypeScript backend to JavaScript and upload the artifact for deployment.

**Steps:**
1. Checkout + setup Node.js (same as test job)
2. `npm ci` + `npx prisma generate`
3. `npm run build` — compiles TypeScript → `dist/`
4. Upload artifact containing: `dist/`, `package.json`, `package-lock.json`, `prisma/`

**Why artifacts include `prisma/`:**
The EC2 server needs the Prisma schema to run `npx prisma generate` and produce the client library matching the server's platform.

---

## Job 3: `deploy` — Deploy to EC2

**Purpose:** Upload the compiled backend to S3, then deploy to the EC2 instance via AWS Systems Manager (SSM).

**Steps:**
1. Download the build artifact
2. Configure AWS credentials via `aws-actions/configure-aws-credentials@v4`
3. Package artifacts as `deploy.tar.gz` and upload to S3
4. Send deployment commands to EC2 via SSM `AWS-RunShellScript` (download from S3, extract, install deps, restart PM2)
5. Wait for completion and verify success
6. Clean up S3 artifact

**Required EC2 setup:**
- SSM Agent running on the instance (pre-installed on Amazon Linux 2023)
- IAM instance profile with `AmazonSSMManagedInstanceCore` policy and S3 read access
- Node.js + npm + PM2 installed on the instance
- PostgreSQL with the `LTIdb` database (see [setup-guide.md](setup-guide.md#27-install-and-configure-postgresql))
- Application directory: `/opt/app/backend`

### Why SSM instead of SSH

| Aspect | SSH/SCP | SSM |
|--------|---------|-----|
| Authentication | Private key in GitHub Secrets | IAM credentials |
| Network | Port 22 must be open to the internet | No inbound ports required |
| Key rotation | Manual | Managed through IAM |
| Auditability | None | CloudTrail command history |
| File transfer | SCP (direct) | Via S3 (requires a bucket) |
| Cost | Free | Free (< 1000 commands/month) |

### SSM Prerequisites

- **SSM Agent**: Pre-installed on Amazon Linux 2023. Verify with `sudo systemctl status amazon-ssm-agent`
- **IAM Instance Role**: Must have `AmazonSSMManagedInstanceCore` managed policy + `s3:GetObject` on the deploy bucket
- **Network**: Instance must have outbound HTTPS (port 443) — default VPC provides this
- **Fleet Manager**: Instance should appear as **Online** in AWS Console → Systems Manager → Fleet Manager

---

## Required GitHub Secrets

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_ID` | AWS IAM access key ID with SSM and S3 permissions |
| `AWS_ACCESS_KEY` | AWS IAM secret access key |
| `EC2_INSTANCE` | EC2 instance ID (e.g., `i-0abcdef1234567890`) |

The `GITHUB_TOKEN` secret is automatically provided by GitHub Actions.

---

## Required AWS IAM Permissions

### IAM User (for GitHub Actions)

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

### EC2 Instance Role

Attach the managed policy `AmazonSSMManagedInstanceCore`, plus this inline policy for S3 access:

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

---

## Challenges and Solutions

### 1. Triggering only on branches with open PRs
**Challenge:** GitHub Actions `push` trigger doesn't fire on the base repo for fork-based PRs.
**Solution:** Use the `pull_request` trigger targeting `main`. This fires on the base repo for both same-repo and fork PRs, on open/synchronize/reopen events.

### 2. Prisma Client needed for tests even without a database
**Challenge:** Tests mock `@prisma/client` but still need the generated types to compile.
**Solution:** Run `npx prisma generate` before tests. This generates the TypeScript types without requiring a database connection.

### 3. Deploying without SSH keys
**Challenge:** Managing SSH private keys in GitHub Secrets is error-prone and a security risk (requires port 22 open to the internet).
**Solution:** Use AWS Systems Manager (SSM) `SendCommand` API for remote execution (authenticates via IAM) and S3 as an intermediary for file transfer. No SSH keys or open ports needed.

### 4. Artifact passing between jobs
**Challenge:** Each job runs on a fresh runner, so build output from the `build` job isn't available in `deploy`.
**Solution:** Use `actions/upload-artifact` and `actions/download-artifact` to pass the compiled code between jobs.

---

## Testing the Pipeline

To test locally before pushing:

```bash
# 1. Run tests
cd backend && npm ci && npx prisma generate && npm test

# 2. Build
npm run build

# 3. Verify dist/ was created
ls dist/
```

To test the full pipeline:
1. Create a feature branch: `git checkout -b feature/test-pipeline`
2. Make a small change and push
3. Open a Pull Request targeting `main` — the pipeline triggers automatically
4. Push additional commits — the pipeline re-triggers on each push
5. Check the Actions tab for the run results
