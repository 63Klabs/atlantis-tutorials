# Part II: Application Starter 02 API Gateway with Lambda using Cache-Data (Node.js)

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

#### settings.js

This file contains the main settings for the application which can be hard coded, brought in through environment variables, or use logic to distinguish between `PROD` and `TEST` values.

You will find that the usefulness of this file will improve greatly as your application evolves and you begin to utilize environment variables and incorporate logic in your settings.

```js
const settings =  {
	"answer": 42,
	"baseUrl": process.env.BASE_URL || "",
	// "someNumSetting": process.env.SOME_SETTING || "all" // load environment variables - you should also implement validation
}

module.exports = settings;
```

#### validations.js

This file contains the validation rules for incoming requests. These rules are applied automatically by the `Request` object which we will look at later. The exported property names much match the request parameters.

```js
// validations.js
const ALLOWED_REFERRERS = ['*'];
const EXCLUDE_PARAMS_WITH_NO_VALIDATION_MATCH = false;

/* Validation Utility functions */

const isStringOfNumbers = (value) => {
	// using regex, check if all the characters are digits
	return /^\d+$/.test(value);
};

/**
 * Ensure id is in 'G-<8-char-hex>' format
 * @param {string} id - The id to validate
 * @returns {boolean} - True if the id is valid, false otherwise
 */
const idPathParameter = (id) => {
	if (!id.match(/^G\-[a-f0-9]{8}$/)) return false;
	return true;
};

/**
 * Ensure players is a number between 1 and 10
 * @param {string} players - The players to validate
 * @returns {boolean} - True if the players is valid, false otherwise
 */
const playersQueryParameter = (players) => {
	if (!isStringOfNumbers(players)) return false;
	const plyrs = parseInt(players, 10); // convert to Int
	if (plyrs < 0 || plyrs > 10) return false;
	return true;
};

module.exports = {
	referrers: ALLOWED_REFERRERS,
	parameters: {
		excludeParamsWithNoValidationMatch: EXCLUDE_PARAMS_WITH_NO_VALIDATION_MATCH,
		pathParameters: {
			id: idPathParameter,
		},
		queryStringParameters: {
			players: playersQueryParameter,
			BY_ROUTE: [{route: "GET:api/example?plyrs", validate: playersQueryParameter}]
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

Here is an example of a connection array:

```js
const { tools: {DebugAndLog} } = require("@63klabs/cache-data");

const connections = [
		{
		name: "8ball",
		host: "api.chadkluck.net",
		path: "/8ball"
	},
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
				headersToRetain: [],
				hostId: "chadkluck", // log entry label - only used for logging
				pathId: "games", // log entry label - only used for logging
				encrypt: true, // encrypt the data in the cache
			}
		]        
	}
]

module.exports = connections;
```

The `name` property is arbitrary and will be used within the code to return the connection details when asked. The host and path of course correspond to the endpoint details (`https` is assumed).

**Cache Profile Properties:**
- **profile**: Named cache configuration (allows multiple profiles per connection)
- **overrideOriginHeaderExpiration**: Ignore cache headers from the origin server
- **defaultExpirationInSeconds**: How long to cache responses (environment-specific)
- **expirationIsOnInterval**: Align cache expiration to time intervals
- **encrypt**: Whether to encrypt cached data (you should just encrypt always)
- **hostId/pathId**: Labels for logging and monitoring

We'll see how to send a request to an endpoint through the cache later.

#### config/index.js

The main configuration file (`config/index.js`) extends the `_ConfigSuperClass` from the @63klabs/cache-data package and provides a centralized initialization system for your Lambda function. Let's examine the key components:

```javascript
const { 
	cache: {
		Cache,
		CacheableDataAccess
	},
	tools: {
		DebugAndLog,
		Timer,
		CachedParameterSecrets,
		CachedSsmParameter,
		AppConfig,
	} 
} = require("@63klabs/cache-data");

const settings = require("./settings.js");
const validations = require("./validations.js");
const connections = require("./connections.js");
const responses = require("./responses.js");

class Config extends AppConfig {

	static init() {

		const timerConfigInit = new Timer("timerConfigInit", true);
				
		try {

			AppConfig.init( { settings, validations, connections, responses, debug: true } );

			// Cache settings
			Cache.init({
				secureDataKey: new CachedSsmParameter(process.env.PARAM_STORE_PATH+'CacheData_SecureDataKey', {refreshAfter: 43200}), // 12 hours
			});

			DebugAndLog.debug("Cache: ", Cache.info());

		} catch (error) {
			DebugAndLog.error(`Could not initialize Config ${error.message}`, error.stack);
		} finally {
			timerConfigInit.stop();
		};

		return AppConfig.promise();
	};

	static async prime() {
		return Promise.all([
			CacheableDataAccess.prime(),
			CachedParameterSecrets.prime()
		]);
	};
};

