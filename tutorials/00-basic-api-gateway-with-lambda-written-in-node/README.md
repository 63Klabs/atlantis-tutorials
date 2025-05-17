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
7. Make changes and merge changes to test (to invoke the pipeline)
8. Watch a Pipeline through the CLI
9. Configure a `prod` Pipeline in the SAM Config repository
10. Deploy the `prod` Pipeline from the SAM Config repository
11. Perform a complete application deployment cycle from `dev` to `prod`

## 1. Create a Repository and Seed it Using a Script

> If you have not yet acquainted yourself with the SAM Config Repository Documentation for Developers please do so as it provides helpful information about the scripts contained within.

We'll start by using your organization's SAM Config Repository (make sure it is cloned (and most recent changes pulled!)) to your machine.

```bash
# Perform these commands from your organization's SAM Config Repository
git pull
./cli/create_repo.py advanced-8-ball
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

Before continuing it is a good idea to commit your configuration changes to the SAM Configuration Repository. Typically all changes can be pushed to the `main` branch as there should only be one source of truth. Check with your organization's policies to confirm.

```bash
git add --all
git commit -m "added test pipeline to adv-8-ball"
git push
```

## 4. Deploy the `test` Pipeline from the SAM Config repository

Copy, paste and execute the deploy command you copied from the config output.

As mentioned, since we are referencing a template stored in S3, the deploy script actually downloads the template to a temporary location on your local machine, and then executes the standard `aws sam deploy` command in the background.

Since the script is executing `aws sam deploy` in the background you will see the familiar `sam deploy` information and (if you set confirm to `true`) prompt to execute the changes.

## 5. Examine the Pipeline and CloudFormation process

While the CloudFormation deployment is executing in your terminal, you can open the AWS Console and also see the CloudFormation stack events there.

Once stack creation is complete, you can go to the Outputs tab of the pipeline stack and use the Pipeline link to view the pipeline progress which will kick off immediately upon stack completion.

As you wait for the pipeline to finish, you can check the account of the email address you entered for `AlarmNotificationEmail` as you should have received an email requesting you to confirm a subscription to be notified of the pipeline status. This is helpful for being notified of deployments.

## 6. Test the endpoint

Once the pipeline has completed, you can go back to your terminal with the CloudFormation stack output or the CloudFormation stack in the console.

Under the stack Output you will see Endpoint Test URL. Click on the link to make sure the application deployed correctly. You should see a standard prediction. (You can ask a Yes/No question prior to clicking on the link if you wish.)

If it wasn't successful be sure to resolve any issues before continuing.

## 7. Make changes and merge changes to test (to invoke the pipeline)

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

## 8. Watch a Pipeline through the CLI

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

## 9. Configure a `prod` Pipeline in the SAM Config repository

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

## 10. Deploy the `prod` Pipeline from the SAM Config repository

Perform the same copy, paste, execute on the deploy.py command as before and deploy the stack. Watch the CloudFormation updates in the terminal or console.

Just as you did for the `test` pipeline, you will receive an email subscription confirmation for updates regarding the production pipeline execution. Be sure to check your email and confirm your subscription.

Once the CloudFormation is done you can monitor the Pipeline in the terminal or console.

## 11. Perform a complete application deployment cycle from `dev` to `prod`

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
