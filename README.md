# Serverless Deployments using 63K Atlantis Tutorials

Tutorials and walk-throughs for deploying serverless applications using CloudFormation and Severless Application Model templates provided by 63Klabs.

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

Clone your organization's SAM Configuration Repository to your local machine and read through the **Documentation for Developers** as it includes important information on installing and using the required Python packages in a Python virtual environment.

If your organization does not have a Configuration Repository set up, or if you are using a personal account and have not set one up, please go back to the section above titled [AWS Account Set-Up and IAM Roles](#aws-account-set-up-and-iam-roles).

## Templates

The pipeline, storage, and network CloudFormation templates used in the tutorials are housed in an S3 bucket. Your organization either set up an internal bucket that you should have access to, or you will be using templates from the public s3://63klabs bucket.

## Application Starters

Starting with 00 Basic API Gateway with Lambda Function Written in Node.js, the tutorials will utilize application starters.

The SAM Config Repository has a script that will create a repository and seed it with the chosen Application Starter code.

Besides using them for the tutorials, application starter code can be used as a template for your next serverless project.

The tutorials will walk you through using the scripts, but here is a quick reference:

1. Run the create repository script:
   - CodeCommit: `./cli/create_repo.py your-repo-name`
   - GitHub: `./cli/create_repo.py your-repo-name your-repo-name --provider github`
2. You will be prompted to choose a starter app.
3. It will create the repository, branches, and provide you with a Clone URL.

The repository will be created with 3 branches:

- main (default but empty)
- test (empty)
- dev (your seeded code will be here)

> The tutorials will use the `dev-test-main` branch merge strategy to get code from development to production (`main` branch). For more information see [Default Git Branch Workflow](./tutorials/default-git-branch-workflow.md)

Clone the repository to your machine and check-out the `dev` branch to see your code. (Note: The `main` and `test` branches will be empty until you merge and push your updated code!)

> Note: AWS CodeCommit is no longer available to new customers. Existing customers of AWS CodeCommit can continue to use the service as normal. [Learn more](https://aws.amazon.com/blogs/devops/how-to-migrate-your-aws-codecommit-repository-to-another-git-provider/)

## Tutorials

To ensure everything is set-up correctly, start with:

- SAM Config Repo Quick Start Documentation for Developers (if you haven't already)
- [Serverless 8 Ball Example](https://github.com/chadkluck/serverless-sam-8ball-example) (if you haven't already)
- [Application Starter: 00 Basic API Gateway with Lambda Function Written in Node.js](./tutorials/00-basic-api-gateway-with-lambda-written-in-node/README.md)

Then, move on to:

1. [Tutorial #1: Basic API Gateway with Lambda written in Python](./tutorials/01-basic-api-gateway-with-lambda-written-in-python/README.md)
2. [Tutorial #2: Advanced API Gateway and Lambda using Cache-Data (Node)](./tutorials/02-advanced-api-gateway-lambda-cache-data-node/README.md)
3. Coming Soon: Storage Stack using S3 and Origin Access Control for Static Website
   - Application Starter: 03 Static Website Deployment with CodeBuild and S3
   - Network Stack using CloudFront and Route53
4. Coming Soon: Application Starter: 04 Event Triggered Lambda
5. Coming Soon: Application Starter: 05 Event Triggered Step Function
6. Coming Soon: Application Starter: 06 Video Transcoding using Elemental MediaConvert
7. Coming Soon: Application Starter: 07 SQS and Lambda