module.exports = {
	Config
};
```

**Key Initialization Steps:**

1. **AppConfig.init()**: Initializes the application using the data exported by the config/ files.
4. **Cache.init()**: Initializes the caching system with encryption key from SSM Parameter Store

**The Config.init() Process:**

The initialization follows this sequence:
- Creates a promise that resolves when all initialization is complete
- Starts a timer to track initialization performance
- Initializes each component in the correct order
- Logs cache information for debugging
- Handles any initialization errors gracefully

**The Config.prime() Method:**

After basic initialization, `prime()` ensures that:
- `CacheableDataAccess` is ready for external API calls
- `CachedParameterSecrets` has loaded any required secrets from AWS Systems Manager

This two-phase initialization allows the Lambda function to start quickly while ensuring all dependencies are ready before processing requests.

### Caching

The @63klabs/cache-data package provides sophisticated caching capabilities using in-memory, DynamoDB, and S3. The caching system is configured through the `connections.js` file and initialized in the main config.

The cache is initialized with an encryption key stored in AWS Systems Manager Parameter Store:

```javascript
Cache.init({
	secureDataKey: new CachedSSMParameter(process.env.PARAM_STORE_PATH+'CacheData_SecureDataKey', {refreshAfter: 43200}), // 12 hours
});
```

**Key Features:**
- **Encryption**: All cached data is encrypted using the secure data key IF the `cache` profile `encrypt` property is set to `true`
- **Automatic Refresh**: The encryption key is checked in SSM Parameter store every 12 hours (or whatever you designate)
- **SSM Integration**: Keys are securely stored in AWS Systems Manager

### Best Practices for Configuration and Caching

**Configuration Best Practices:**

1. **Environment-Specific Settings**: Use environment detection for different cache durations:
```javascript
// In connections.js
defaultExpirationInSeconds: (DebugAndLog.isProduction() ? (24 * 60 * 60) : (5 * 60))
```

2. **Error Handling**: Always wrap initialization in try-catch blocks:
```javascript
try {
	Cache.init({
		secureDataKey: new CachedSSMParameter(process.env.PARAM_STORE_PATH+'CacheData_SecureDataKey', {refreshAfter: 43200})
	});
	DebugAndLog.debug("Cache: ", Cache.info());
} catch (error) {
	DebugAndLog.error(`Could not initialize Config ${error.message}`, error.stack);
}
```

3. **Cache Strategy**: Use meaningful cache profiles for different data types:
```javascript
// Different profiles for different data lifecycles
{
	name: "user-data",
	cache: [
		{ profile: "currentStatus", defaultExpirationInSeconds: 300 },    // 5 minutes
		{ profile: "projAssignments", defaultExpirationInSeconds: 3600 },  // 1 hour
		{ profile: "userProfile", defaultExpirationInSeconds: 86400 }   // 24 hours
	]
}
```

4. **Encryption for Sensitive Data**: Always encrypt sensitive cached data:
```javascript
{
	profile: "secure",
	encrypt: true,
	defaultExpirationInSeconds: 1800 // 30 minutes for sensitive data
}
```

5. **Interval-Based Expiration**: Use interval expiration for consistent cache refresh:
```javascript
{
	expirationIsOnInterval: true, // Aligns cache expiration to time intervals
	defaultExpirationInSeconds: 3600 // All caches expire at the top of the hour
}
```

**Monitoring and Debugging:**

1. **Performance Tracking**: Use timers for all major operations:
```javascript
const timer = new Timer(`${logIdentifier}-${operation}`, true);
try {
	// ... operation code ...
} finally {
	const timing = timer.stop();
	if (timing.duration > 5000) { // Log slow operations
		DebugAndLog.log(`Slow operation detected: ${timing.label} took ${timing.duration}ms`);
	}
}
```

2. **Debug and Log**: Use DebugAndLog instead of `console.log` to output debugging information:
```javascript
const tools: {DebugAndLog} = require("@63klabs/cache-data");

const obj = {}; // you can pass objects as a second parameter

DebugAndLog.debug("Debug level", obj);
DebugAndLog.diag("Diagnostic level", obj);
DebugAndLog.message("Message level", obj);
DebugAndLog.info("Info level", obj);
DebugAndLog.warn("Warning level", obj);
DebugAndLog.error("Error", obj);
```

## 2. Identify components of the application structure

The atlantis-starter-02 application follows a Model-View-Controller (MVC) architectural pattern specifically adapted for serverless Lambda functions. This structure provides clear separation of concerns, making the code maintainable, testable, and scalable.

### Application Architecture Overview

The application processes requests through a well-defined flow:

```
API Gateway → Lambda Handler → Routes → Controller → Service → Model/DAO → External API/Database
                                                        ↓
