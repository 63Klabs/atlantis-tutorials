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

TODO

#### Local Development Environment

TODO

#### AWS Account Set-Up and IAM Roles

TODO

#### Configuration Repository

TODO

#### Template Repository

TODO

#### Application Starters

TODO

## Tutorials

To ensure everything is set-up correctly, start with:

- Serverless 8 Ball Example (if you haven't already)
- Basic API Gateway with Lambda Function Written in Node.js

Then, move on to:

1. TODO
2. TODO
3. TODO
