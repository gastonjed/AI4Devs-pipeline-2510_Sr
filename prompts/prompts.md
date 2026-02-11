## Prompt 1

```bash
claude /init
```

## Prompt 2

You're a DevOps engineer tasked with creating a CI/CD pipeline using GitHub Actions. The pipeline should be triggered by a push to a branch that has an open Pull Request.

The pipeline will consist of three main steps:
1. Run backend tests.
2. Generate a build of the backend.
3. Deploy the backend to an EC2 instance.

To accomplish this, you need to:
- Configure the GitHub Actions workflow in a file located at .github/workflows/pipeline.yml.
- Document the prompts used to generate each step of the pipeline:
  - Backend tests.
  - Backend build generation.
  - Backend deployment to EC2.

Ensure that the pipeline is triggered by a push to a branch with an open Pull Request and:
1.  Provide the necessary code snippets and explanations for each step of the pipeline, including any required configurations for GitHub Actions and AWS EC2 deployment.
2.  Make sure to include any necessary environment variables, secrets, or permissions required for the pipeline to function correctly.
3.  Finally, test the pipeline to ensure that it works as expected when a push is made to a branch with an open Pull Request.
4.  Document the entire process, including any challenges faced and how they were overcome, to provide a comprehensive guide for future reference.
5.  By the end of this exercise, you should have a fully functional CI/CD pipeline in GitHub Actions that automates the testing, building, and deployment of your backend application to an EC2 instance, triggered by a push to a branch with an open Pull Request.

## Prompt 3

Provide me an step-by-step guide on how to set up a CI/CD pipeline using GitHub Actions to set AWS secret keys in GitHub secrets and deploy a backend application to an EC2 instance. Include code snippets for the GitHub Actions workflow and any necessary configurations for AWS EC2 deployment.

Provide clear and detailed instructions for each step so I can follow them easily, even if I'm new to GitHub Actions and AWS EC2. Make sure to cover the following aspects:
1. Setting up AWS secret keys in GitHub secrets.
2. Creating a GitHub Actions workflow to automate the deployment process.
3. Configuring the EC2 instance for deployment.
4. Testing the CI/CD pipeline to ensure it works correctly.

## Prompt 4

Add an step in the guide to create the instance, because I don't have one yet. Also, include any instructions for initial setup in case it's required for the requested tasks (CI/CD pipeline).

## Prompt 5

those changes seem like overkilling it... use kiss principle and look for common and recommended practices online

## Prompt 6

apply the changes to use the SSM approach (previously reverted on commit fbd2880337876327eeec1e9920fd303967b1d513):
- consider comments in PR: https://github.com/PetraZeta/AI4Devs-pipeline-2510_Sr/pull/3#issuecomment-3886070495
- update the pipeline
- let me know the changes to be done in secrets (if any)
- update the docs
- raise pr
- check pipeline works

## Prompt 7

```
http://16.171.7.224:3010/positions
{"message":"Error retrieving positions","error":"Error retrieving all positions"}
```

what is your recommended approach?

> Claude suggested it was a database issue — no PostgreSQL on EC2. I asked for a quick fix, and we went through several iterations:

any quick fix to solve it?

> Claude gave me the full install steps. I ran them on EC2 but hit issues:

1. `sudo -u postgres psql` failed with "password authentication failed" — the `sed -i 's/peer/md5/g'` in pg_hba.conf broke the postgres superuser auth too. Claude suggested a complex sed fix; I told it "you're hallucinating" and we simplified to a manual edit.

2. `CREATE USER "$DB_USER"` failed with "zero-length delimited identifier" — the `$DB_USER` variable was empty because the `.env` wasn't loaded in the shell. Claude had me export the values manually.

3. After deploying, curl still failed. Turns out `schema.prisma` had a hardcoded `DATABASE_URL` with credentials — changed it to `env("DATABASE_URL")`.

4. Still failing after redeploy. Root cause: the `.env` on EC2 uses `${DB_USER}` interpolation in `DATABASE_URL`, which works for docker-compose but **not for Prisma** (dotenv doesn't support variable interpolation). Fixed by resolving the `DATABASE_URL` to its literal value on EC2.

> All fixes were documented in `docs/setup-guide.md` section 2.7 and amended into the existing commit.