API Gateway ← Response ← View ← Controller ← Service ← Model/DAO
```

**Key Architectural Principles:**

1. **Single Responsibility**: Each layer has a specific purpose
2. **Dependency Direction**: Higher layers depend on lower layers, not vice versa
3. **Separation of Concerns**: Business logic, data access, and presentation are separated
4. **Testability**: Each component can be tested independently

### Routes

The routing system acts as the traffic controller for incoming requests, determining which controller should handle each request based on HTTP method and path.

**Location**: `src/routes/index.js`

**Primary Responsibilities:**
- Parse incoming Lambda events into structured request objects
- Route requests to appropriate controllers based on method and path
- Handle request validation and security checks
- Manage error responses for invalid or unauthorized requests

**Routing Strategy:**
- **Simple Path Matching**: Uses `REQ.getResource(2)` to get the first two path segments
- **Method-First Routing**: Routes by HTTP method first, then by path
- **Extensible Design**: Easy to add new routes by extending the switch statement
- **Error Handling**: Provides appropriate HTTP status codes for various error conditions

**Best Practices for Routing:**
1. Keep routing logic simple and focused on path/method matching
2. Delegate complex logic to controllers
3. Use consistent path patterns (e.g., `/api/{resource}`)
4. Handle all HTTP methods explicitly, even if not supported

### Controllers

Controllers act as the orchestration layer, coordinating between the routing system and business logic services. They handle request-specific logic and prepare data for views.

**Location**: `src/controllers/`

**Primary Responsibilities:**
- Receive structured request properties from routes
- Extract and validate request parameters
- Call appropriate services with prepared queries
- Handle service responses and errors
- Format data through views before returning to routes

**Controller Patterns:**

1. **Parameter Extraction**: Extract needed parameters from request properties
2. **Query Preparation**: Create structured query objects for services
3. **Service Coordination**: Call one or more services as needed
4. **View Processing**: Format service responses through views
5. **Error Handling**: Log errors and return appropriate responses
6. **Performance Monitoring**: Use timers to track controller performance

**Best Practices for Controllers:**
1. Keep controllers thin - delegate business logic to services
2. Always use timers for performance monitoring
3. Log key operations for debugging
4. Handle errors gracefully without exposing internal details
5. Use consistent naming patterns (e.g., `get`, `post`, `put`, `delete`)

### Services

Services contain the core business logic of the application. They coordinate data access, implement business rules, and manage external API interactions through the caching system.

**Location**: `src/services/`

**Primary Responsibilities:**
- Implement business logic and rules
- Coordinate data access through models/DAOs
- Manage caching strategies for external APIs
- Transform and validate data
- Handle service-level error conditions

**Service Patterns:**

1. **Configuration Management**: Get connection and cache configurations
2. **Timeout Handling**: Set appropriate timeouts based on request deadlines
3. **Query Preparation**: Transform controller queries into DAO-specific queries
4. **Cached Data Access**: Use CacheableDataAccess for external API calls
5. **Error Recovery**: Handle errors gracefully and return safe defaults
6. **Performance Monitoring**: Track service execution time

**Service Types:**

1. **External API Services**: Use CacheableDataAccess for cached external calls
2. **Database Services**: Direct database operations (DynamoDB, RDS)
3. **Computation Services**: Business logic and data transformation
4. **Aggregation Services**: Combine data from multiple sources

### Views and Utilities

Views handle data presentation and transformation, while utilities provide reusable helper functions across the application.

#### Views

**Location**: `src/views/`

**Primary Responsibilities:**
- Transform service data into presentation format
- Filter and format data for client consumption
- Apply business rules for data display
- Handle data aggregation and summarization

**View Patterns:**

1. **Filter-Transform Pipeline**: Filter data first, then transform
2. **Performance Monitoring**: Use timers to track view processing time
3. **Safe Data Handling**: Always check for null/undefined data
4. **Consistent Output Format**: Return structured objects with predictable properties
5. **Utility Integration**: Use utility functions for common operations

#### Utilities

**Location**: `src/utils/`

**Primary Responsibilities:**
- Provide reusable helper functions
- Handle common operations (hashing, string manipulation, etc.)
- Encapsulate complex algorithms
- Support cross-cutting concerns

**Utility Categories:**

1. **Hash Functions**: SHA256 hashing with configurable output length
2. **Helper Functions**: Query string building, URI generation
3. **Validation Functions**: Input validation and sanitization
4. **Data Transformation**: Common data manipulation operations

### Models

Models represent the data access layer, implementing the Data Access Object (DAO) pattern for external APIs and databases. They handle the technical details of data retrieval and provide a clean interface for services.

**Location**: `src/models/`

**Primary Responsibilities:**
- Implement data access patterns (DAO)
- Handle external API communication
- Manage request/response formatting
- Provide data abstraction for services

## Part II Summary

In Part II we covered two key areas:

1. **Configuration and Caching** — How the `@63klabs/cache-data` library manages application settings through `connections.js`, including cache profiles with configurable expiration, encryption via SSM Parameter Store, environment-specific settings, and monitoring with `DebugAndLog` and `Timer`.
2. **Application Structure** — The MVC-based architecture of the atlantis-starter-02 application, tracing the request flow from API Gateway through Routes, Controllers, Services, Views, and Utilities, with each layer maintaining a single responsibility and clear separation of concerns.

[Move on to Part IV](./part-04.md)