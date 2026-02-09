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

