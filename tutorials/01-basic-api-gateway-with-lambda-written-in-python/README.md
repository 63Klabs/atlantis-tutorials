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

> Note: This tutorial uses the Prefix `acme` and profile `ACME_DEV_PROFILE`. Be sure to replace with your own organization's requirements. Also, if your organization requires you to add your username or name to the front of the repository name or ProjectID you may do so to keep the account tidy.

Using the `create_repo` script in your organization's SAM Config repository, create and seed the repository with application starter number 2.

```bash
./cli/create_repo.py tutorial-py8ball-advanced --profile ACME_DEV_PROFILE
```

Choose application starter 01 basic-apigw-lambda-py when prompted.

Clone the repository to your local environment and merge the `dev` branch into the `test` branch.

In the SAM Config repository, create the pipeline for your application.

```bash
./cli/config.py pipeline acme py8ball-adv test --profile ACME_DEV_PROFILE
```

Copy, paste and execute the deploy command from the config output.

```bash
# Perform this command in the SAM Config Repo
./cli/deploy.py pipeline acme py8ball-adv test --profile ACME_DEV_PROFILE
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

## 7. Clean Up

To avoid ongoing charges to your AWS account, delete the resources created in this tutorial.

The `delete.py` script is provided to perform clean-up operations in proper order.

As the accidental deletion of stacks can be devastating, the delete script requires several confirmation steps.

You will be required to provide the ARNs for both the pipeline and application stack. You may obtain these from the Stack Info tab in the CloudFormation web console.

> Proceed with caution! Double check your work and make sure you are deleting the correct stack!

There are 2 manual steps that need to take place prior to running the delete script. Some organizations may restrict who can perform these steps to ensure proper checks and balances.

1. Manually add a tag to the pipeline stack with the key `DeleteOnOrAfter` and a value of a date in `YYYY-MM-DD` format. (Add `Z` to end for UTC. Example `2026-07-09Z`). This can be done using the AWS CLI:
	- `aws resourcegroupstaggingapi tag-resources --resource-arn-list "arn:aws:cloudformation:region:account:stack/stack-name/stack-id" --tags DeleteOnOrAfter=YYYY-MM-DD --profile your-profile`
	- Successful completion will result in receiving an empty `FailedResourcesMap`
	- You can double check by going to the pipeline stack in the CloudFormation console.
2. Disable termination protection: `aws cloudformation update-termination-protection --stack-name STACK_NAME --no-enable-termination-protection`

The delete script is now ready to be ran from the SAM config repository:

```bash
# Perform this command in the SAM Config Repo
./cli/delete.py pipeline acme py8ball-adv test --profile ACME_DEV
```

Be sure to perform this operation for any unwanted stages of your application (`test`, `beta`, `prod`, etc.).

Performing the delete does not delete the repository. Since the size of the repository is minimal you will not incur charges and may leave the repository as-is for future reference.
