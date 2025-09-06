# Tutorial #1: Basic API Gateway with Lambda written in Python

> Uses [Atlantis App Starter - 01 - Basic API Gateway with Lambda written in Python](https://github.com/63Klabs/atlantis-starter-01-basic-apigw-lambda-py)

Refer to the README in the app starter GitHub repository above for an overview of the code.

## Prerequisite

An understanding of concepts outlined in previous tutorials is required.

If you have not read through the [introductory README](../../README.md), or have not completed the [previous tutorials](../../README.md#tutorials), please do so before proceeding. Each tutorial builds on concepts from the previous and is not something to just "jump into."

## Objectives

By the end of this tutorial you will be able to identify the various stages of the CodePipeline build and deployment process, and utilize automated scripts during the build.

1. Seed repository and create pipeline
2. Inspect the Pipeline progress through the console
3. Review buildspec.yml and identify the phases
4. Review CodeBuild logs in the console
5. Create a simple script to run during CodeBuild
6. Build packages vs Application packages
7. Package and Library Cache
8. Multiple BuildSpec files are Discouraged
9. Clean-Up

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

The AWS Console is useful for inspecting settings, workflows, and logs.

The AWS Code Pipeline console provides a workflow diagram of the state of your pipeline's execution. You also have access to previous executions, the CodeBuild console logs, and insight into deployment errors.

The first stage of the pipeline is the Source stage. This stage just specifies where the code is coming from. When code is pushed to the repository an event is triggered that lets the pipeline know it is time to execute. The pipeline pulls the changes from the repository and passes it to the second stage.

The second stage of the pipeline is the Build stage. This stage is where the CodeBuild instance is created and the buildspec.yml file is executed. The buildspec.yml file contains a series of commands that are run in sequence.

You can think of CodeBuild as a virtual machine or container running Linux. It is not just for deploying code from a repository, it can be used anytime you need to execute scripts, AWS CLI, or AWS CDK commands in a temporary Linux environment.

The third stage of the pipeline is the Deploy stage. This stage uses CloudFormation to deploy the application to the specified environment. In this case, it is deploying to a test environment. You'll notice that the Deploy stage has two steps: Generate Change Set and Execute Change Set.

The Generate Change Set step creates a change set that describes the changes that will be made to the CloudFormation stack. This compares the current list of resources to those being deployed and noting what resources need to be added, modified, or deleted.

The Execute Change Set step executes the change set and applies the changes to the stack and it's resources.

## 3. Review buildspec.yml and identify the phases

By default, the pipeline specified by Atlantis looks for the buildspec file in the `application-infrastructure` directory of the files copied from the repository (this, along with CodeBuild environment variables are specified when you configured and deployed your pipeline stack).

Go to your application's repository and open your `application-infrastructure/buildspec.yml` file.

When the CodeBuild instance starts, it brings up a Linux environment with AWS CLI, Python, and Node already installed. It then receives a copy of the code from the specific branch in the repository.

Next, it begins to execute a series of commands specified in the various phases of the buildspec file.

The phases you can use are:

- install
- pre_build
- build
- post_build
- finally

Each phase is optional. You can use as many or as few as you need and you can assign commands as you see fit. Do not worry too much about whether a command should be in the `pre_build` or `build` phase, only the logical placement for sequence and organization required for your application. You should follow a standard practice set by yourself, or your organization, to ensure future maintainability by you or other developers.

If the `install` or `pre_build` phase fails, subsequent phases are skipped. However, if `build` fails, the `post_build` is still attempted. Knowing this will assist in determining where to place your commands. For a flow chart see: [Build phase transitions](https://docs.aws.amazon.com/codebuild/latest/userguide/view-build-details-phases.html).

Each phase can contain a list of commands, and a finally step can be specified for each phase to ensure certain commands always run, even if other commands within that phase fail. This phased approach provides granular control over the build process in AWS CodeBuild.

### Install

This phase is typically used for setting up the build environment and installing necessary dependencies. Commands in this phase might include installing specific versions of programming languages, package managers, or other required tools.

### Pre-build

This phase executes commands before the main build process begins. It can be used for preliminary tasks such as running pre-build scripts, setting up environment variables, or performing static code analysis.

### Build

This is the core phase where the actual build commands are executed. This includes compiling code, running tests, and packaging the application.

### Post-build

This phase runs after the build phase completes, regardless of whether the build was successful or not. It is commonly used for tasks like uploading build artifacts to Amazon S3, sending notifications, or performing cleanup operations.

### More Information about the Build Specification file

- [Run buildspec commands for the INSTALL, PRE_BUILD, and POST_BUILD phases](https://docs.aws.amazon.com/codebuild/latest/userguide/sample-runner-buildkite-buildspec.html)
- [Build specification reference for CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)

## 4. Review CodeBuild logs in the console

TODO

## 5. Create a simple script to run during CodeBuild

TODO

> Note: Scripts must be able to run "headless" as there is no chance to respond to interactive prompts during the CodeBuild process. If you write a script that you wish to use both interactively when running locally and headless when executed in a CodeBuild environment, you should include a flag such as `--headless` and acquire prompt information either through environment variables or parameters passed to the script.

> Another Note: A dry-run option is also useful for testing scripts without actually executing changes. If you include something such as a `--dryrun` flag or environment variable option for your script, you can output confirmations such as "`DRYRUN: Value would have been saved to SSM Parameter Store`". This can further enhance documenting the differences, yet still test and notify of skipped commands, between `TEST` and `PROD` environments.

## 6. Build packages vs Application packages

As explained earlier, we perform two installs of Python and Node packages. Let's dive into this deeper.

The first install is for the build environment. This includes packages needed to run build scripts, perform linting, tests, and any other tasks necessary to prepare the application for deployment. These packages are installed during the `install` phase and executed during the `pre_build` phase. As mentioned before, if either the `install` or `pre_build` phases fail, the `build` and `post_build` phases are not attempted.

The second install is for the application code being packaged and deployed to Lambda. This includes Node or Python packages/libraries that are required for the application to run in its target Lambda environment. These packages are installed in the `pre_build` phase of the buildspec file.

Compare the buildspec of this tutorial with the [buildspec of Application Starter #00](https://github.com/63Klabs/atlantis-starter-00-basic-apigw-lambda-nodejs/blob/main/application-infrastructure/buildspec.yml) used in the previous tutorial. You'll notice that because Node packages were installed in the previous starter, there are additional flags for production vs development and security audits.

## 7. Package and Library Cache

Even though each CodeBuild environment is fresh and clean (you cannot save state between runs, and therefore do not need to worry about contamination) CodeBuild does provide the ability to cache libraries and packages which allows for faster installs and reduces the strain on external package providers.

AWS documentation doesn't seem to specify how long these caches are kept around (hours, days, weeks?), but if you are deploying to test several times a day, you will notice a difference.

First, the standard cache settings are set up for Python and/or Node in the `install` phase. You can see this in both the current buildspec and the buildspec for the previous tutorial.

```yaml
phases:
  install:
	commands:
	  # ... other commands

	  # Configure pip cache directory
      - mkdir -p /root/.cache/pip
      - pip config set global.cache-dir /root/.cache/pip

      # Set npm caching (This 'offline' cache is still tar zipped, but it helps.) - https://blog.mechanicalrock.io/2019/02/03/monorepos-aws-codebuild.html
      - npm config -g set prefer-offline true
      - npm config -g set cache /root/.npm
      - npm config get cache
```

Next, the cache paths are specified at the bottom of the buildspec file. This tells CodeBuild which directories to save and restore between builds.

```yaml
cache:
  paths:
	- '/root/.cache/pip/**/*'
	- '/root/.npm/**/*'
```

It is important to note that the ability for CodeBuild to maintain a cache is not set by default when creating your own pipelines from scratch, the Atlantis pipeline template specifies that caching is enabled for CodeBuild so you don't have to.

## 8. Multiple BuildSpec files are Discouraged

It is possible to have multiple buildspec files in a single repository, however it is discouraged. Having multiple buildspec files can lead to confusion and maintenance challenges. It is best practice to have a single buildspec file per repository to ensure clarity and consistency in the build process.

Utilize environment variables to determine execution paths for `PROD` and `TEST` environments. This helps self-document any differences between prod and test environments.

> While you may come across some tutorials or examples on the web that use `buildspec_prod.yml` and `buildspec_test.yml` it is best practice to maintain a single `buildspec.yml` file.

## 9. Clean-Up

To avoid ongoing charges to your AWS account, delete the resources created in this tutorial.

> Note: This step is optional and is dependent upon your user permissions and whether or not you wish or are required to delete the stacks created in this tutorial. It is recommended, for practice and if you have the proper permissions, to delete at least one of your stages. This helps with practice and you can always go through the steps of creating and deploying the stage later. That's the nice thing about automation!

The `delete.py` script is provided to perform clean-up operations in proper order.

As the accidental deletion of stacks can be devastating, the delete script requires several confirmation steps.

You will be required to provide the ARNs for both the pipeline and application stack. You may obtain these from the Stack Info tab in the CloudFormation web console.

> Proceed with caution! Double check your work and make sure you are deleting the correct stack!

There are 2 manual steps that need to take place prior to running the delete script. Some organizations may restrict who can perform these steps to ensure proper checks and balances.

1. Manually add a tag to the pipeline stack with the key `DeleteOnOrAfter` and a value of a date in `YYYY-MM-DD` format. (Add `Z` to end for UTC. Example `2026-07-09Z`). This can be done using the AWS CLI:
	- `aws resourcegroupstaggingapi tag-resources --resource-arn-list "arn:aws:cloudformation:region:account:stack/stack-name/stack-id" --tags DeleteOnOrAfter=YYYY-MM-DD --profile your-profile`
	- Be sure to replace the ARN in the command with the pipeline stack ARN.
	- Successful completion will result in receiving an empty `FailedResourcesMap`
	- You can double check by going to the pipeline stack in the CloudFormation console.
2. Disable termination protection: `aws cloudformation update-termination-protection --stack-name STACK_NAME --no-enable-termination-protection`
    - Be sure to replace `STACK_NAME` with the name of the stack. You do not need the full ARN for this command.
	- Do this for both the `pipeline` and `application` stacks.

The delete script is now ready to be ran from the SAM config repository:

```bash
# Perform this command in the SAM Config Repo
./cli/delete.py pipeline acme py8ball-adv beta --profile ACME_DEV
```

You will have the chance to either retain the stage's environment settings in the `samconfig` file for later re-deployment, or to delete it completely. Once all stage environments of a `samconfig` file are deleted the file and directory for that project is also deleted.

Be sure to perform this operation for any unwanted stages of your application (`test`, `beta`, `prod`, etc.).

Performing the delete does not delete the repository. Since the size of the repository is minimal you will not incur charges and may leave the repository as-is for future reference.

## Summary

Congratulations! You have completed Tutorial #1! You have successfully deployed a basic API Gateway with Lambda written in Python using an automated CI/CD pipeline. You have also inspected the pipeline stages and CodeBuild logs, and reviewed the buildspec.yml file.

You can now use this knowledge to deploy more complex applications and services using the same principles and techniques.

- [Next Tutorial: Tutorial #2: Advanced API Gateway and Lambda using Cache-Data (Node)](./../02-advanced-api-gateway-lambda-cache-data-node/README.md)
- [All Tutorials](../../README.md)