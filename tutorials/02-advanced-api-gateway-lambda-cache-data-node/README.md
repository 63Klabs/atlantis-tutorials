# Tutorial #2: Advanced API Gateway and Lambda using Cache-Data (Node)

This tutorial is in two parts and covers many best practices used for production applications. Therefore, it is much longer than other tutorials. However, all following tutorials will utilize these concepts without necessarily calling them out.

> This tutorial is THE tutorial that bridges learning and application development for production environments. It is also THE tutorial for working with the @63klabs/cache-data npm package.

## Prerequisite

An understanding of concepts outlined in previous tutorials is required.

If you have not read through the [introductory README](../../README.md), or have not completed the [previous tutorials](../../README.md#tutorials), please do so before proceeding. Each tutorial builds on concepts from the previous and is not something to just "jump into."

## Objective

By the end of this tutorial you will be able to deploy a production-ready application that utilizes a separate storage stack with S3 and DynamoDb resources for use with a Lambda application stack that provides an API endpoint for a web service that incorporates caching, monitoring, deployment tests, and production capabilities.

### Part I Objectives: Multiple Stacks

1. Stack Organization: "Separation of Stacks"
2. Create a storage stack with S3 and DynamoDb resources
3. Caching using DynamoDb table and an S3 bucket
4. Cache security
5. Stack Outputs and Exports

### Part II Objectives: Templates and Environment

1. Seed repository and create pipeline
2. Inspect Parameters and Environment Variables  
3. Check endpoint and cache in DynamoDb
4. Identify components of application template
   1. Metadata
   2. Parameters and overrides
   3. Utilize conditionals for resource creation and properties
   4. Mapping
   5. Using `ImportValue` instead of parameters
5. Identify the components of build process
   1. Secure secrets using SSM Parameter Store
   2. Installs and scripts

### Part III Objectives: Application Starter 02 API Gateway with Lambda using Cache-Data (Node.js)

1. Inspect and utilize various `cache` and `tools` features of the @63klabs/cache-data npm package
   1. Configuration
   2. Caching
   3. Debug logs and Timer
2. Identify components of the application structure
   1. Routes
   2. Request
   3. Controller
   4. Service
   5. View
   6. Models
   7. Response
3. Dive deeper into controllers, services, models, and views
   1. Implement a copy of Example components for the Games API (using CachedData)
   2. Implement a basic service with a direct call to an endpoint with no caching (8 Ball)
   3. Implement a service using Data Access Object with an api key behind caching (Weather)
   4. Implement a service with a call to a DynamoDb table
   4. Static, Sample, and Test data
   5. Create a Controller and View utilizing the four data services

### Part IV Objectives: Monitoring and Performance

1. Trace logs using X-Ray
2. Monitor performance using Lambda Insights and CloudWatch Dashboards
3. Create Alarms and Rollbacks
4. Create unit tests and automate pre-deployment testing
5. Automate post-deployment testing

> Note: This tutorial uses the Prefix `acme` and profile `ACME_DEV_PROFILE`. Be sure to replace with your own requirements. Also, if your organization requires you to add your username or name to the front of the repository name or ProjectID you may do so to keep the account tidy.

1. Part I
2. Part II
3. Part III
4. Part IV