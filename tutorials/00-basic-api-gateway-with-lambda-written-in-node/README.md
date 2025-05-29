# Tutorial #0 - Basic API Gateway with Lambda written in Node.js

> Uses [Atlantis App Starter - 00 - Basic API Gateway with Lambda written in Node.js](https://github.com/63Klabs/atlantis-starter-00-basic-apigw-lambda-nodejs)

Refer to the README in the app starter GitHub repository above for an overview of the code.

## Objective

By the end of this tutorial you will be able to create a repository and seed it with a starter app, configure and deploy a pipeline for both a test and production deployment, understand the manual deployment process for pipeline stacks from the cli, and utilize automated deployments for application stacks.

We will utilize an app starter and scripts in the SAM Config repository to:

1. Create a repository seeded with our app starter using `create_repo.py`
2. Examine the new repository and the branches
3. Clone the New Repository
4. Configure a `test` Pipeline in the SAM Config repository using `config.py pipeline`
5. Deploy the `test` Pipeline from the SAM Config repository using `deploy.py pipeline`
6. Examine the Pipeline and CloudFormation process
7. Test the endpoint
8. Make changes and merge changes to `test` (to invoke the pipeline)
9. Watch a Pipeline through the CLI
10. Configure a `prod` Pipeline in the SAM Config repository
11. Deploy the `prod` Pipeline from the SAM Config repository
12. Perform a complete application deployment cycle from `dev` to `prod`
13. Deployment Strategies: `TEST` vs `PROD`

## 1. Create a Repository and Seed it Using `create_repo.py`

> If you have not yet acquainted yourself with the SAM Config Repository Documentation for Developers please do so as it provides helpful information about the scripts contained within.

We'll start by using your organization's SAM Config Repository (make sure it is cloned and most recent changes pulled) to your machine.

```bash
# Perform these commands from your organization's SAM Config Repository
git pull
./cli/create_repo.py advanced-8-ball --profile your-profile-if-not-default
```

Choose `00-basic-apigw-lambda-nodejs.zip` from the prompt.

## 2. Repository Configuration

Go to GitHub or the AWS CodeCommit Console and explore the repository. The `main` and `test` branches will be empty, and your code will be in the `dev` branch.

> The tutorials will use the `dev-test-main` branch merge strategy to get code from development to production (`main` branch). For more information see [Default Git Branch Workflow](./tutorials/default-git-branch-workflow.md)

### CodeCommit

If using CodeCommit, your repository was also given AWS resource tags to assist in account management. Note that resource tags are not the same as git tags. Resource tags are a feature of AWS as a type of meta data used to organize, manage, and report. Git tags are snapshots of repository contents and are a feature of all git repositories.

To view your resource tags go to your repository settings and select the "Repository Tags" tab.

Typical tags you will see are "CostCenter," "Creator," "Department," and "Owner" along with other tags your organization has configured for repository initialization.

## 3. Clone New Repository

Go back to your terminal for the SAM Config repository and copy the URL in the confirmation message for "Clone URL (HTTPS)".

Open a new terminal window (we want to keep the SAM Config repository open) and go to where you store your directories.

In the CLI, type `git clone` and paste the URL you copied.

```bash
git clone https://the-git-url-you-copied
cd advanced-8-ball
```

Check out the `dev` branch.

```bash
git checkout dev
```

Inspect the `template.yml` file. You'll notice various sections including:

- Metadata
- Parameters
- Conditions
- Globals
- Resources
- Outputs

Each section has a link to CloudFormation documentation to learn more about that section. It is recommended you review the linked documentation.

The resources included are:

- AWS::Serverless::Api (API Gateway)
- AWS::Serverless::Function (Lambda Function)
- AWS::IAM::Role (Lambda Execution Role)
- AWS::Lambda::Permission (Allows API Gateway to Execute Lambda)
- AWS::CloudWatch::Alarm (Alarm if there are errors to roll back a new deployment and alert an admin (Enabled in PROD environments only))
- AWS::SNS::Topic (Sends notification to individual or group noted in the `AlarmNotificationEmail` parameter)
- AWS::Logs::LogGroup (CloudWatch Log Group to send Lambda Logs to)

To learn more about each resource template and the properties available, just perform a search for the resource type (for example, `AWS::Serverless::Api`).

> Note that API Gateway logging is commented out as your account administrator will need to set up API Gateway logging at the account level first. Even if API Gateway logging is enabled, it is recommended for the first deploy to leave as-is so it isn't in the way of troubleshooting.

So, as you can see with the alarms and logs, there is some level of observability and monitoring. In future tutorials and application starters we will have access to additional monitoring options. It is recommended that you keep going with the tutorials to learn more and implement those features for production workloads.

We want to deploy the code as-is, so merge `dev` into `test`:

```bash
git checkout test
git merge dev
git push
```

The code is now in test, but we don't yet have a pipeline set up.

## 4. Configure a `test` Pipeline in the SAM Config repository using `config.py pipeline`

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

When prompted for a template, choose `template-pipeline-github.yml` (or `template-pipeline.yml` for CodeCommit).

Answer the template parameter prompts. If you need to view the description enter `?`. Most often you can accept the defaults.

If you make any mistakes you can continue answering the prompts and re-run the script to enter new values, or quit the script without saving by entering `^`. 

For additional information about the prompts and their values, refer to SAM Config Repo Documentation for Developers.

After completing the prompts, the script will display the command to deploy. We'll use this when deploying.

```text
Deploy commands are saved in the samconfig file for later reference. 
Since the template is in S3, 'sam deploy' will NOT work. 
Use ./cli/deploy.py instead 
./cli/deploy.py pipeline acme adv-8-ball test --profile acme-dev 
```

Before continuing it is a good idea to commit your configuration changes to the SAM Configuration Repository. Typically all changes can be pushed to the `main` branch as there should only be one source of truth. Check with your organization's policies to confirm.

```bash
git add --all
git commit -m "added test pipeline to adv-8-ball"
git push
```

## 5. Deploy the `test` Pipeline from the SAM Config repository using `deploy.py pipeline`

Copy, paste and execute the deploy command from the config output.

```bash
# Perform this command in the SAM Config Repo
./cli/deploy.py pipeline your-prefix adv-8-ball test --profile your-profile
```

Since the script is executing `sam deploy` in the background you will see the familiar `sam deploy` information and (if you set confirm to `true`) prompt to execute the changes.

If the `deploy.py` script performs `sam deploy` in the background, why not just do it directly? Well, we'll discuss the [limitations of `samconfig`](../atlantis-formatted-samconfig.md) later, but for now, just know that you can't point to a template in S3 when using `samconfig` files. The script downloads the template from S3 to a local temporary directory and  then performs the `sam deploy` using the local copy.

## 6. Examine the Pipeline and CloudFormation process

While the CloudFormation deployment is executing in your terminal, you can also open the AWS Web Console and watch the CloudFormation stack process there.

Once stack creation is complete, you can go to the Outputs tab of the pipeline stack and use the Pipeline link to view the pipeline progress which will kick off immediately upon stack completion.

As you wait for the pipeline to finish, you can check the account of the email address you entered for `AlarmNotificationEmail` as you should have received an email requesting you to confirm a subscription to be notified of the pipeline status. This is helpful for being notified of deployments.

## 7. Test the endpoint

Once the pipeline has completed, you can go back to your terminal with the CloudFormation stack output or the CloudFormation stack in the console.

Under the stack Output you will see Endpoint Test URL. Click on the link to make sure the application deployed correctly. You should see a standard prediction. (You can ask a Yes/No question prior to clicking on the link if you wish.)

If it wasn't successful be sure to resolve any issues before continuing.

## 8. Make changes and merge changes to test (to invoke the pipeline)

Right now the 8 Ball only makes a very vague prediction. Let's move the logic to a separate script, add some lucky numbers, and a certainty value.

In your `advanced-8-ball` repository, check out the `dev` branch:

```bash
git checkout dev
```

Now, add a new script file called `predictions.js` in the main `src` directory:

```js
// src/predictions.js

const getLuckyNumbers = function () {
	const numbers = [];
	const usedNumbers = new Set();
	
	while (numbers.length < 7) {
		const num = Math.floor(Math.random() * 98) + 1;
		if (!usedNumbers.has(num)) {
			usedNumbers.add(num);
			numbers.push(num);
		}
	}
	return numbers;
};

const answers = [ 
	"It is certain",
	"It is decidedly so",
	"Without a doubt",
	"Yes definitely",
	"You may rely on it",
	"As I see it, yes",
	"Most likely",
	"Outlook good",
	"Yes",
	"Signs point to yes",
	"Reply hazy try again",
	"Ask again later",
	"Better not tell you now",
	"Cannot predict now",
	"Concentrate and ask again",
	"Don't count on it",
	"My reply is no",
	"My sources say no",
	"Outlook not so good"
];

const getPrediction = function () {
  // Get a random number between 0 and the length of the answers array
	const rand = Math.floor(Math.random() * answers.length);
	return answers[rand];
}

const calculateCertainty = function (prediction, luckyNumbers) {

	// -- For entertainment purposes only --
	// -- No real science behind this --
	// -- Or magic (or is there?) --

	// sum the lucky numbers
	const sum = luckyNumbers.reduce((acc, num) => acc + num, 0);
	// get the length of the prediction
	const length = prediction.length;
	// multiply the length by sum
	const product = length * sum;
	// get a random number between 0 and product
	const rand = Math.floor(Math.random() * product);
	// calculate the certainty with 4 decimal places
	const certainty = (rand % 100000) / 100000;
	return certainty;
}

const getResponse = () => {
	const prediction = getPrediction();
	const luckyNumbers = getLuckyNumbers();
	const certainty = calculateCertainty(prediction, luckyNumbers);
	return {
		prediction,
		luckyNumbers,
		certainty
	};
}

// export
module.exports = {
	getResponse
}
```

Then modify `src/index.js` to remove the existing `answers` object and replace the setter of `prediction` in the `processRequest` method:

```js
// index.js

const predictions = require('./predictions.js');

// remove const answers = {...}

// ... handler code

const processRequest = function(event, context) {


	// Change prediction to equal the output from our new function

	// const rand = Math.floor(Math.random() * answers.length);
	// const prediction = answers[rand];
	const prediction = predictions.getResponse();

	// ... rest of code
};
```

Commit and push your changes to `dev`, then merge and push into `test` to kick-off a deploy:

```bash
git add --all
git commit -m "feat: added lucky numbers and certainty to prediction"
git push
git merge test
git push
```

## 9. Watch a Pipeline through the CLI

You can then check the CodePipeline console to see the progress, or utilize the terminal:

```bash
aws codepipeline get-pipeline-state --name your-pipeline-name
```

Replace `your-pipeline-name` with the name of your pipeline.

If you need to get the name of your pipeline you can list available pipelines:

```bash
aws codepipeline list-pipelines
```

To receive continuous status updates every 10 seconds:

```bash
watch -n 10 "aws codepipeline get-pipeline-state --name your-pipeline-name"
```

After the pipeline has completed, refresh your endpoint in the browser to see the changes.

## 10. Configure a `prod` Pipeline in the SAM Config repository

Go back to the terminal for your SAM Configuration Repository to add a second pipeline. This one for deploying a separate production instance.

We will again use `adv-8-ball` as our `ProjectId` argument but this time use `prod` as `StageId`. Replace `your-prefix` and `your-profile` with your own.

```bash
# Perform this command in the SAM Config Repo
./cli/config.py pipeline your-prefix adv-8-ball prod --profile your-profile
```

Since this will modify the SAM config file for your project, we will want to commit and push the changes to the SAM Config Repository:

```bash
git add --all
git commit -m "added prod pipeline to adv-8-ball"
git push
```

## 11. Deploy the `prod` Pipeline from the SAM Config repository

Perform the same copy, paste, execute on the deploy.py command as before and deploy the stack. Watch the CloudFormation updates in the terminal or console.

Just as you did for the `test` pipeline, you will receive an email subscription confirmation for updates regarding the production pipeline execution. Be sure to check your email and confirm your subscription.

Once the CloudFormation is done you can monitor the Pipeline in the terminal or console.

## 12. Perform a complete application deployment cycle from `dev` to `prod`

Go back and check out the `dev` branch.

Let's add additional features to the endpoint such as dice rolls and card deals.

```js
// src/predictions.js

// ...

// add new methods

const rollDie = function () {
	const rand = Math.floor(Math.random() * 6) + 1;
	return rand;
}

const getDiceRoll = function (numDice = 1) {
	const rolls = [];
	for (let i = 0; i < numDice; i++) {
		rolls.push(rollDie());
	}
	return rolls;
}

const drawCards = function (numCards = 1) {
	const suits = ['♠', '♥', '♦', '♣'];
	const values = ['A', '2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K'];
	const cards = [];
	const drawnCards = new Set();
	
	while (cards.length < numCards) {
		const suit = suits[Math.floor(Math.random() * suits.length)];
		const value = values[Math.floor(Math.random() * values.length)];
		const card = `${value}${suit}`;
		
		if (!drawnCards.has(card)) {
			drawnCards.add(card);
			cards.push(card);
		}
	}
	return cards;
}

// ...

// add call the new methods and add to the return value
const getResponse = () => {
	const prediction = getPrediction();
	const luckyNumbers = getLuckyNumbers();
	const diceRoll = getDiceRoll(3);
	const cards = drawCards(3);
	const certainty = calculateCertainty(prediction, luckyNumbers);
	return {
		prediction,
		luckyNumbers,
		diceRoll,
		cards,
		certainty
	};
}
```

Commit and push your changes to `dev` and then merge into `test`:

```bash
git add --all
git commit -m "added dice and cards"
git push
git checkout test
git merge dev
git push
```

Verify the changes once deployment is complete. 

If everything is satisfactory, then you are ready to merge to production!

```bash
git checkout main
git merge test
git push
```

This time, as the code moves through the pipeline, you may notice that the CloudFormation stage takes longer than before.

This is because production branches (`beta`, `stage`, `main`) typically use a _gradual_ deployment method.

## 13. Deployment Strategies: `TEST` vs `PROD`

If you examine the application's CloudFormation template, you will notice conditionals based on whether the template parameter `DeployEnvironment` is set to `PROD` or `TEST`.

Any deployment environment that should mimic a **Production** environment is considered `PROD`. This includes the branches/stages `beta`, `stage`, and `main`. All other deployments should be considered `TEST`. (`DEV` is also available but only when deploying on a local machine).

`TEST` environments deploy with fewer resources and increased logging for debugging purposes. `PROD` environments are production-like environments where the additional resources (such as alarms and dashboards which incur additional cost) have a place to be previewed and tested in `beta` or `stage` before moving to production. The `beta` stage may also be used in conjunction with a CloudFront distribution to perform blue/green testing.

Another noticable difference is that `PROD` environments utilize gradual deployments. This means that after a deploy to a production-like environment, traffic is slowly shifted from the old version of the application to the new version. If errors occur in the new version during the shift, the production environment can revert back to the old version.

It is important to note that during a gradual deployment the CloudFormation status will remain as _In Progress_ while the shift is occuring. Also, if you refresh the endpoint URL you will occasionally shift between the old and new version.

### Features of `TEST` environments

- Alarms are not created (they cost money and will be reserved for PROD deployments)
- Dashboards are not created (they cost money and will be reserved for PROD deployments) (We don't use dashboards in this simple app starter but will see them in a future tutorial)
- Deployment is `AllAtOnce` and not gradual. This allows developers to see the effects of new code right away
- Lambda logging levels can be set higher to allow for greater debugging
- Shorter CloudWatch log retention

### Features of `PROD` environments

- Alarms are created
- Dashboards are created
- Deployment is gradual
- Lambda logging levels can be set lower, only logging important information such as final execution information, errors, and warnings.
- Longer CloudWatch log retention

### Don't Confuse `DeployEnvironment` with `NODE_ENV` or `StageId`

If you are familiar with Node package dependencies, you know that there are dependencies reserved for development that are not included when an application is deployed to production. This typically includes developer tools and testing packages.

Setting `NODE_ENV` to `development` on your local machine is fine, but by default the CodePipeline template sets the CodeBuild environment variable `NODE_ENV` to `production` as developer tools are not needed. Also, adding all the dev dependencies to your Lambda function takes up extra space and can prevent inspecting the Lambda code in the console.

So remember, `NODE_ENV` is always set to `production` during deployments.

Also, while the `test` branch and `test StageId` is named similar to `DeployEnvironment TEST`, the `DeployEnvironment PROD` can refer to any branch or `StageId` that should be _production-like_ for staging, beta testing, or production purposes. So a `PROD` environment can be assumed by `beta`, `stage`, `staging`, `main` or `prod` branches/stages.

Finally, you can use `DeployEnvironment` to set Lambda environment variables for internal testing and logging. For example, if you look at the environment variables in the Lambda template:

```yaml

Conditions:
  IsProduction: !Equals [!Ref DeployEnvironment, "PROD"]

Resources:

  AppFunction:
    Type: AWS::Serverless::Function
    Properties:

	# ....

      Environment:
        Variables:
          detailedLogs: !If [ IsProduction, "0",  "5"] # 0 for prod, 2-5 for non-prod
          deployEnvironment: !Ref DeployEnvironment
          paramStore: !Ref ParameterStoreHierarchy
          lambdaTimeoutInSeconds: !Ref FunctionTimeOutInSeconds # so we can calculate any external connection timeout in our code

```

The Lambda environment variable `detailedLogs` is set to `0` for `PROD` and `5` for anything that is not production-like (`TEST` and `DEV`)

The Lambda environment variable `deployEnvironment` can be used to shorten cache expiration for testing purposes if your function caches data.

### Use Sparingly

Traditionally development and test environments have utilized smaller memory and CPU size. While you could lower the memory on your Lambda function there is little need to do so as a Lambda function at rest incurs zero cost. Use this to your benefit so that you can run your code in a Lambda function that has the same memory size and speed as production.

Also, don't go overboard when turning on or off resources and features. The more complex your conditional resources become, the greater chance for error.

You'll receive the greatest benefits by turning off alarms, dashboards, and log retention when not in production.
