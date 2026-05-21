# Run CodeBuild on a Schedule for Operations

> This tutorial is still under development. However, the basic structure is listed below. Please be advised that the content is short, may be missing, and may have inaccuracies or typos. If you would like to contribute updates, please submit an [issue via this repository on GitHub](https://github.com/63Klabs/atlantis-tutorials/issues). Be sure to include the page and what content should be added/updated. If you'd be willing to write a few sentences (or more, but be clear and succinct), please do. Thank you for your understanding.

CodeBuild provides a managed service, a clean slate, ability to run a variety of scheduled scripts and commands without having to adapt them to run as a Lambda function.

Data and file persistence can be maintained by:

- Cloning a repo (read only)
- S3 sync or copy
- S3 File System mount
- CLI or API calls

If system persistence is not required, then CodeBuild is a good, lightweight, low-maintenance option and this application is meant to fill that need.

## Deploy the Solution As-Is

Follow the steps outlined in the [Atlantis Starter #03 Serverless CodeBuild Container DEPLOYMENT.md](https://github.com/63Klabs/atlantis-starter-03-serverless-codebuild-container/blob/main/DEPLOYMENT.md) file to create the repository, deploy an S3 bucket, a pipeline, and application.

## Run the Schedule Manually

TODO

## Add a Script

TODO

## Use the AWS CLI

TODO

Update the execution role

Query an AWS Resource

Output the results to S3

## Clean-Up

TODO
