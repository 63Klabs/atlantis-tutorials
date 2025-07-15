# Tutorial #2: S3 and DynamoDb storage stack for API Gateway and Lambda using Cache-Data (Node)

This tutorial is in two parts and covers many best practices used for production applications. Therefore, it is much longer than other tutorials. However, all following tutorials will utilize these concepts without necessarily calling them out.

> This tutorial is THE tutorial that bridges learning and application development for production environments.

## Prerequisite

An understanding of concepts outlined in previous tutorials is required.

If you have not read through the [introductory README](../../README.md), or have not completed the [previous tutorials](../../README.md#tutorials), please do so before proceeding. Each tutorial builds on concepts from the previous and is not something to just "jump into."

## Objective

By the end of this tutorial you will be able to deploy a production-ready application that utilizes a separate storage stack with S3 and DynamoDb resources for use with a Lambda application stack that provides an API endpoint for a web service that incorporates caching, monitoring, deployment tests, and production capabilities.

1. Stack Organization: "Separation of Stacks"
2. Create a storage stack with S3 and DynamoDb resources
3. Obtain the DynamoDb table and S3 bucket names
4. Seed repository and create pipeline
5. Utilize conditionals for resource creation and properties
6. `ImportValue` and Parameter Overrides
7. Secure secrets using SSM Parameter Store
8. Inspect and utilize various features of the @63klabs/cache-data npm package
   1. Configuration
   2. Caching
   3. Debugging
   4. Request handling
9. Trace logs using X-Ray
10. Monitor performance using Lambda Insights and CloudWatch Dashboards
11. Create Alarms and Rollback Deployments
12. Automate unit testing
13. Automate post-deployment testing

> Note: This tutorial uses the Prefix `acme` and profile `acme-dev`. Be sure to replace with your own requirements. Also, if your organization requires you to add your username or name to the front of the repository name or ProjectID you may do so to keep the account tidy.

## Part I: Storage Stack using S3 and Dynamo DB for Cache-Data

> This first part uses an Atlantis template for deploying an S3 bucket and DynamoDb table for storing cached data.

### 1. Stack Organization: "Separation of Stacks"

Previously we used the `config.py` and `deploy.py` scripts in the SAM Config repository to create the SAM configuration and deploy an application pipeline.

The config and deploy scripts can also deploy CloudFormation stacks that maintain other infrastructure such as storage, network, and even IAM roles and policies.

These are manual processes as they don't occur often and don't require a deployment pipeline like an application does. They have a different **lifecycle** than your application, and are often handled and **owned** by those in roles outside of application developers. This follows the "Separation of Stacks" best practice to [Organize your stacks by lifecycle and ownership](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html#organizingstacks). 

### 2. Create a storage stack with S3 and DynamoDb resources

> If you were given a DynamoDb table and S3 bucket to use, or if you already created the cache storage in a previous tutorial, you can read through this section without running the `config.py` and `deploy.py` commends.

In the SAM Config repository, create the storage configuration for cache-data.

Note instead of `pipeline` we are creating a `storage` stack. Also note that since this is shared among all applications, and cache-data automatically partitions data between applications and their instances, we do not supply a stage identifier (`StageId`). (However, if you needed a sandbox or test instance, you could append it to the project identifier (`ProjectId`).)

```bash
./cli/config.py storage acme cache-data --profile acme-dev
```

When prompted to select a template you'll see that you will have a list that differs from before. Instead of pipelines, since you provided the `storage` type in the script arguments, it will display available storage templates.

Choose `template-storage-cache-data.yml` since it is an already provided template specific for use with the `63klabs/cache-data` NPM package.

Copy, paste and execute the deploy command from the config output.

```bash
# Perform this command in the SAM Config Repo
./cli/deploy.py storage acme cache-data default --profile acme-dev
```

Be sure to commit and push your configuration to the SAM config repository.

### 3. Obtain the DynamoDb table and S3 bucket names

Before you can use the cache storage in your applications, you will need the DynamoDb table and S3 bucket name.

These can be obtained from the Outputs section of the CloudFormation stack which were displayed after you successfully deployed the cache-data storage.

From the stack outputs, obtain the following:

- S3BucketExport
- S3BucketArnExport

> You can always obtain these values later by going into the AWS Web Console and viewing the Outputs section of the cache-data CloudFormation storage stack.

As we will see later, your application template will have parameters that will accept these values.

## Part II: Application Starter: 02 API Gateway with Lambda using Cache-Data (Node.js)

> Uses [Atlantis App Starter - 02 - API Gateway and Lambda using @63Klabs/Cache-Data (Node)](https://github.com/63Klabs/atlantis-starter-02-apigw-lambda-cache-data-nodejs)

Refer to the README in the app starter GitHub repository above for an overview of the code.

### 3. Seed repository and create pipeline

Using the `create_repo` script in your organization's SAM Config repository, create and seed the repository with application starter 02.

```bash
./cli/create_repo.py tutorial-games-proxy --profile acme-dev
```

Choose application starter 02 (`atlantis-starter-02-apigw-lambda-cache-data-nodejs`) when prompted.

Clone the repository to your local environment and merge the `dev` branch into the `test` branch.

In the SAM Config repository, create the pipeline for your application.

```bash
./cli/config.py pipeline acme games-proxy test --profile acme-dev
```

Copy, paste and execute the deploy command from the config output.

```bash
# Perform this command in the SAM Config Repo
./cli/deploy.py pipeline acme games-proxy test --profile acme-dev
```

After the pipeline has been created successfully, a link to the pipeline will be displayed in the Output. Follow the link to view the pipeline in the console. (You may need to log into the console first before following the link.)

### 5. Utilize conditionals for resource creation and properties

TODO

### 6. `ImportValue` and Parameter Overrides

TODO

### 7. Secure secrets using SSM Parameter Store

TODO

### 7. Inspect and utilize various features of the @63klabs/cache-data npm package

TODO

#### 7.1. Configuration

TODO

#### 7.2. Caching

TODO

#### 7.3. Debugging

TODO

#### 7.4. Request handling

TODO

### 8. Trace logs using X-Ray

TODO

### 9. Monitor performance using Lambda Insights and CloudWatch Dashboards

TODO

### 10. Create Alarms and Rollback Deployments

TODO

### 11. Automate unit testing

TODO

### 12. Automate post-deployment testing

TODO
