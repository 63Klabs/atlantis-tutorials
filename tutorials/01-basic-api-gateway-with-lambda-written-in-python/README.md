# Tutorial #1: Basic API Gateway with Lambda written in Python

> Uses [Atlantis App Starter - 01 - Basic API Gateway with Lambda written in Python](https://github.com/63Klabs/atlantis-starter-01-basic-apigw-lambda-py)

Refer to the README in the app starter GitHub repository above for an overview of the code.

## Prerequisite

An understanding of concepts outlined in previous tutorials is required.

If you have not read through the [introductory README](../../README.md), or have not completed the [previous tutorials](../../README.md#tutorials), please do so before proceeding. Each tutorial builds on concepts from the previous and is not something to just "jump into."

## Objective

By the end of this tutorial you will be able to identify the various stages of the CodePipeline build and deployment process, and utilize automated scripts during the build.

1. Seed repository and create pipeline
2. Inspect the Pipeline progress through the console
3. Review CodeBuild logs in the console
4. Review buildspec.yml and identify the phases
5. Create a simple script to run during CodeBuild
6. Build packages vs Application packages

## 1. Seed repository and create pipeline

> Note: This tutorial uses the Prefix `acme` and profile `acme-dev`. Be sure to replace with your own organization's requirements. Also, if your organization requires you to add your username or name to the front of the repository name or ProjectID you may do so to keep the account tidy.

Using the `create_repo` script in your organization's SAM Config repository, create and seed the repository with application starter number 2.

```bash
./cli/create_repo.py tutorial-py8ball-advanced --profile acme-dev
```

Choose application starter 01 basic-apigw-lambda-py when prompted.

Clone the repository to your local environment and merge the `dev` branch into the `test` branch.

In the SAM Config repository, create the pipeline for your application.

```bash
./cli/config.py pipeline acme py8ball-adv test --profile acme-dev
```

After the pipeline has been created successfully, a link to the pipeline will be displayed in the Output. Follow the link to view the pipeline in the console. (You may need to log into the console first before following the link.)

## 2. Inspect the Pipeline progress through the console

TODO

## 3. Review CodeBuild logs in the console

TODO

## 4. Review buildspec.yml and identify the phases

TODO

## 5. Create a simple script to run during CodeBuild

TODO

## 6. Build packages vs Application packages

TODO
