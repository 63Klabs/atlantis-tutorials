# Part III: Application Starter 02 API Gateway with Lambda using Cache-Data (Node.js)

## 1. Inspect and utilize various `cache` and `tools` features of the @63klabs/cache-data npm package

It is always recommended to first deploy the application in its "Hello World" configuration to ensure proper permissions and configurations are established.

We will now go over the default "Hello World" configuration to understand how the pieces fit together before we make any changes.

As in previous examples, the Lambda code lives in the `application-infrastructure/src` directory.

### Configuration

The `config` directory within Lambda source (`src`) contains files, settings, and configurations that do not change during the execution of the Lambda function. It is loaded and initialized outside of the Lambda handler and therefore remains consistent for each deployment.

In the main function `index.js`, the configuration is initialized asynchronously and the handler must ensure that initialization has completed before moving on to code that utilizes it. However, we don't want to hold up any process that doesn't have to wait, so we will perform the `await` within the handler just before we need it.

```js
/* src/index.js */
/* Code is Simplified - Lines irrelevant to configuration topic removed */

const { Config } = require("./config");

/* initialize the Config */
Config.init(); // we will await completion in the handler

exports.handler = async (event, context) => {

	let response = null;

	try {

		// ... You can do things for the response that don't need the config here ...

		/* wait for Config.promise and THEN Config.prime to be settled as we need it before continuing. */
		await Config.promise(); // makes sure general config init is complete
		await Config.prime(); // makes sure all prime tasks (tasks that need to be completed AFTER init but BEFORE handler) are completed

		/* Process the request and wait for result */
		response = await process(event, context);
	} catch (error) {
		// ...
	} finally {
		/* Send the result back to API Gateway */
		return response;
	}
}
```

> Note: Since the `Config.init()` is performed OUTSIDE the `handler()`, it ONLY executes during the COLDSTART. Because of this, the `Config.promise()` and `Config.prime()` will already have been resolved on subsequent executions and will continue without pausing.

The `config` directory contains all the settings and methods that get the Lambda function ready. The provided configuration is set-up for many common use-cases which we will experiment with in this tutorial. However, you can organize it for your own needs. 

Let's take a look at that directory. You'll notice that it contains a main `index.js` file, and mix of `*.json` and `*.js` files.

#### settings.json

This file contains the main settings for the application which are static and do not change based upon other settings or environment variables.

You will find that the usefulness of this file will vary as your application evolves and you begin to utilize environment variables and incorporate logic in your settings.

This is true for any `*.json` file. If you require more dynamic settings, you may want to rename this to a `*.js` file instead. We will explore this later in the tutorial.

#### validations.js

This file contains the validation rules for incoming requests. These rules are applied automatically by the `Request` object which we will look at later. The exported property names much match the request parameters.

```js
// validations.js
const { tools: {DebugAndLog} } = require("@63klabs/cache-data");

const referrers = [
	"example.com"
];

/* Validation Utility functions */

const isString = (value) => {
	return typeof value === "string";
};

const isStringOfNumbers = (value) => {
	// using regex, check if all the characters are digits
	return /^\d+$/.test(value);
};

const categoryId = (category) => {
	if (!category) return false;
	if (!isString(category)) return false;
	if (category.length === 0) return false;
	return ['GRE', 'BRN', 'SEK'].includes(category);
};

const productId = (productId) => {
	if (!productId) return false;
	if (!isStringOfNumbers(productId)) return false;
	if (productId.length === 0) return false;
	return true;
};

const sizeFilter = (size) => {
	if (!size) return false;
	if (!isStringOfNumbers(size)) return false;
	if (size < 1 || size > 10) return false;
	return true;
};

/**
 * The exported alias must match the parameter name in the request coming in.
 * The Request object will automatically validate the parameter based on the function name and exclude any request parameter that does not have a check.
 * You can define and re-use simple checks such as isString for multiple parameters if that is all you need.
 */
module.exports = {
	referrers,
	parameters: {
		pathParameters: {
			categoryId: categoryId // /products/{categoryId}
			productId: productId // /products/{categoryId}/{productId}
		},
		queryParameters: {
			size: sizeFilter, // products?size=10
			cat: categoryId, // products?cat=BRN
			id: productId
		},
		// headerParameters: {},
		// cookieParameters: {},
		// bodyParameters: {},	
	}
};
```

