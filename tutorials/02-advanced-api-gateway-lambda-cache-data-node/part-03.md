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

#### config/index.js

TODO

### Caching

TODO

### Debug logs and Timer

TODO

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