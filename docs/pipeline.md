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

**Purpose:** Copy the compiled backend to an EC2 instance via SCP and restart the application via SSH.

**Steps:**
1. Download the build artifact
2. Copy files to EC2 via SCP (`appleboy/scp-action`)
3. SSH into EC2, install production deps, generate Prisma client, restart PM2 (`appleboy/ssh-action`)

**Required EC2 setup:**
- SSH access (port 22 open, key pair configured)
- Node.js + npm + PM2 installed on the instance
- Application directory: `/opt/app/backend`

---

## Required GitHub Secrets

| Secret | Description |
|--------|-------------|
| `EC2_HOST` | EC2 instance public IP address (e.g., `13.53.123.45`) |
| `EC2_SSH_KEY` | Full contents of the `.pem` private key file |

The `GITHUB_TOKEN` secret is automatically provided by GitHub Actions.

---

## Challenges and Solutions

### 1. Triggering only on branches with open PRs
**Challenge:** GitHub Actions `push` trigger doesn't fire on the base repo for fork-based PRs.
**Solution:** Use the `pull_request` trigger targeting `main`. This fires on the base repo for both same-repo and fork PRs, on open/synchronize/reopen events.

### 2. Prisma Client needed for tests even without a database
**Challenge:** Tests mock `@prisma/client` but still need the generated types to compile.
**Solution:** Run `npx prisma generate` before tests. This generates the TypeScript types without requiring a database connection.

### 3. Deploying to EC2
**Challenge:** Need to transfer build artifacts and run commands on the remote instance.
**Solution:** Use `appleboy/scp-action` to copy files and `appleboy/ssh-action` to run deployment commands — the most common and straightforward approach for EC2 deployments from GitHub Actions.

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