#### connections.js

The `connections.js` file is similar to `settings.json` in that it exports settings, in this case an array of connections.

These connections can be to databases, APIs, or other services that your application needs to connect to.

Unlike a `.json` file, the `.js` file can be dynamic based on deployment environment (PROD vs TEST) or other deployment settings. Calculations and methods can be used to set values that may change from deployment to deployment (not execution to execution though!).

Here is an example of a connection array:

```js
const connections = [
	{
		name: "games",
		host: "api.chadkluck.net",
		path: "/games",
		cache: [
			{
				profile: "default",
				overrideOriginHeaderExpiration: true,
				defaultExpirationInSeconds: (DebugAndLog.isProduction() ? (24 * 60 * 60) : (5 * 60)),// , // 5 minutes for non-prod
				expirationIsOnInterval: true,
				headersToRetain: "",
				hostId: "chadkluck", // log entry label - only used for logging
				pathId: "games", // log entry label - only used for logging
				encrypt: true, // encrypt the data in the cache
			}
		]        
	}
]
```

The `name` property is arbitrary and will be used within the code to return the connection details when asked. The host and path of course correspond to the endpoint details (`https` is assumed).

The `cache` property is optional, but provides details on how the cache should perform. The `profile`, much like the `name` is so that your code can call it.

The `defaultExpirationInSeconds` property is how long you want the cache to live. In this example we set a longer cache in production compared to test environments. You don't have to write out the seconds as a mathematical formula, but it can be helpful in quickly determining what the seconds represent in case you don't have your multiples of 60 memorized.

The `expirationIsOnInterval` when set to `true` lets the cache know that it is good for UP TO 24 hours in this case, but MUST expire at midnight. Similarly a cache set for one hour would expire on the hour. And a cache set for 30 minutes would expire on the hour and 30 minutes after the hour. When set to `false` it will be a rolling expiration. For example a cache item set at 9:07 with a 60 minute expiration will expire at 10:07. This can be useful for ensuring set intervals but must be balanced with an onset of cache invalidations all at the same time, thereby increasing the number of requests sent to the endpoint at certain times of the day.

The `hostId` and `pathId` are just for logging purposes.

Finally `encrypt` is set to `true` in this example and for all practical purposes, should always be true except in early stages of development. This provides an extra layer of encryption on the data stored in the cache (which is in DynamoDB or S3). Instead of just relying on encryption at rest, it also applies an application layer of encryption preventing anyone with access to read records or objects in DynamoDB and S3 from accessing the data. It also ensures that data cannot be accidentally (or intentionally) used by another application without the key.

We'll see how to send a request to an endpoint through the cache later.

#### config/index.js

The `config/index.js` script pulls everything together into a `Config` object that must be initialized before the application can handle requests. (We saw this when walking through the initial code with the `awaits`).

At its simplest form, the `init()` method initializes other objects such as `ClientRequest`, `Response`, and `Cache` by passing the objects exported by validations, settings, and connections.

You can change the structure around (put the connections array into settings, change settings to `.js` etc) but the main point is to initialize `ClientRequest`, `Response`, `Cache` and any other object or class that may require initialization.

You can also import the `Config` class into other scripts to use the configuration information.

#### Secret Store and SSM Parameter Store

The `config/index.js` file also contains methods to retrieve secrets from AWS Secrets Manager and parameters from AWS Systems Manager Parameter Store.

For this it utilizes the AWS Parameters and Secrets Lambda Extension which you will have seen included as a layer in the application's CloudFormation template.

This extension was chosen as it uses a retrieval code base managed by AWS and can be set to refresh at regular intervals if the parameters are rotated.

