# Tutorials for Serverless Deployments using 63K Atlantis Templates and Scripts Platform

Tutorials and walk-throughs for deploying serverless applications using CloudFormation and Severless Application Model (SAM) templates provided by the [63Klabs Atlantis Templates and Scripts Platform](https://github.com/63klabs/atlantis).

> The Atlantis Platform is named after the space shuttle "Atlantis", named after the oceanographic research ship "RV Atlantis", named after the legendary "lost city" of Atlantis.

The Atlantis Platform is meant to bridge the gap between learning and deploying production-ready serverless applications on AWS and consists of templates, scripts, and application starter code using:

- Best Practices in security, testing, CI/CD and organization
- Infrastructure as Code (CloudFormation and SAM)
- Native AWS resources including `samconfig` files

Each tutorial builds on the concepts of building and automating deployments. While Atlantis has proven to scale and accomodate large workloads in production environments, it has also proven simple enough when used as a tool for education and training.

The templates and architecture explored intend to demonstrate implementing best practices using the [AWS Well Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html):

1. Operational excellence
2. Security
3. Reliability
4. Performance efficiency
5. Cost optimization
6. Sustainability

And:

- [CloudFormation Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html)
- [Best Practices for Organizing Larger Serverless Applications](https://aws.amazon.com/blogs/compute/best-practices-for-organizing-larger-serverless-applications/)

With all of that out of the way, let's get started!

## Setting Up Work Environment

### Development Environment

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


## SAM Configuration Repository

All configurations and deployments start in the SAM Configuration Repository. This repository is maintained by your organization and should already be set up for you by the account administrator.

Make sure you obtain the following information from your instructor, teacher, supervisor, or IT partner as the account and SAM configuration repository should be ready:

- Prefix (usually tied to an account, department, or team)
- Location of SAM Configuration repository for your team/organization

Clone your organization's SAM Configuration Repository to your local machine and read through the **Documentation for Developers** as it includes important information on installing and using the required Python packages in a Python virtual environment.

The SAM Configuration Repository contains the scripts and `samconfig` file used to maintain the infrastructure needed to support your application.

## Tutorials

### Intro Tutorials

Start with the basics.

- [Prerequisite #0: Serverless 8 Ball Example (chadkluck/serverless-sam-8ball-example)](https://github.com/chadkluck/serverless-sam-8ball-example) (if you haven't already)
- [Prerequisite #1: SAM Config Repo Quick Start Documentation for Developers](https://github.com/63Klabs/atlantis-cfn-configuration-repo-for-serverless-deployments) (if you haven't already)
- [Tutorial #0: Basic API Gateway with Lambda Function Written in Node.js](./tutorials/00-basic-api-gateway-with-lambda-written-in-node/README.md)

### Intermediate to Advanced Tutorials

Move into advanced concepts as you explore Atlantis templates and scripts.

1. [Tutorial #01: Basic API Gateway with Lambda (Python)](./tutorials/01-basic-api-gateway-with-lambda-written-in-python/README.md)
2. [Tutorial #02: Advanced API Gateway and Lambda (Node)](./tutorials/02-advanced-api-gateway-lambda-cache-data-node/README.md)
3. [Tutorial #03: Deploying a Static Website](./tutorials/03-static-website-deployment/README.md)
4. [Tutorial #04: Atlantis SAM Config Scripts In Depth](./tutorials/04-atlantis-sam-config-scripts-in-depth/README.md)
5. [Tutorial #05: Run CodeBuild on a Schedule for Operations](./tutorials/05-run-codebuild-on-a-schedule-for-operations/README.md)
6. [Tutorial #06: Implementing @63klabs/cache-data in a Web Service](./tutorials/06-implementing-cache-data/README.md)
7. [Tutorial #07: Using the Atlantis MCP Server for AI-Assisted Development](./tutorials/07-using-the-atlantis-mcp-server/README.md)
8. [Tutorial #08: Atlantis SAM Templates In-Depth](./tutorials/08-atlantis-sam-templates-in-depth/README.md)
9. [Tutorial #09: Creating Custom Atlantis Compatible Applications](./tutorials/09-creating-custom-atlantis-compatible-applications/README.md)
10. [Tutorial #10: Atlantis Platform In-Depth](./tutorials/10-atlantis-platform-in-depth/README.md)

### Built on Atlantis

These are **ready-to-deploy-and-run** projects built on Atlantis. They don't offer tutorials beyond installation and configuration documentation, however, use them as examples as you deploy serverless workloads on AWS.

- [Serverless CloudFrontCache Invalidation (63klabs/serverless-cloudfront-cache-invalidation)](https://github.com/63Klabs/atlantis-starter-03-serverless-cloudfront-cache-invalidation)
- [Serverless Image Resizer (chadkluck/serverless-image-resizer)](https://github.com/chadkluck/serverless-image-resizer)
- [Video Transcoding using Elemental MediaConvert (chadkluck/serverless-video-converter)](https://github.com/chadkluck/serverless-video-converter)
- [Atlantis (serverless) MCP Server (63klabs/atlantis-mcp)](https://github.com/63Klabs/atlantis-mcp)

## More Info and Resources

- [Atlantis web site](https://atlantis.63klabs.net)
- [Atlantis on GitHub](https://github.com/63klabs/atlantis)
- [Atlantis SAM Templates](https://github.com/63klabs/atlantis-sam-templates)
- [Atlantis SAM Scripts](https://github.com/63klabs/atlantis-sam-config-scripts)
- [Atlantis MCP Server](https://mcp.atlantis.63klabs.net)
