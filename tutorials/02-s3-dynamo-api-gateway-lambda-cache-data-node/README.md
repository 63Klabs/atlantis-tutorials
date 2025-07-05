# Tutorial #2: S3 and DynamoDb storage stack for API Gateway and Lambda using Cache-Data (Node)

This tutorial is in two parts and covers many best practices used for production applications. Therefore, it is much longer than other tutorials. However, all following tutorials will utilize these concepts without necessarily calling them out.

> This tutorial is THE tutorial that bridges learning and application development for production environments.

## Prerequisite

An understanding of concepts outlined in previous tutorials is required.

If you have not read through the [introductory README](../../README.md), or have not completed the [previous tutorials](../../README.md#tutorials), please do so before proceeding. Each tutorial builds on concepts from the previous and is not something to just "jump into."

## Objective

By the end of this tutorial you will be able to deploy a production-ready application that utilizes a separate storage stack with S3 and DynamoDb resources for use with a Lambda application stack that provides an API endpoint for a web service that incorporates caching, monitoring, deployment tests, and production capabilities.

1. Create a storage stack with S3 and DynamoDb resources
2. Seed repository and create pipeline
3. Utilize conditionals for resource creation and properties
4. Secure secrets using SSM Parameter Store
4. Inspect and utilize various features of the @63klabs/cache-data npm package
   1. Configuration
   2. Caching
   3. Debugging
   4. Request handling
5. Trace logs using X-Ray
6. Monitor performance using Lambda Insights and CloudWatch Dashboards
7. Create Alarms and Rollback Deployments
8. Automate unit testing
9. Automate post-deployment testing

## Part I: Storage Stack using S3 and Dynamo DB for Cache-Data

> This first part uses an Atlantis template for deploying an S3 bucket and DynamoDb table for storing cached data.

TODO

## Part II: Application Starter: 02 API Gateway with Lambda using Cache-Data (Node.js)

> Uses [Atlantis App Starter - 02 - API Gateway and Lambda using @63Klabs/Cache-Data (Node)](https://github.com/63Klabs/atlantis-starter-02-apigw-lambda-cache-data-nodejs)

Refer to the README in the app starter GitHub repository above for an overview of the code.

TODO