# Part III: Application Starter 02 API Gateway with Lambda using Cache-Data (Node.js)

## 1. Inspect and utilize various `cache` and `tools` features of the @63klabs/cache-data npm package

TODO

### Configuration

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

DynamoDb can be used to keep track of user profiles, cached data, sessions, static application data, and more. When ever your data does not have multiple keys, does not require SQL or joins, it serves the purpose of being a reliable, scalable (serverless) and low latency data source.

For this example, we will use a simple DynamoDb table to store responses with a unique identifier allowing the ability to retrieve and share a link with others during a period of time. The link will expire after a set amount of time such as 60 minutes or 24 hours.

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
	  BillingMode: PAY_PER_REQUEST
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