You can find more about the extension in the AWS User Guide: ["Using Parameter Store parameters in AWS Lambda functions"](https://docs.aws.amazon.com/systems-manager/latest/userguide/ps-integration-lambda-extensions.html)

### Lambda Environment Variables vs Settings File

The `settings.json` file is a great place to put static settings that do not change between deployments or environments.

However, there are times when you want to have settings that change based upon the deployment environment (TEST vs PROD) or other deployment specific information.

For example, you may want to have a setting that turns on verbose logging in non-production environments, but turns it off in production.

You can accomplish this by using Lambda Environment Variables.

In the `settings.json` file, you can reference environment variables using the `process.env` object, but you will need to change it to a `.js` file and provide an export as demonstrated in `connections.js`.

### Debug logs and Timer

By now you will have run across `DebugAndLog` and `Timer` as we examined the handler and configuration files. These two Classes can be used to assist in debugging your application by sending data back to CloudWatch logs.

For more information see the [documentation for 63Klabs Cache Data tools](https://github.com/63Klabs/cache-data/blob/main/docs/features/tools/).

#### Debug

The `DebugAndLog` class provides methods to log debug, info, warn, and error messages to CloudWatch Logs.

You can control the level of logging by setting the `LOG_LEVEL` environment variable in your Lambda function. The levels are:

- ERROR - `DebugAndLog.error()`
- WARN - `DebugAndLog.warn()`
- INFO - `DebugAndLog.info()`
- MSG - `DebugAndLog.msg()`
- DIAG - `DebugAndLog.diag()`
- DEBUG - `DebugAndLog.debug()`

The `DebugAndLog` class WILL NOT output any logging sent to `MSG`, `DIAG`, or `DEBUG` in production.

For example, to log a debug message:

```js
DebugAndLog.debug("This is a debug message");
```

#### Timer

The `Timer` class can be used to measure the time taken for a block of code to execute. It is useful for performance monitoring and optimization.

```js
const timer = new Timer("MyFunction", true); // true means start now
// code to be timed
timer.stop();
```

You will usually want to place the timer creation outside of a `try/catch` block and stop the timer in a `finally` block:

```js
const timer = new Timer("MyFunction", true);
try {
	// code to be timed
} catch (error) {
	// handle error
} finally {
	timer.stop();
}
```

## 2. Identify components of the application structure

TODO

### Routes

TODO

### Requests

TODO

### Controllers

TODO

### Services

TODO

### Views and Utilities

TODO

### Models

TODO

### Responses

TODO

## 3. Dive deeper into controllers, services, models, and views

TODO

### Implement a copy of Example components for the Games API (using CachedData)

TODO

### Implement a basic service with a direct call to an endpoint with no caching (8 Ball)

TODO

### Implement a service using Data Access Object with an api key behind caching (Weather)

TODO

### Implement a service with a write and call to a DynamoDb table

DynamoDb can be used to keep track of user profiles, cached data, sessions, static application data, and more. When ever your data does not have multiple keys, does not require SQL or joins, DyamoDB can serve the purpose of being a reliable, scalable (serverless) and low latency data source.

For this example, we will use a simple DynamoDB table to store responses with a unique identifier allowing the ability to retrieve and share a link with others during a period of time. The link will expire after a set amount of time such as 60 minutes or 24 hours.

First, we will add our own DynamoDb table definition to our application template:

```yaml
Resources:
  # .....
  GameTokensTable:
	Type: AWS::DynamoDB::Table
	DeletionPolicy: Retain # Contains critical data
	UpdateReplacePolicy: Retain # Contains critical data
	Properties:
	  TableName: !Sub "${Prefix}-${ProjectId}-${StageId}-GameTokens"
      AttributeDefinitions: 
        - AttributeName: "game_token"
          AttributeType: "S"
      KeySchema: 
        - AttributeName: "game_token"
          KeyType: "HASH"
      TimeToLiveSpecification: # This will keep track of when automatically expire game tokens
        AttributeName: "expiry"
        Enabled: true
	  BillingMode: PAY_PER_REQUEST # As a web service it will be sporadic
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: !If [IsProduction, true, false] # set to true for non-prod as well if needed
```

Next, we will add the DynamoDb table information as an Environment variable for our Lambda function:

```yaml
Resources:
  # .....
  AppFunction:
    Type: AWS::Serverless::Function
	Properties:
	  Environment:
		Variables:
		  GAME_TOKENS_TABLE: !Ref GameTokensTable
```

Then, we need to add permission for our Lambda function to access the table using the Lambda Execution role:

```yaml
Resources:
  # .....
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    # .....
    Properties:
	  # .....
      Policies:
      - PolicyName: LambdaResourceAccessPolicies
        PolicyDocument:
          Statement:

			# ....

          - Sid: LambdaAccessToDynamoDBTableGameTokens
            Action:
            - dynamodb:GetItem
            - dynamodb:Scan
            - dynamodb:Query
            - dynamodb:BatchGetItem
            - dynamodb:PutItem
            - dynamodb:UpdateItem
            - dynamodb:BatchWriteItem
            Effect: Allow
            Resource: !GetAtt GameTokensTable.Arn		
```

Commit and push your changes to the test branch to ensure the stack deploys without errors. Using the AWS Web Console:

- Inspect the newly created DynamoDb table
- Verify the Execution role includes the permissions to access the table
- Verify the Lambda function has a new Environment variable for the table.

Now that we have added a DynamoDb table to our application infrastructure, we need to add the following functionality to our code:

- Generate a unique identifier to use as the `game_token`
- Using a new service, save the token to the DynamoDb table and return the token to the controller for use in the view
- For incoming requests, check for the token in the query string, validate it, and then retreive the response information from the table
- Properly handle an expired token

We'll do this in two steps to ensure it works:

1. Implement the saving of the token. Then deploy and verify.
2. Implement the retrieval of the token. Then deploy and verify.

### Implement a token generator in Utils

TODO

### Implement a service with a write and call to a DynamoDb table (Responses)

The `GameTokens` service will handle the creation and retrieval of game tokens.

Create a new service file `services/gameTokens.service.js` and implement the following:

```javascript
// gameTokens.service.js

const { 
	tools: {
		DebugAndLog,
		Timer,
		AWS
	}
} = require("@63klabs/cache-data");

const { Config } = require("../config");

const logIdentifier = "GameToken Service";

exports.fetch = async (query) => {
	return new Promise(async (resolve, reject) => {
		let results = {};
		// TODO: We'll fill this in later
		resolve(results);
	});
};

exports.put = async (data) => {
	return new Promise(async (resolve, reject) => {

		// TODO
		//  access id gen from utils
		// const result = await AWS.dynamo.put(params);

	});
}

```

TODO

### Add a fetch method to the GameToken service

```javascript

exports.fetch = async (query) => {

	return new Promise(async (resolve, reject) => {

		let results = {};
		const timer = new Timer(logIdentifier, true);
		DebugAndLog.debug(`${logIdentifier}: Query Received`, query);

		try {
			const tableName = process.env.GAME_TOKENS_TABLE;

			const params = {
			TableName: tableName,
			Key: {
				game_token: query.gameToken
			},
			};

			DebugAndLog.debug(`${logIdentifier}: Query to Table`, params);

			const result = await AWS.dynamo.get(params);

			if (result.Item && result.Item.response) {
				DebugAndLog.debug(`${logIdentifier}: Retrieved data from DynamoDB table: ${tableName}`);
				results = { success: true, data: result.Item.response};
			} else {
				DebugAndLog.debug(`${logIdentifier}: Game Token ${gameToken} not found in DynamoDB table: ${tableName}`);
				results = {success: false, data: {}};
			}
		} catch (error) { 
			DebugAndLog.error(`${logIdentifier}: Error fetching Game Token ${gameToken} from DynamoDB: ${error.message}`, error.stack);
			results = {success: false, data: null};
		} finally {
			timer.stop();
		}

		resolve(results);

	});
};
```

### Static, Sample, and Test data

TODO

### Create a Controller and View utilizing the four data services

TODO

## Part III Summary

TODO

[Move on to Part IV](./part-04.md)