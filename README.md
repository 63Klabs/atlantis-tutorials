# Serverless Deployments using 63K Atlantis Tutorials

Tutorials and walk-throughs for deploying serverless applications using CloudFormation and Severless Application Model templates provided by 63Klabs.

> Note: Though you can create and seed GitHub repositories using the scripts provided, the pipeline templates do not yet incorporate using anything other than CodeCommit as a source. Unfortunately, if you do not have CodeCommit available in your AWS account, you will not be able to complete the tutorials at this time. I'm working to get templates that utilize GitHub as a source.

## Before You Get Started

If you have not already completed [Serverless 8 Ball Example](https://github.com/chadkluck/serverless-sam-8ball-example) by [chadkluck](https://github.com/chadkluck) it is recommended you do so before continuing.

A variety of tutorials are available in the wild, and should be explored, to gain an understanding of what serverless is, how to deploy applications, and how they differ from traditional "server" applications.

The templates and tutorials developed by Chad/63Klabs and code-named Atlantis were created to fill the gap between standard tutorials and understanding and producing near-production ready applications using the Serverless Application Model (SAM).

Building on the concepts of Serverless 8 Ball, each tutorial described here goes into automating deployments using Infrastructure as Code (IaC) by incorporating pipelines, build scripts, and CloudFormation templates.

The templates intend to demonstrate implementing best practices using the [AWS Well Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html):

1. Operational excellence
2. Security
3. Reliability
4. Performance efficiency
5. Cost optimization
6. Sustainability

And:

- [CloudFormation Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html)
- [Best Practices for Organizing Larger Serverless Applications](https://aws.amazon.com/blogs/compute/best-practices-for-organizing-larger-serverless-applications/)

How:

- **Infrastructure as Code (IaC):** Utilize CI/CD, pipelines, build scripts, configuration files, and version control to document and deploy replicable infrastructure
- **Principle of Least Privilege (PoLP):** Utilize scoped down IAM policies for execution roles and access. By default, applications only have access to their own resources.
- **Observability and Monitoring:** Utilize CloudWatch Logs, S3 Logs, CloudWatch Alarms, CloudWatch Dashboards, Lambda Insights, X-Ray tracing, and testing methods
- **Separation of Stacks:** Organize stacks by lifecycle and ownership. Modularize.
- **Event Driven and Modular Architecture:** Utilize CloudTrail, Event Bridge, and S3 Events to trigger invocations. Instead of large code libraries, utilize native services such as Simple Notification Service (SNS), Simple Queue Service (SQS), S3, Step Functions, and API Gateway to handle tasks.

While the templates provided may not be perfect, they are continually being improved upon and can serve as a reference for how to implement various properties, resources, and undocumented (or hard to discover) advanced configurations and bugs/feature fixes. Hence, "near-production" ready. Can they be used in production? Yes. Are they used in production? Yes. Can they be used in production at your organization? Maybe. (Check your organization's requirements for logging, alarms, and compliance.)

These templates are provided AS-IS and offer NO WARRANTY. However, they are a great tool and reference for:

- Beginners
- Experimentation
- Education
- Training

With all of that out of the way, let's get started!

### Setting Up Work Environment

#### Development Environment

The following must be installed on your development environment:

- [Python >3.12](https://www.python.org/downloads/)
- [Node.js >22](https://nodejs.org/en/download) (NVM recommended: [Free Code Camp NVM Install Guide](https://www.freecodecamp.org/news/node-version-manager-nvm-install-guide/))
- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) (use GitBash for Windows)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [AWS SAM](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)
- [GitHub CLI](https://cli.github.com/) (If using GitHub repositories)

Bash commands to check on Linux or Mac:

```bash
if [ $(command -v python3) ]; then python3 --version; else echo "python3 - NOT INSTALLED"; fi &&
if [ $(command -v pip) ]; then pip --version; else echo "pip - NOT INSTALLED"; fi &&
if [ $(command -v git) ]; then git --version; else echo "git - NOT INSTALLED"; fi &&
. ~/.nvm/nvm.sh && # since it is a shell script, we need to make it available here
if [ $(command -v nvm) ]; then echo "nvm $(nvm --version)"; else echo "nvm - NOT INSTALLED"; fi &&
if [ $(command -v node) ]; then echo "node $(node --version)"; else echo "node - NOT INSTALLED"; fi &&
if [ $(command -v aws) ]; then aws --version; else echo "aws - NOT INSTALLED"; fi &&
if [ $(command -v sam) ]; then sam --version; else echo "sam - NOT INSTALLED"; fi &&
if [ $(command -v gh) ]; then gh --version; else echo "gh - NOT INSTALLED"; fi
```

I personally recommend running Ubuntu >24 on a virtual environment using [WSL (Windows Subsystem for Linux)](https://learn.microsoft.com/en-us/windows/wsl/install) or [UTM (Mac)](https://mac.getutm.app/), but you do you!

#### AWS Account Set-Up and IAM Roles

If you are in charge of setting up IAM User Roles, Configuration Repository, Template Repository for your organization (or personal) AWS Account (you're a Cloud Architect, Platform Engineer, Account Administrator) then perform the steps outlined in [Atlantis Cfn Configuration Repo for Serverless Deployments](https://github.com/63Klabs/atlantis-cfn-configuration-repo-for-serverless-deployments).

If you are a student, learner, trainee, or employee using an account already set up for you, then make sure you obtain the following information from your instructor, teacher, supervisor, or IT partner as the account should be ready:

- Prefix
- Permissions Boundary ARN
- Role Path
- Service Role ARN
- S3 Bucket Org Prefix
- Parameter Store Path

## SAM Configuration Repository

Clone your organization's SAM Configuration Repository to your local machine and read through the Documentation for Developers.

If your organization does not have a Configuration Repository set up, or if you are using a personal account and have not set one up, please go back to the section above titled [AWS Account Set-Up and IAM Roles](#aws-account-set-up-and-iam-roles).

## Templates

The pipeline, storage, and network CloudFormation templates used in the tutorials are housed in an S3 bucket. Your organization either set up an internal bucket that you should have access to, or you will be using templates from the public s3://63klabs bucket.

## Application Starters

Starting with 00 Basic API Gateway with Lambda Function Written in Node.js, the tutorials will utilize application starters.

The SAM Config Repository has a script that will create a repository and seed it with the chosen Application Starter code.

Besides using them for the tutorials, application starter code can be used as a template for your next serverless project.

The tutorials will walk you through using the scripts, but here is a quick reference:

- CodeCommit
  1. `./cli/create_repo.py your-repo-name`
  2. You will be prompted to choose a starter app.
  3. It will create the repository, branches, and provide you with a Clone URL.
- GitHub
  1. `./cli/create-gh-repo.sh your-repo-name app-starter-location` where `app-starter-location` is either an S3 location or GitHub repository URL for the Starter App.
  2. It will create the repository, branches, and provide you with a Clone URL.

  The repository will be created with 3 branches:

  - main (default but empty)
  - test (empty)
  - dev (your seeded code will be here)

  Clone the repository to your machine and check-out the `dev` branch to see your code. (Note: The `main` and `test` branches will be empty until you merge and push your updated code!)

  > Note: AWS CodeCommit is no longer available to new customers. Existing customers of AWS CodeCommit can continue to use the service as normal. [Learn more](https://aws.amazon.com/blogs/devops/how-to-migrate-your-aws-codecommit-repository-to-another-git-provider/)

  > While an `app-starter-location` is required for a GitHub repository, it is not required for the `create_repo.py` script. However, for either script you can always point it to a zip file in an S3 bucket or a GitHub repository to seed your new repository.

## Branching and Git Workflow Strategy for Tutorials

You'll notice your repositories are initially created with 3 branches:

- `dev` (for constant commits of unfinished code)
- `test` (for automated deployments to test)
- `main` (default and for automated deployments to production)

When we initially create the repository, the `main` and `test` branches only include a single placeholder file. This is because we haven't had any finished code to deploy.

The repositories are set up using a simplified `dev-test-main` workflow where a developer can continually commit and push changes to the `dev` branch without invoking an automated deployment. Since continually running an AWS CodePipeline can incur cost, code is only merged and pushed to `test` when it is ready to deploy to a test instance.

If changes need to be made, go back to the `dev` branch, make changes, commit and push to `dev`, then merge and push to `test`. This cycle continues until the code in `test` is deemed ready for production.

Once ready for production, the `test` branch is merged and pushed into `main` which kicks off a deployment for production.

To complete the cycle, a developer may wish to merge main into dev just to ensure no side branches (feature or hotfix) were merged that have not been received into `dev`.

If a staging or beta branch is desired, it can be inserted in between `test` and `main`. (`dev-test-beta-main` or `dev-test-stage-main`).

We will be using this branch workflow strategy for the tutorials as diving into feature, hot-fix, etc. is beyond the scope of these tutorials. Your Git workflow strategy depends on your organization, how many developers are working on a repository, and your needs. Outside of the tutorials, you can reconfigure, use different strategies, and point a pipeline to any branch, including `feat-328` or `fix-98`. At the end of the day all you need is a branch or two that can receive a commit to kick off a deployment pipeline.

## Tutorials

To ensure everything is set-up correctly, start with:

- SAM Config Repo Documentation for Developers (if you haven't already)
- [Serverless 8 Ball Example](https://github.com/chadkluck/serverless-sam-8ball-example) (if you haven't already)
- Application Starter: 00 Basic API Gateway with Lambda Function Written in Node.js

Then, move on to:

1. TODO
2. TODO
3. TODO
