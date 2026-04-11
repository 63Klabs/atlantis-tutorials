# Tutorial #02: Advanced API Gateway and Lambda using Cache-Data (Node)

This tutorial is in four parts and covers many best practices used for production applications. Therefore, it is much longer than other tutorials. However, all following tutorials will utilize these concepts without necessarily calling them out.

## Prerequisite

An understanding of concepts outlined in previous tutorials is required.

If you have not read through the [introductory README](../../README.md), or have not completed the [previous tutorials](../../README.md#tutorials), please do so before proceeding. Each tutorial builds on concepts from the previous and is not something to just "jump into."

You will also need to ensure a CloudFormation stack named `<prefix>-cache-data-storage` exists as it is required for Application Starter #02. One easy way to check is to run the command from the CLI (replace 'YOUR_PROFILE' and 'acme'):

```
aws cloudformation list-exports --profile YOUR_PROFILE --query "Exports[?starts_with(Name, 'acme-CacheData')]"
```

- **If it returns `[]`** then the stack does not exist and you will need to tell your instructor, supervisor, or account administrator and move on to the [next tutorial](../03-static-website-deployment/README.md).
- **If it returns a JSON list** of Exports, Names, and Values then you are good to continue with this tutorial.

## Objectives

By the end of this tutorial, you will be able to deploy a production-ready application that provides an API endpoint for a complex web service that incorporates monitoring, deployment tests, and production capabilities.

### Part I Objectives: Templates and Environment

1. Seed Repository and Create Pipeline
2. Inspect Parameters and Environment Variables  
3. Identify Components of the Application Template
   1. Metadata
   2. Parameters and Overrides
   4. Mappings
   5. Conditionals
   6. Globals
   7. Resources
   8. Using `ImportValue` instead of parameters
   9. Using `Fn::Transform` and `AWS::Include`
   10. Outputs
4. Identify the Components of the Build Process
   1. Secure Secrets Using SSM Parameter Store
   2. Installs and Scripts

### Part II Objectives: Application Starter 02 API Gateway with Lambda using Cache-Data (Node.js)

1. Inspect and utilize various `cache` and `tools` features of the @63klabs/cache-data npm package
   1. Configuration
   2. Caching
   3. Best Practices for Configuration and Caching
   4. Monitoring and Debugging
2. Identify components of the application structure
   1. Application Architecture Overview
   2. Routes
   4. Controllers
   5. Services
   6. Views and Utilities
   7. Models

### Part III Objectives: Dive deeper into controllers, services, models, and views

1. Implement a basic service with a direct call to an endpoint with no caching (8 Ball)
2. Implement a service using Cache Data Access Object (Games)
3. Implement a Route, Controller, and View that brings two services together

### Part IV Objectives: Monitoring and Performance

1. Trace logs using X-Ray
2. Monitor performance using Lambda Insights
3. Monitor using CloudWatch Dashboards
4. Create Alarms and Rollbacks
5. Create tests
6. Automated Testing Strategies
7. Clean up

## Start

> Note: This tutorial uses the `Prefix`: `acme` and `--profile YOUR_PROFILE`. Be sure to replace with your own requirements. Also, if your organization requires you to add your username or name to the front of the repository name or `ProjectId` you may do so to keep the account tidy.

1. [Part I](./part-01.md)
2. [Part II](./part-02.md)
3. [Part III](./part-03.md)
4. [Part IV](./part-04.md)
