# Tutorial #1 - Basic API Gateway with Lambda written in Node.js

> Uses [Atlantis App Starter - 00 - Basic API Gateway with Lambda written in Node.js](https://github.com/63Klabs/atlantis-starter-00-basic-apigw-lambda-nodejs)

Refer to the README in the app starter GitHub repository above for an overview of the code.

## Objective

By the end of this tutorial you will be able to create a repository and seed it with a starter app, configure and deploy a pipeline for both a test and production deployment, understand the manual deployment process for pipeline stacks, and automated deployments for application stacks.

We will utilize an app starter and scripts in the SAM Config repository to:

1. Create a repository seeded with our app starter using a script
2. Examine the new repository and the branches
3. Configure a `test` Pipeline in the SAM Config repository
4. Deploy the `test` Pipeline from the SAM Config repository
5. Examine the Pipeline and CloudFormation process
6. Test the endpoint
7. Make changes and commit changes to test (to invoke the pipeline)
8. Configure a `prod` Pipeline in the SAM Config repository
9. Deploy the `prod` Pipeline from the SAM Config repository
10. Perform a complete application deployment cycle from `dev` to `prod`

## 1. Create a Repository and Seed it Using a Script

> If you have not yet acquainted yourself with the SAM Config Repository Documentation for Developers please do so as it provides helpful information about the scripts contained within.

We'll start by using your organization's SAM Config Repository (make sure it is cloned (and most recent changes pulled!)) to your machine.

```bash
# Perform these commands from your organization's SAM Config Repository
git pull
./cli/create_repo.py your-repo-name
```

Choose `00-app-starter` from the prompt.

## 2. Repository Configuration

Go into the AWS Console and explore the repository in CodeCommit.

Notice there are three branches as we are using the `dev-test-main` workflow. (See [Branching and Git Workflow Strategy for Tutorials](../../README.md#branching-and-git-workflow-strategy-for-tutorials))

If using CodeCommit, your repository was also given AWS resource tags to assist in account management. Note that resource tags are not the same as git tags. Resource tags are a feature of AWS as a type of meta data used to organize, manage, and report. Git tags are snapshots of repository contents and are a feature of all git repositories.

It is assumed that the starter code works without modification, so let's merge the code from `dev` to `test`.

Check out the `dev` branch if you haven't already.

```bash
git checkout dev
```

## 3. Configure a `test` Pipeline in the SAM Config repository

In a separate terminal, access the SAM Configuration Repository.

We will use the `config.py` script which will walk through configuring an AWS CodePipeline that will trigger a deployment when code is pushed to the `test` branch.

> Note: During the configuration of the template parameters you will need information provided by your organization such as Prefix, Permissions Boundary ARN, Role Path, etc. Be sure you have it available.

Use the `-h` option for information about the script including parameters, named parameters, and flags.

```bash
./cli/config.py -h
```

Note that if you are not using the `default` AWS profile, you will need to use the `--profile` option.

We will use `adv-8-ball` as our `ProjectId` argument. Replace `your-prefix` and `your-profile` with your own.

```bash
# Perform this command in the SAM Config Repo
./cli/config.py pipeline your-prefix adv-8-ball test --profile your-profile
```

When prompted for a template, choose `template-pipeline.yml`

Answer the template parameter prompts. If you need to view the description enter `?`

If you make any mistakes you can continue answering the prompts and re-run the script to enter new values, or quit the script without saving by entering `^`. 

For more information about the prompts and their values, refer to SAM Config Repo Documentation for Developers.

After completing the prompts, the script will display the command to deploy. We'll use this when deploying.

Note that there is also a link to the SAM Config file that saves your deployment settings. It is in the same format as the `samconfig` files we created for deploying the original Serverless 8-Ball Example.

One noticeable difference is that we are referencing a template in S3 rather than a local file. However, `samconfig` DOES NOT support obtaining a template from S3. We will get around this limitation by using the `deploy.py` script.

## 4. Deploy the `test` Pipeline from the SAM Config repository

Copy, paste and execute the deploy command you copied from the config output.

As mentioned, since we are referencing a template stored in S3, the deploy script actually downloads the template to a temporary location on your local machine, and then executes the standard `aws sam deploy` command in the background.

Since the script is executing `aws sam deploy` in the background you will see the familiar `sam deploy` information and (if you set confirm to `true`) prompt to execute the changes.

Once the CloudFormation stack is created you can view it, and the pipeline in the AWS console.

## 5. Examine the Pipeline and CloudFormation process

While the CloudFormation deployment is executing in your terminal, you can open the AWS Console and also see the CloudFormation stack events there.

Once stack creation is complete, you can go to the Outputs tab of the pipeline stack and use the Pipeline link to view the pipeline progress which will kick off immediately upon stack completion.

## 6. Test the endpoint

Once the pipeline has completed, you can go back to your terminal with the CloudFormation stack output or the CloudFormation stack in the console.

Under the stack output you will see Test Endpoint. Click on the link to make sure the application deployed correctly. You should see a standard prediction. (You can ask a Yes/No question prior to clicking on the link if you wish.)

If it wasn't successful resolve any issues before continuing.

## 7. Make changes and commit changes to test (to invoke the pipeline)

TODO

## 8. Configure a `prod` Pipeline in the SAM Config repository

We will again use `adv-8-ball` as our `ProjectId` argument but this time use `prod` as `StageId`. Replace `your-prefix` and `your-profile` with your own.

```bash
# Perform this command in the SAM Config Repo
./cli/config.py pipeline your-prefix adv-8-ball prod --profile your-profile
```

## 9. Deploy the `prod` Pipeline from the SAM Config repository

Perform the same copy, paste, execute on the deploy.py command as before and deploy the stack. Watch the updates in the terminal or console. You can also monitor the pipeline.

## 10. Perform a complete application deployment cycle from `dev` to `prod`

Let's add additional features to the endpoint such as dice rolls and card deals.

```js
```

TODO
