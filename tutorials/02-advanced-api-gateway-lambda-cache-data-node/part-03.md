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
		CachedSSMParameter,
		_ConfigSuperClass,
		ClientRequest,
		Response,
		Connections
	} 
} = require("@63klabs/cache-data");

const settings = require("./settings.json");
const validations = require("./validations.js");
const connections = require("./connections.js");

class Config extends _ConfigSuperClass {

	static async init() {
		
		_ConfigSuperClass._promise = new Promise(async (resolve, reject) => {

			const timerConfigInit = new Timer("timerConfigInit", true);
				
			try {

				ClientRequest.init( { validations } );
				Response.init( { settings } );
				_ConfigSuperClass._connections = new Connections(connections);

				// Cache settings
				Cache.init({
					secureDataKey: new CachedSSMParameter(process.env.PARAM_STORE_PATH+'CacheData_SecureDataKey', {refreshAfter: 43200}), // 12 hours
				});

				DebugAndLog.debug("Cache: ", Cache.info());
				
				resolve(true);
			} catch (error) {
				DebugAndLog.error(`Could not initialize Config ${error.message}`, error.stack);
				reject(false);
			} finally {
				timerConfigInit.stop();
			};
			
		});

	};

	static async prime() {
		return Promise.all([
			CacheableDataAccess.prime(),
			CachedParameterSecrets.prime()
		]);
	};
};
```

**Key Initialization Steps:**

1. **ClientRequest.init()**: Initializes request validation using the rules defined in `validations.js`
2. **Response.init()**: Sets up response formatting using settings from `settings.json`
3. **Connections**: Configures external API connections for caching
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

The @63klabs/cache-data package provides sophisticated caching capabilities using DynamoDB and S3. The caching system is configured through the `connections.js` file and initialized in the main config.

#### Cache Configuration

The cache is initialized with an encryption key stored in AWS Systems Manager Parameter Store:

```javascript
Cache.init({
	secureDataKey: new CachedSSMParameter(process.env.PARAM_STORE_PATH+'CacheData_SecureDataKey', {refreshAfter: 43200}), // 12 hours
});
```

**Key Features:**
- **Encryption**: All cached data is encrypted using the secure data key
- **Automatic Refresh**: The encryption key is refreshed every 12 hours
- **SSM Integration**: Keys are securely stored in AWS Systems Manager

#### Connection Profiles

External API connections are defined in `connections.js` with specific caching behaviors:

```javascript
const connections = [
	{
		name: "games",
		host: "api.chadkluck.net",
		path: "/games",
		cache: [
			{
				profile: "default",
				overrideOriginHeaderExpiration: true,
				defaultExpirationInSeconds: (DebugAndLog.isProduction() ? (24 * 60 * 60) : (5 * 60)), // 24 hours prod, 5 minutes non-prod
				expirationIsOnInterval: true,
				headersToRetain: "",
				hostId: "chadkluck", // log entry label
				pathId: "games", // log entry label
				encrypt: true, // encrypt the data in the cache
			}
		]        
	}
]
```

**Cache Profile Properties:**
- **profile**: Named cache configuration (allows multiple profiles per connection)
- **overrideOriginHeaderExpiration**: Ignore cache headers from the origin server
- **defaultExpirationInSeconds**: How long to cache responses (environment-specific)
- **expirationIsOnInterval**: Align cache expiration to time intervals
- **encrypt**: Whether to encrypt cached data
- **hostId/pathId**: Labels for logging and monitoring

#### Using CacheableDataAccess

The `CacheableDataAccess` class provides the interface for making cached API calls. It uses a static method pattern where you pass your DAO function as a parameter:

```javascript
// In a Service
const { cache: { CacheableDataAccess } } = require("@63klabs/cache-data");
const { Config } = require("../config");
const { ExampleDao } = require("../models");

exports.fetch = async (query) => {
	try {
		// Get connection configuration
		let connection = Config.getConnection("games");
		let conn = connection.toObject();
		
		// Configure timeout
		conn.options ??= {};
		conn.options.timeout ??= query?.calcMsToDeadline(query?.deadline) ?? 8000;

		// Get cache profile
		let cacheCfg = connection.getCacheProfile("default");

		// Prepare query for DAO
		const daoQuery = {
			organizationCode: query?.organizationCode,
			gamePrimaryId: query?.gamePrimaryId
		};

		// Make cached request
		const cacheObj = await CacheableDataAccess.getData(
			cacheCfg,        // Cache configuration
			ExampleDao.get,  // DAO function to call
			conn,           // Connection configuration
			daoQuery        // Query parameters
		);

		// Extract response body
		return cacheObj.getBody(true);

	} catch (error) {
		DebugAndLog.error(`Error: ${error.message}`, error.stack);
		return null;
	}
};
```

**CacheableDataAccess.getData() Parameters:**
1. **cacheCfg**: Cache profile configuration from connections.js
2. **daoFunction**: The DAO method that makes the actual API call
3. **connection**: Connection configuration object
4. **query**: Parameters to pass to the DAO function

**Cache Behavior:**
1. **Cache Hit**: Returns cached data immediately if not expired
2. **Cache Miss**: Calls the DAO function, caches the result, returns data
3. **Cache Refresh**: Automatically updates cache when data expires
4. **Error Handling**: Can return stale cache data on API failures (configurable)

**Cache Response Object Methods:**
- `getBody(true)`: Returns the response body (true = parsed JSON)
- `getBody(false)`: Returns raw response body
- `getHeaders()`: Returns response headers
- `getStatusCode()`: Returns HTTP status code
- `isFromCache()`: Returns true if data came from cache

### Debug logs and Timer

The @63klabs/cache-data package provides robust logging and performance monitoring tools.

#### DebugAndLog Usage

The `DebugAndLog` utility provides environment-aware logging:

```javascript
const { tools: { DebugAndLog } } = require("@63klabs/cache-data");

// Different log levels
DebugAndLog.debug("Detailed debugging information", dataObject);
DebugAndLog.log("General information", "LABEL");
DebugAndLog.error("Error occurred", errorStack);

// Environment detection
if (DebugAndLog.isProduction()) {
	// Production-specific logic
} else {
	// Development/test logic
}
```

**Log Levels and Usage:**
- **debug()**: Detailed information for troubleshooting (only in non-production)
- **log()**: General operational information
- **error()**: Error conditions with stack traces
- **isProduction()**: Environment detection for conditional logic

**Practical Examples:**

```javascript
// Configuration initialization
DebugAndLog.debug("Cache: ", Cache.info());

// Request processing
DebugAndLog.debug(`${logIdentifier}: Query Received`, query);
DebugAndLog.debug(`${logIdentifier}: Query to DAO`, daoQuery);

// Results logging
DebugAndLog.debug(`${logIdentifier}: Retrieved games by organizationCode: ${query?.organizationCode}`, results);

// Error handling
DebugAndLog.error(`${logIdentifier}: Error: ${error.message}`, error.stack);

// Environment-specific caching
defaultExpirationInSeconds: (DebugAndLog.isProduction() ? (24 * 60 * 60) : (5 * 60))
```

#### Timer Usage

The `Timer` class provides performance monitoring for operations:

```javascript
const { tools: { Timer } } = require("@63klabs/cache-data");

// Basic timer usage
const timer = new Timer("operationName", true); // true = auto-start
// ... perform operation ...
timer.stop(); // Returns timing information

// Cold start monitoring
const coldStartInitTimer = new Timer("coldStartTimer", true);
// ... initialization code ...
if (coldStartInitTimer.isRunning()) { 
	DebugAndLog.log(coldStartInitTimer.stop(), "COLDSTART"); 
}
```

**Timer Features:**
- **Auto-start**: Begin timing immediately on creation
- **Performance tracking**: Measure operation duration
- **Cold start detection**: Special handling for Lambda cold starts
- **Logging integration**: Automatic formatting for log output

**Practical Examples:**

```javascript
// Service method timing
exports.fetch = async (query) => {
	const timer = new Timer(logIdentifier, true);
	try {
		// ... service logic ...
	} finally {
		timer.stop(); // Automatically logs timing
	}
};

// Configuration initialization timing
const timerConfigInit = new Timer("timerConfigInit", true);
try {
	// ... initialization code ...
} finally {
	timerConfigInit.stop();
}

// View processing timing
const viewTimer = new Timer(`Timer: ${logIdentifier}`, true);
// ... view processing ...
// Timer automatically stops and logs when method completes
```

**Performance Benefits:**
- Identify slow operations
- Monitor cold start impact
- Track performance across deployments
- Debug timeout issues

#### Best Practices for Configuration and Caching

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

3. **Timeout Configuration**: Set appropriate timeouts for external requests:
```javascript
conn.options ??= {};
conn.options.timeout ??= query?.calcMsToDeadline(query?.deadline) ?? 8000;
```

**Caching Best Practices:**

1. **Cache Key Strategy**: Use meaningful cache profiles for different data types:
```javascript
// Different profiles for different data lifecycles
{
	name: "user-data",
	cache: [
		{ profile: "short", defaultExpirationInSeconds: 300 },    // 5 minutes
		{ profile: "medium", defaultExpirationInSeconds: 3600 },  // 1 hour
		{ profile: "long", defaultExpirationInSeconds: 86400 }   // 24 hours
	]
}
```

2. **Encryption for Sensitive Data**: Always encrypt sensitive cached data:
```javascript
{
	profile: "secure",
	encrypt: true,
	defaultExpirationInSeconds: 1800 // 30 minutes for sensitive data
}
```

3. **Interval-Based Expiration**: Use interval expiration for consistent cache refresh:
```javascript
{
	expirationIsOnInterval: true, // Aligns cache expiration to time intervals
	defaultExpirationInSeconds: 3600 // All caches expire at the top of the hour
}
```

**Monitoring and Debugging:**

1. **Comprehensive Logging**: Log key operations with context:
```javascript
DebugAndLog.debug(`${logIdentifier}: Cache ${cacheObj.isFromCache() ? 'HIT' : 'MISS'}`, {
	organizationCode: query?.organizationCode,
	cacheAge: cacheObj.getAge(),
	statusCode: cacheObj.getStatusCode()
});
```

2. **Performance Tracking**: Use timers for all major operations:
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

3. **Cache Health Monitoring**: Monitor cache performance:
```javascript
// Log cache statistics periodically
DebugAndLog.debug("Cache Info", Cache.info());
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

**Key Components:**

```javascript
const process = async function(event, context) {
	// Create request and response objects
	const REQ = new ClientRequest(event, context);
	const RESP = new Response(REQ);

	if (REQ.isValid()) {
		const props = REQ.getProps();
		REQ.addPathLog(); // Log the request path for monitoring

		// Route based on HTTP method
		if (props.method === "GET") {
			// Route based on path structure
			let route = REQ.getResource(2); // Get first 2 path segments

			switch (route) {
				case "api/example":
					RESP.setBody(await Controllers.ExampleCtrl.get(props));
					break;
				default:
					RESP.reset({statusCode: 404});
					break;
			}
		} else {
			RESP.reset({statusCode: 405}); // Method not allowed
		}
	} else {
		RESP.reset({statusCode: 403}); // Forbidden
	}

	return RESP;
};
```

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

### Requests

The request handling system transforms raw Lambda events into structured, validated request objects that controllers can easily work with.

**Key Classes:**
- **ClientRequest**: Parses and validates incoming requests
- **Request Properties**: Structured data extracted from the Lambda event

**Request Processing Flow:**

```javascript
// 1. Create request object from Lambda event
const REQ = new ClientRequest(event, context);

// 2. Validate request against security rules
if (REQ.isValid()) {
	// 3. Extract structured properties
	const props = REQ.getProps();
	
	// props contains:
	// - method: HTTP method (GET, POST, etc.)
	// - pathParameters: URL path variables
	// - queryParameters: Query string parameters
	// - headers: Request headers
	// - body: Request body (parsed if JSON)
	// - calcMsToDeadline: Function to calculate remaining time
	// - deadline: Request deadline for timeout management
}
```

**Request Validation:**

The validation system uses rules defined in `config/validations.js`:

```javascript
// Example validation rules
module.exports = {
	referrers: ["example.com"], // Allowed referrer domains
	parameters: {
		pathParameters: {
			categoryId: categoryId, // Custom validation function
			productId: productId
		},
		queryParameters: {
			size: sizeFilter,
			cat: categoryId
		}
	}
};
```

**Validation Features:**
- **Parameter Validation**: Automatic validation of path and query parameters
- **Referrer Checking**: Validates request origin for security
- **Type Checking**: Ensures parameters match expected types
- **Custom Validators**: Supports custom validation functions

**Request Properties Structure:**

```javascript
const props = {
	method: "GET",
	pathParameters: {
		categoryId: "GRE",
		productId: "12345"
	},
	queryParameters: {
		size: "10",
		format: "json"
	},
	headers: {
		"content-type": "application/json",
		"user-agent": "..."
	},
	body: null, // or parsed JSON object for POST requests
	calcMsToDeadline: function(deadline) { /* ... */ },
	deadline: 1640995200000 // Unix timestamp
};
```

### Controllers

Controllers act as the orchestration layer, coordinating between the routing system and business logic services. They handle request-specific logic and prepare data for views.

**Location**: `src/controllers/`

**Primary Responsibilities:**
- Receive structured request properties from routes
- Extract and validate request parameters
- Call appropriate services with prepared queries
- Handle service responses and errors
- Format data through views before returning to routes

**Example Controller Structure:**

```javascript
// controllers/example.controller.js
const { tools: {Timer, DebugAndLog} } = require("@63klabs/cache-data");
const { ExampleSvc } = require("../services");
const { ExampleView } = require("../views");

const logIdentifier = "Example Controller GET";

exports.get = async (props) => {
	let data = null;
	const timer = new Timer(logIdentifier, true);
	DebugAndLog.debug(`${logIdentifier}: Properties received`, props);

	return new Promise(async (resolve, reject) => {
		try {
			// 1. Extract parameters from request properties
			let id = props?.pathParameters?.id;
			DebugAndLog.debug(`${logIdentifier}: id: ${id}`);

			// 2. Prepare query object for service
			const query = {
				id: id,
				calcMsToDeadline: props?.calcMsToDeadline,
				deadline: props?.deadline
			};

			// 3. Call service and format through view
			data = ExampleView.view(await ExampleSvc.fetch(query));

			DebugAndLog.debug(`${logIdentifier}: Example by Id: ${query?.id}`, data);

		} catch (error) {
			DebugAndLog.error(`${logIdentifier}: Error: ${error.message}`, error.stack);
			// Leave data as null for error cases
		} finally {
			timer.stop();
			resolve(data);
		}
	});
};
```

**Controller Patterns:**

1. **Parameter Extraction**: Extract needed parameters from request properties
2. **Query Preparation**: Create structured query objects for services
3. **Service Coordination**: Call one or more services as needed
4. **View Processing**: Format service responses through views
5. **Error Handling**: Log errors and return appropriate responses
6. **Performance Monitoring**: Use timers to track controller performance

**Controller Organization:**

```javascript
// controllers/index.js - Central export point
const ExampleCtrl = require("./example.controller");

module.exports = {
	ExampleCtrl
};
```

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

**Example Service Structure:**

```javascript
// services/example.service.js
const { 
	cache: { CacheableDataAccess },
	tools: { DebugAndLog, Timer }
} = require("@63klabs/cache-data");

const { Config } = require("../config");
const { ExampleDao } = require("../models");

const logIdentifier = "Example Service GET";

exports.fetch = async (query) => {
	return new Promise(async (resolve, reject) => {
		let results = {};
		const timer = new Timer(logIdentifier, true);
		DebugAndLog.debug(`${logIdentifier}: Query Received`, query);

		try {
			// 1. Get connection configuration
			let connection = Config.getConnection("games");
			let conn = connection.toObject();
			
			// 2. Configure timeout based on deadline
			conn.options ??= {};
			conn.options.timeout ??= query?.calcMsToDeadline(query?.deadline) ?? 8000;

			// 3. Get cache configuration
			let cacheCfg = connection.getCacheProfile("default");

			// 4. Prepare query for DAO
			const daoQuery = {
				organizationCode: query?.organizationCode,
				gamePrimaryId: query?.gamePrimaryId
			};

			DebugAndLog.debug(`${logIdentifier}: Query to DAO`, daoQuery);

			// 5. Make cached request through DAO
			const cacheObj = await CacheableDataAccess.getData(
				cacheCfg,        // Cache configuration
				ExampleDao.get,  // DAO function
				conn,           // Connection configuration
				daoQuery        // Query parameters
			);

			// 6. Extract response body
			results = cacheObj.getBody(true);

			DebugAndLog.debug(`${logIdentifier}: Retrieved games by organizationCode: ${query?.organizationCode}`, results);

		} catch (error) {
			DebugAndLog.error(`${logIdentifier}: Error: ${error.message}`, error.stack);
			// Return null or empty object for error cases
		} finally {
			timer.stop();
		}

		resolve(results);
	});
};
```

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

**Example View Structure:**

```javascript
// views/example.view.js
const { tools: { Timer } } = require("@63klabs/cache-data");
const utils = require('../utils');

const logIdentifier = "Example View";

// Generic filter to exclude null values
const filter = (dataItem = null) => {
	let include = true;
	if (dataItem == null) return false;
	return include;
};

// Generic transformer
const transform = (data) => {
	// Create an 8 character hash for unique IDs
	const hashId = utils.hash.takeLast(data, 8);

	const returnData = {
		id: `G-${hashId}`,
		display_name: data
	};

	return returnData;
};

// Main view function
exports.view = (resultsFromSvc) => {
	const viewTimer = new Timer(`Timer: ${logIdentifier}`, true);

	// Ensure we have an array to work with
	const dataArray = Array.isArray(resultsFromSvc?.gamechoices) ? 
		resultsFromSvc?.gamechoices : [];

	// Filter the data
	const filteredArray = dataArray.filter((item) => filter(item));

	// Transform the data
	const transformedArray = filteredArray.map((item) => transform(item));

	// Create final view structure
	const finalView = { 
		count: transformedArray.length, 
		items: transformedArray
	};

	viewTimer.stop();
	return finalView;
};
```

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

**Example Utility Structure:**

```javascript
// utils/hash.js
const crypto = require('crypto');

const isValidSliceInput = (input, n) => {
	if (typeof input !== 'string') {
		throw new Error('input must be a string');
	}
	if (typeof n !== 'number') {
		throw new Error('n must be a number');
	}
	if (n < 1 || n > 64) {
		throw new Error('n must be between 1 and 64');
	}
	return true;
};

const takeLast = (input, n=8) => {
	if (!isValidSliceInput(input, n)) {
		return "";
	}
	return crypto.createHash('sha256').update(input).digest('hex').slice(-n);
};

const takeFirst = (input, n=8) => {
	if (!isValidSliceInput(input, n)) {
		return "";
	}
	return crypto.createHash('sha256').update(input).digest('hex').slice(0, n);
};

module.exports = { takeLast, takeFirst };
```

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

**Model Architecture:**

The model layer uses a two-tier DAO pattern:

1. **Generic Base DAO**: Handles common API operations (`ApiExample.dao.js`)
2. **Specific Implementation DAO**: Handles endpoint-specific logic (`Example.dao.js`)

**Example Base DAO Structure:**

```javascript
// models/ApiExample.dao.js
const { tools: {DebugAndLog, APIRequest} } = require("@63klabs/cache-data");

class ApiExampleDao {
	constructor(connection) {
		this.request = {
			method: this._setRequestSetting(connection, "method", "GET"),
			uri: this._setRequestSetting(connection, "uri", ""),
			protocol: this._setRequestSetting(connection, "protocol", "https"),
			host: this._setRequestSetting(connection, "host", "api.chadkluck.net"),
			path: this._setRequestSetting(connection, "path", ""),
			body: this._setRequestSetting(connection, "body", null),
			note: this._setRequestSetting(connection, "note", "Get data from ApiSample"),
			parameters: this._setParameters(connection),
			headers: this._setHeaders(connection),
			options: this._setOptions(connection),
			cache: this._setCache(connection)
		};
	}

	_setParameters(connection) {
		if (!("parameters" in connection)) {
			connection.parameters = {};
		}
		// Set required parameters for all requests
		connection.parameters.format = "json";
		return connection.parameters;
	}

	_setHeaders(connection) {
		if (!("headers" in connection)) {
			connection.headers = {};
		}
		// Set standard headers
		connection.headers['content-type'] = "application/json";
		connection.headers['accept'] = "application/json";
		return connection.headers;
	}

	async get() {
		let response = null;
		try {
			response = await this._call();
			// Parse JSON response
			if (response.body !== "" && response.body !== null) {
				response.body = JSON.parse(response.body);
			}
		} catch (error) {
			DebugAndLog.error(`Error in call to ApiSample: ${error.message}`, error.stack);
		}
		return response;
	}

	async _call() {
		let response = null;
		try {
			const apiRequest = new APIRequest(this.request);
			response = await apiRequest.send();
		} catch (error) {
			DebugAndLog.error(`Error in ApiSample call: ${error.message}`, error.stack);
			response = APIRequest.responseFormat(false, 500, "Error in call()");
		}
		return response;
	}
}
```

**Example Specific DAO Structure:**

```javascript
// models/Example.dao.js
const { tools: {DebugAndLog, APIRequest} } = require("@63klabs/cache-data");
const { ApiExampleDao } = require("./ApiExample.dao");

const get = async (connection, query) => {
	return (new ExampleDao(connection, query).get());
};

class ExampleDao extends ApiExampleDao {
	constructor(connection, query) {
		super(connection);
		this.query = query;
		this._setRequest(connection);
	}

	async get() {
		let response = null;
		try {
			response = await super.get();
		} catch (error) {
			DebugAndLog.error(`Error in ExampleDao get: ${error.message}`, error.stack);
			response = APIRequest.responseFormat(false, 500, "Request failed");
		}
		return response;
	}

	async _call() {
		var response = null;
		try {
			response = await super._call();
		} catch (error) {
			DebugAndLog.error(`Error in ExampleDao call: ${error.message}`, error.stack);
			response = APIRequest.responseFormat(false, 500, "Request failed");
		}
		return response;
	}

	_setRequest() {
		DebugAndLog.debug(`Request: ${JSON.stringify(this.request)}`);
		this.request.note += " (example)";
		this.request.origNote = this.request.note;
	}
}

module.exports = { get };
```

**DAO Patterns:**

1. **Inheritance**: Specific DAOs extend base DAOs for common functionality
2. **Configuration**: Base DAOs handle standard request configuration
3. **Customization**: Specific DAOs customize requests for their endpoints
4. **Error Handling**: Multiple layers of error handling and recovery
5. **Logging**: Comprehensive logging for debugging and monitoring

**Model Organization:**

```javascript
// models/index.js - Central export point
const ExampleDao = require("./Example.dao");
const statusCodes = require("./static-data/statusCodes");

module.exports = {
	ExampleDao,
	statusCodes
};
```

**Data Organization:**

- **static-data/**: Static reference data (status codes, constants)
- **sample-data/**: Sample responses for testing and development
- **test-data/**: Test fixtures and expected outputs

### Responses

The response system handles the formatting and delivery of data back to API Gateway, ensuring consistent response structure and proper HTTP status codes.

**Key Classes:**
- **Response**: Main response object that formats data for API Gateway
- **Response Formatting**: Consistent structure for all API responses

**Response Processing Flow:**

```javascript
// 1. Create response object (in routes)
const RESP = new Response(REQ);

// 2. Set response body from controller
RESP.setBody(await Controllers.ExampleCtrl.get(props));

// 3. Handle errors with appropriate status codes
if (error) {
	RESP.reset({statusCode: 500});
}

// 4. Finalize and return to API Gateway
return RESP.finalize();
```

**Response Structure:**

```javascript
// Successful response
{
	statusCode: 200,
	headers: {
		"Content-Type": "application/json",
		"Access-Control-Allow-Origin": "*"
	},
	body: JSON.stringify({
		count: 3,
		items: [
			{ id: "G-a1b2c3d4", display_name: "Game 1" },
			{ id: "G-e5f6g7h8", display_name: "Game 2" },
			{ id: "G-i9j0k1l2", display_name: "Game 3" }
		]
	})
}

// Error response
{
	statusCode: 404,
	headers: {
		"Content-Type": "application/json"
	},
	body: JSON.stringify({
		message: "Resource not found"
	})
}
```

**Response Features:**

1. **Automatic Headers**: Sets appropriate content-type and CORS headers
2. **Status Code Management**: Handles HTTP status codes consistently
3. **JSON Serialization**: Automatically serializes response bodies
4. **Error Formatting**: Provides consistent error response structure
5. **Security Headers**: Includes security-related headers as needed

**Best Practices for Responses:**

1. Use appropriate HTTP status codes (200, 404, 500, etc.)
2. Include consistent response structure across all endpoints
3. Handle CORS headers for browser compatibility
4. Log response details for monitoring and debugging
5. Sanitize error messages to avoid exposing internal details

## 3. Dive deeper into controllers, services, models, and views

TODO

### Implement a copy of Example components for the Games API (using CachedData)

Now that we understand the application architecture, let's examine how the existing Example components work together to implement a complete API endpoint using the @63klabs/cache-data package.

The Games API demonstrates the full MVC flow with caching integration. Let's trace through each component to understand how they work together.

#### Understanding the Complete Flow

The Games API follows this request flow:

1. **Route** → Receives `/api/example` request and calls `ExampleCtrl.get()`
2. **Controller** → Extracts parameters and calls `ExampleSvc.fetch()`
3. **Service** → Uses `CacheableDataAccess` to call `ExampleDao.get()` with caching
4. **DAO** → Makes actual HTTP request to external API
5. **View** → Transforms and filters the response data
6. **Response** → Returns formatted JSON to API Gateway

#### Examining the Example Controller

The controller (`controllers/example.controller.js`) demonstrates the standard controller pattern:

```javascript
const { tools: {Timer, DebugAndLog} } = require("@63klabs/cache-data");
const { ExampleSvc } = require("../services");
const { ExampleView } = require("../views");

const logIdentifier = "Example Controller GET";

exports.get = async (props) => {
	let data = null;
	const timer = new Timer(logIdentifier, true);
	DebugAndLog.debug(`${logIdentifier}: Properties received`, props);

	return new Promise(async (resolve, reject) => {
		try {
			// 1. Extract path parameters
			let id = props?.pathParameters?.id;
			DebugAndLog.debug(`${logIdentifier}: id: ${id}`);

			// 2. Prepare query object for service
			const query = {
				id: id,
				calcMsToDeadline: props?.calcMsToDeadline,
				deadline: props?.deadline
			};

			// 3. Call service and format through view
			data = ExampleView.view(await ExampleSvc.fetch(query));

			DebugAndLog.debug(`${logIdentifier}: Example by Id: ${query?.id}`, data);

		} catch (error) {
			DebugAndLog.error(`${logIdentifier}: Error: ${error.message}`, error.stack);
			// Leave data as null for error cases
		} finally {
			timer.stop();
			resolve(data);
		}
	});
}
```

**Key Controller Patterns:**

1. **Performance Monitoring**: Uses `Timer` to track execution time
2. **Parameter Extraction**: Gets needed values from request properties
3. **Query Preparation**: Creates structured query object for service
4. **Service Coordination**: Calls service and formats response through view
5. **Error Handling**: Logs errors and returns null for failures
6. **Timeout Management**: Passes deadline information to service

#### Examining the Example Service

The service (`services/example.service.js`) demonstrates CacheableDataAccess integration:

```javascript
const { 
	cache: { CacheableDataAccess },
	tools: { DebugAndLog, Timer }
} = require("@63klabs/cache-data");

const { Config } = require("../config");
const { ExampleDao } = require("../models");

const logIdentifier = "Example Service GET";

exports.fetch = async (query) => {
	return new Promise(async (resolve, reject) => {
		let results = {};
		const timer = new Timer(logIdentifier, true);
		DebugAndLog.debug(`${logIdentifier}: Query Received`, query);

		try {
			// 1. Get connection configuration
			let connection = Config.getConnection("games");
			let conn = connection.toObject();
			
			// 2. Configure timeout
			conn.options ??= {};
			conn.options.timeout ??= query?.calcMsToDeadline(query?.deadline) ?? 
				Config.getSettings()?.externalRequestDefaultTimeoutInMs ?? 8000;

			// 3. Get cache profile
			let cacheCfg = connection.getCacheProfile("default");

			// 4. Prepare DAO query
			const daoQuery = {
				organizationCode: query?.organizationCode,
				gamePrimaryId: query?.gamePrimaryId
			};

			DebugAndLog.debug(`${logIdentifier}: Query to DAO`, daoQuery);

			// 5. Make cached request
			const cacheObj = await CacheableDataAccess.getData(
				cacheCfg,        // Cache configuration
				ExampleDao.get,  // DAO function to call
				conn,           // Connection configuration
				daoQuery        // Query parameters
			);

			// 6. Extract response body
			results = cacheObj.getBody(true);

			DebugAndLog.debug(`${logIdentifier}: Retrieved games by organizationCode: ${query?.organizationCode}`, results);

		} catch (error) {
			DebugAndLog.error(`${logIdentifier}: Error: ${error.message}`, error.stack);
		} finally {
			timer.stop();
		}

		resolve(results);
	});
};
```

**Key Service Patterns:**

1. **Connection Management**: Gets connection configuration from Config
2. **Timeout Configuration**: Sets appropriate timeouts based on request deadline
3. **Cache Integration**: Uses `CacheableDataAccess.getData()` for cached requests
4. **Query Transformation**: Converts controller query to DAO-specific format
5. **Response Extraction**: Gets parsed JSON from cache response object

#### Understanding CacheableDataAccess Integration

The `CacheableDataAccess.getData()` method is the core of the caching system:

```javascript
const cacheObj = await CacheableDataAccess.getData(
	cacheCfg,        // Cache profile from connections.js
	ExampleDao.get,  // DAO function that makes the actual API call
	conn,           // Connection configuration object
	daoQuery        // Parameters to pass to the DAO function
);
```

**How CacheableDataAccess Works:**

1. **Cache Check**: First checks if cached data exists and is not expired
2. **Cache Hit**: Returns cached data immediately if valid
3. **Cache Miss**: Calls the DAO function with provided parameters
4. **Cache Storage**: Stores the DAO response in cache with encryption
5. **Response Return**: Returns a cache response object with helper methods

**Cache Response Object Methods:**

- `getBody(true)`: Returns parsed JSON response body
- `getBody(false)`: Returns raw response body string
- `getHeaders()`: Returns response headers object
- `getStatusCode()`: Returns HTTP status code
- `isFromCache()`: Returns true if data came from cache
- `getAge()`: Returns age of cached data in seconds

#### Examining the Example DAO

The DAO layer uses inheritance to share common functionality:

**Base DAO (`models/ApiExample.dao.js`):**
```javascript
class ApiExampleDao {
	constructor(connection) {
		this.request = {
			method: this._setRequestSetting(connection, "method", "GET"),
			uri: this._setRequestSetting(connection, "uri", ""),
			protocol: this._setRequestSetting(connection, "protocol", "https"),
			host: this._setRequestSetting(connection, "host", "api.chadkluck.net"),
			path: this._setRequestSetting(connection, "path", ""),
			body: this._setRequestSetting(connection, "body", null),
			note: this._setRequestSetting(connection, "note", "Get data from ApiSample"),
			parameters: this._setParameters(connection),
			headers: this._setHeaders(connection),
			options: this._setOptions(connection),
			cache: this._setCache(connection)
		};
	}

	_setParameters(connection) {
		if (!("parameters" in connection)) {
			connection.parameters = {};
		}
		// Set required parameters for all requests
		connection.parameters.format = "json";
		return connection.parameters;
	}

	_setHeaders(connection) {
		if (!("headers" in connection)) {
			connection.headers = {};
		}
		// Set standard headers
		connection.headers['content-type'] = "application/json";
		connection.headers['accept'] = "application/json";
		return connection.headers;
	}

	async get() {
		let response = null;
		try {
			response = await this._call();
			// Parse JSON response
			if (response.body !== "" && response.body !== null) {
				response.body = JSON.parse(response.body);
			}
		} catch (error) {
			DebugAndLog.error(`Error in call to ApiSample: ${error.message}`, error.stack);
		}
		return response;
	}

	async _call() {
		let response = null;
		try {
			const apiRequest = new APIRequest(this.request);
			response = await apiRequest.send();
		} catch (error) {
			DebugAndLog.error(`Error in ApiSample call: ${error.message}`, error.stack);
			response = APIRequest.responseFormat(false, 500, "Error in call()");
		}
		return response;
	}
}
```

**Specific DAO (`models/Example.dao.js`):**

```javascript
const { ApiExampleDao } = require("./ApiExample.dao");

const get = async (connection, query) => {
	return (new ExampleDao(connection, query).get());
};

class ExampleDao extends ApiExampleDao {
	constructor(connection, query) {
		super(connection);
		this.query = query;
		this._setRequest(connection);
	}

	async get() {
		let response = null;
		try {
			response = await super.get();
		} catch (error) {
			DebugAndLog.error(`Error in ExampleDao get: ${error.message}`, error.stack);
			response = APIRequest.responseFormat(false, 500, "Request failed");
		}
		return response;
	}

	async _call() {
		var response = null;
		try {
			response = await super._call();
		} catch (error) {
			DebugAndLog.error(`Error in ExampleDao call: ${error.message}`, error.stack);
			response = APIRequest.responseFormat(false, 500, "Request failed");
		}
		return response;
	}

	_setRequest() {
		DebugAndLog.debug(`Request: ${JSON.stringify(this.request)}`);
		this.request.note += " (example)";
		this.request.origNote = this.request.note;
	}
}

module.exports = { get };
```

**DAO Patterns:**

1. **Inheritance**: Specific DAOs extend base DAOs for shared functionality
2. **Configuration**: Base DAO handles standard request setup
3. **Customization**: Specific DAO customizes requests for their endpoint
4. **Error Handling**: Multiple layers of error handling and recovery
5. **Request Building**: Constructs complete HTTP request objects

#### Examining the Example View

The view (`views/example.view.js`) demonstrates data transformation:

```javascript
const { tools: { Timer } } = require("@63klabs/cache-data");
const utils = require('../utils');

const logIdentifier = "Example View";

const filter = (dataItem = null) => {
	let include = true;
	if (dataItem == null) return false;
	return include;
}

const transform = (data) => {
	// Create an 8 character hash for unique IDs
	const hashId = utils.hash.takeLast(data, 8);

	const returnData = {
		id: `G-${hashId}`,
		display_name: data
	};

	return returnData;
}

exports.view = (resultsFromSvc) => {
	const viewTimer = new Timer(`Timer: ${logIdentifier}`, true);

	// Ensure we have an array to work with
	const dataArray = Array.isArray(resultsFromSvc?.gamechoices) ? 
		resultsFromSvc?.gamechoices : [];

	// Filter the data
	const filteredArray = dataArray.filter((item) => filter(item));

	// Transform the data
	const transformedArray = filteredArray.map((item) => transform(item));

	// Create final view structure
	const finalView = { 
		count: transformedArray.length, 
		items: transformedArray
	};

	viewTimer.stop();
	return finalView;
};
```

**View Patterns:**

1. **Filter-Transform Pipeline**: Filter data first, then transform
2. **Safe Data Handling**: Check for null/undefined data
3. **Utility Integration**: Use hash utilities for ID generation
4. **Consistent Output**: Return structured objects with count and items
5. **Performance Monitoring**: Track view processing time

#### Connection Configuration

The connection configuration (`config/connections.js`) defines caching behavior:

```javascript
const connections = [
	{
		name: "games",
		host: "api.chadkluck.net",
		path: "/games",
		cache: [
			{
				profile: "default",
				overrideOriginHeaderExpiration: true,
				defaultExpirationInSeconds: (DebugAndLog.isProduction() ? (24 * 60 * 60) : (5 * 60)),
				expirationIsOnInterval: true,
				headersToRetain: "",
				hostId: "chadkluck",
				pathId: "games",
				encrypt: true,
			}
		]        
	}
]
```

**Connection Features:**

1. **Environment-Specific Caching**: Different cache durations for prod vs non-prod
2. **Interval Expiration**: Aligns cache expiration to time intervals
3. **Encryption**: Encrypts cached data for security
4. **Override Headers**: Ignores cache headers from origin server
5. **Logging Labels**: Provides hostId and pathId for log identification

#### Testing the Games API

To test the complete Games API implementation:

1. **Deploy the Application**: Ensure the stack is deployed with the current code
2. **Test the Endpoint**: Make a GET request to `/api/example`
3. **Monitor Logs**: Check CloudWatch logs for the complete flow
4. **Verify Caching**: Make multiple requests to see cache hits vs misses

**Example Request:**
```bash
curl -X GET "https://your-api-gateway-url/api/example" \
  -H "Accept: application/json"
```

**Expected Response:**
```json
{
  "count": 3,
  "items": [
    {
      "id": "G-a1b2c3d4",
      "display_name": "Game Choice 1"
    },
    {
      "id": "G-e5f6g7h8", 
      "display_name": "Game Choice 2"
    },
    {
      "id": "G-i9j0k1l2",
      "display_name": "Game Choice 3"
    }
  ]
}
```

#### Key Takeaways from the Games API Example

1. **Separation of Concerns**: Each layer has a specific responsibility
2. **Caching Integration**: CacheableDataAccess provides transparent caching
3. **Error Handling**: Multiple layers of error handling ensure robustness
4. **Performance Monitoring**: Timers track performance at each layer
5. **Configuration Management**: Centralized connection and cache configuration
6. **Data Transformation**: Views provide clean, consistent output format

This Games API example demonstrates the complete MVC pattern with caching integration. The next sections will show how to implement different types of services using this same architectural pattern.

### Implement a basic service with a direct call to an endpoint with no caching (8 Ball)

Now let's implement a simpler service that makes direct API calls without caching. This demonstrates how to handle services that need real-time data or where caching isn't appropriate.

The 8 Ball service will provide random answers to questions, similar to a Magic 8 Ball toy. Since we want fresh, random responses each time, we won't use caching.

#### Step 1: Create the 8 Ball Controller

Create a new file `controllers/eightball.controller.js`:

```javascript
const { tools: {Timer, DebugAndLog} } = require("@63klabs/cache-data");

const { EightBallSvc } = require("../services");
const { EightBallView } = require("../views");

const logIdentifier = "EightBall Controller GET";

/**
 * Get function called by Router for 8 Ball predictions
 * @param {object} props - Request properties from router
 * @returns {object} data - Formatted 8 Ball response
 */
exports.get = async (props) => {
	let data = null;
	const timer = new Timer(logIdentifier, true);
	DebugAndLog.debug(`${logIdentifier}: Properties received`, props);

	return new Promise(async (resolve, reject) => {
		try {
			// Extract question from query parameters (optional)
			let question = props?.queryParameters?.question || "Will this work?";
			
			DebugAndLog.debug(`${logIdentifier}: question: ${question}`);

			// Prepare query object for service
			const query = {
				question: question,
				calcMsToDeadline: props?.calcMsToDeadline,
				deadline: props?.deadline
			};

			// Call service and format through view
			data = EightBallView.view(await EightBallSvc.fetch(query));

			DebugAndLog.debug(`${logIdentifier}: 8 Ball response for question: ${question}`, data);

		} catch (error) {
			DebugAndLog.error(`${logIdentifier}: Error: ${error.message}`, error.stack);
			// Leave data as null for error cases
		} finally {
			timer.stop();
			resolve(data);
		}
	});
};
```

#### Step 2: Create the 8 Ball Service

Create a new file `services/eightball.service.js`:

```javascript
const { 
	tools: {
		DebugAndLog,
		Timer,
		APIRequest
	}
} = require("@63klabs/cache-data");

const logIdentifier = "EightBall Service GET";

/**
 * Fetch 8 Ball prediction without caching
 * @param {object} query - Query parameters from controller
 * @returns {object} results - Raw API response
 */
exports.fetch = async (query) => {
	return new Promise(async (resolve, reject) => {
		let results = {};
		const timer = new Timer(logIdentifier, true);
		DebugAndLog.debug(`${logIdentifier}: Query Received`, query);

		try {
			// Configure the API request
			const request = {
				method: "GET",
				protocol: "https",
				host: "8ball.delegator.com",
				path: "/magic/JSON/question",
				parameters: {
					question: encodeURIComponent(query?.question || "Will this work?")
				},
				headers: {
					'accept': 'application/json',
					'user-agent': 'atlantis-starter-02/1.0'
				},
				options: {
					timeout: query?.calcMsToDeadline(query?.deadline) ?? 5000
				},
				note: "Get 8 Ball prediction"
			};

			DebugAndLog.debug(`${logIdentifier}: API Request`, request);

			// Make direct API call without caching
			const apiRequest = new APIRequest(request);
			const response = await apiRequest.send();

			// Parse JSON response
			if (response.statusCode === 200 && response.body) {
				try {
					results = JSON.parse(response.body);
					DebugAndLog.debug(`${logIdentifier}: Successful response`, results);
				} catch (parseError) {
					DebugAndLog.error(`${logIdentifier}: JSON Parse Error: ${parseError.message}`, parseError.stack);
					results = { error: "Failed to parse response" };
				}
			} else {
				DebugAndLog.error(`${logIdentifier}: API Error - Status: ${response.statusCode}`);
				results = { error: "API request failed" };
			}

		} catch (error) {
			DebugAndLog.error(`${logIdentifier}: Error: ${error.message}`, error.stack);
			results = { error: "Service unavailable" };
		} finally {
			timer.stop();
		}

		resolve(results);
	});
};
```

#### Step 3: Create the 8 Ball View

Create a new file `views/eightball.view.js`:

```javascript
const { 
	tools: {
		Timer
	}
} = require("@63klabs/cache-data");

const utils = require('../utils');

const logIdentifier = "EightBall View";

/**
 * Transform 8 Ball API response into consistent format
 * @param {object} resultsFromSvc - Raw service response
 * @returns {object} - Formatted 8 Ball response
 */
exports.view = (resultsFromSvc) => {
	const viewTimer = new Timer(`Timer: ${logIdentifier}`, true);

	let finalView = {};

	try {
		// Handle error responses
		if (resultsFromSvc?.error) {
			finalView = {
				success: false,
				error: resultsFromSvc.error,
				answer: "Ask again later",
				question: "Unknown"
			};
		} 
		// Handle successful responses
		else if (resultsFromSvc?.magic) {
			// Generate a unique ID for this prediction
			const predictionId = utils.hash.takeFirst(
				`${resultsFromSvc.magic.question}-${resultsFromSvc.magic.answer}-${Date.now()}`, 
				12
			);

			finalView = {
				success: true,
				id: `8B-${predictionId}`,
				question: resultsFromSvc.magic.question || "Unknown question",
				answer: resultsFromSvc.magic.answer || "Ask again later",
				type: resultsFromSvc.magic.type || "neutral",
				timestamp: new Date().toISOString()
			};
		}
		// Handle unexpected response format
		else {
			finalView = {
				success: false,
				error: "Unexpected response format",
				answer: "The spirits are unclear",
				question: "Unknown"
			};
		}

	} catch (error) {
		// Handle view processing errors
		finalView = {
			success: false,
			error: "View processing failed",
			answer: "The magic 8 ball is broken",
			question: "Unknown"
		};
	}

	viewTimer.stop();
	return finalView;
};
```

#### Step 4: Add Routing Configuration

Update the routing in `routes/index.js` to include the 8 Ball endpoint. Add this case to the switch statement:

```javascript
// In the GET method switch statement, add:
case "api/8ball":
	RESP.setBody(await Controllers.EightBallCtrl.get(props));
	break;
```

#### Step 5: Update Controller Index

Update `controllers/index.js` to export the new controller:

```javascript
const ExampleCtrl = require("./example.controller");
const EightBallCtrl = require("./eightball.controller");

module.exports = {
	ExampleCtrl,
	EightBallCtrl
};
```

#### Step 6: Update Service Index

Update `services/index.js` to export the new service:

```javascript
const ExampleSvc = require("./example.service");
const EightBallSvc = require("./eightball.service");

module.exports = {
	ExampleSvc,
	EightBallSvc
};
```

#### Step 7: Update View Index

Update `views/index.js` to export the new view:

```javascript
const ExampleView = require("./example.view");
const EightBallView = require("./eightball.view");

module.exports = {
	ExampleView,
	EightBallView
};
```

#### Key Differences from Cached Services

**Direct API Calls vs Caching:**

1. **No CacheableDataAccess**: Uses `APIRequest` directly
2. **No Connection Configuration**: Builds request object inline
3. **Real-time Data**: Each request gets fresh data
4. **Simpler Flow**: Fewer abstraction layers

**When to Use Direct Calls:**

- **Real-time Data**: Stock prices, random numbers, current time
- **Personalized Responses**: User-specific data that shouldn't be shared
- **Small, Fast APIs**: When caching overhead isn't worth it
- **Testing/Development**: When you need to see immediate changes

**Performance Considerations:**

- **Latency**: Direct calls have higher latency than cache hits
- **Rate Limits**: External APIs may have rate limiting
- **Reliability**: More dependent on external service availability
- **Cost**: May incur more charges from external APIs

#### Testing the 8 Ball Service

**Test Requests:**

```bash
# Basic request
curl -X GET "https://your-api-gateway-url/api/8ball"

# With custom question
curl -X GET "https://your-api-gateway-url/api/8ball?question=Will%20it%20rain%20tomorrow"
```

**Expected Response:**

```json
{
  "success": true,
  "id": "8B-a1b2c3d4e5f6",
  "question": "Will it rain tomorrow?",
  "answer": "Outlook not so good",
  "type": "negative",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

**Error Response:**

```json
{
  "success": false,
  "error": "Service unavailable",
  "answer": "Ask again later",
  "question": "Unknown"
}
```

#### Monitoring and Debugging

**Log Analysis:**

1. **Request Logs**: Check that questions are properly encoded
2. **API Response Logs**: Verify external API responses
3. **Timing Logs**: Monitor response times for performance
4. **Error Logs**: Track API failures and parsing errors

**Performance Monitoring:**

```javascript
// In CloudWatch, look for:
// - Timer logs showing response times
// - Error rates for external API calls
// - Request volume patterns
```

#### Best Practices for Direct API Services

1. **Timeout Management**: Always set appropriate timeouts
2. **Error Handling**: Handle all possible failure modes
3. **Input Validation**: Validate and sanitize user inputs
4. **Rate Limiting**: Respect external API rate limits
5. **Fallback Responses**: Provide meaningful fallbacks for failures
6. **Monitoring**: Track external API performance and reliability

The 8 Ball service demonstrates how to implement services that require real-time data without caching. This pattern is useful for APIs that provide dynamic, personalized, or time-sensitive information.

### Implement a service using Data Access Object with an api key behind caching (Weather)

Now let's implement a more sophisticated service that uses API key authentication with caching. This demonstrates the DAO pattern for external APIs that require authentication and benefit from caching.

The Weather service will fetch current weather data using an API key and cache the results to reduce API calls and improve performance.

#### Step 1: Add SSM Parameter for Weather Key

The weather API requires an API key. 

You can sign up for a FREE key without entering payment information by visiting:

- [OpenWeather Pricing](https://home.openweathermap.org/users/sign_up) and finding the Free Access option and Get API Key
  - [Full pricing Chart](https://openweathermap.org/full-price#current) - Get API Key
  - [Create New Account (Free)](https://home.openweathermap.org/users/sign_up)

> Note: Links may change, you may have to dig around for the free option.

Alternatively, you can check out [Weather API](https://www.weatherapi.com/) but may need to modify some of the examples to be compatible with their API.

Once you have an API key, add this to your buildspec.yml under the current call to `generate-put-ssm.py` for `CacheData_SecureDataKey`:

```yaml
      - python3 ./build-scripts/generate-put-ssm.py ${PARAM_STORE_HIERARCHY}WeatherApiKey
```

This will create a new SSM Parameter named `WeatherApiKey` with the initial value of `BLANK` (since we didn't pass it a value).

Best practice is to assign the value OUTSIDE of CloudFormation templates and build scripts. We'll perform a CLI call to set the value after we deploy.

#### Step 2: Add Weather Connection Configuration

Next, update `config/connections.js` to add the weather API connection along with the API key which we will fetch from SSM Parameter store:

```javascript
const { tools: {DebugAndLog} } = require("@63klabs/cache-data");

const connections = [
	{
		name: "games",
		host: "api.chadkluck.net",
		path: "/games",
		// ... existing games configuration ...     
	},
	{
		name: "weather",
		host: "api.openweathermap.org",
		path: "/data/2.5/weather",
		parameters: {
			q: "Chicago",
			units: "imperial",
			appid: new CachedSSMParameter(process.env.PARAM_STORE_PATH+'WeatherApi', {refreshAfter: 43200}), // 12 hours
		},
		cache: [
			{
				profile: "current",
				overrideOriginHeaderExpiration: true,
				defaultExpirationInSeconds: (DebugAndLog.isProduction() ? (10 * 60) : (2 * 60)), // 10 min prod, 2 min non-prod
				expirationIsOnInterval: true,
				headersToRetain: "",
				hostId: "openweather",
				pathId: "current",
				encrypt: true,
			}
		]
	}
]

module.exports = connections;
```

#### Step 3: Create the Weather Service

> TODO: change tutorial to make a direct connection first, then convert to DAO

Create a new file `services/weather.service.js`:

```javascript
const { 
	cache: { 
		CacheableDataAccess 
	},
	tools: {
		DebugAndLog,
		Timer
	}
} = require("@63klabs/cache-data");

const { Config } = require("../config");
const { WeatherDao } = require("../models");

const logIdentifier = "Weather Service GET";

/**
 * Fetch current weather data with caching
 * @param {object} query - Query parameters from controller
 * @returns {object} results - Weather data
 */
exports.fetch = async (query) => {
	return new Promise(async (resolve, reject) => {
		let results = {};
		const timer = new Timer(logIdentifier, true);
		DebugAndLog.debug(`${logIdentifier}: Query Received`, query);

		try {
			// Get weather connection configuration
			let connection = Config.getConnection("weather");
			let conn = connection.toObject();
			
			// Configure timeout
			conn.options ??= {};
			conn.options.timeout ??= query?.calcMsToDeadline(query?.deadline) ?? 
				Config.getSettings()?.externalRequestDefaultTimeoutInMs ?? 8000;

			// Get cache profile for current weather
			let cacheCfg = connection.getCacheProfile("current");

			// Prepare DAO query with location and options
			const daoQuery = {
				city: query?.city,
				lat: query?.lat,
				lon: query?.lon,
				units: query?.units || "metric",
				lang: query?.lang || "en"
			};

			DebugAndLog.debug(`${logIdentifier}: Query to DAO`, daoQuery);

			// Make cached request through DAO
			const cacheObj = await CacheableDataAccess.getData(
				cacheCfg,        // Cache configuration
				WeatherDao.get,  // DAO function
				conn,           // Connection configuration
				daoQuery        // Query parameters
			);

			// Extract response body
			results = cacheObj.getBody(true);

			// Log cache performance
			DebugAndLog.debug(`${logIdentifier}: Cache ${cacheObj.isFromCache() ? 'HIT' : 'MISS'} - Weather data for ${daoQuery.city || 'coordinates'}`, {
				fromCache: cacheObj.isFromCache(),
				statusCode: cacheObj.getStatusCode(),
				cacheAge: cacheObj.isFromCache() ? cacheObj.getAge() : 0
			});

		} catch (error) {
			DebugAndLog.error(`${logIdentifier}: Error: ${error.message}`, error.stack);
			results = { error: "Weather service unavailable" };
		} finally {
			timer.stop();
		}

		resolve(results);
	});
};
```

#### Step 6: Create the Weather Controller

Create a new file `controllers/weather.controller.js`:

```javascript
const { tools: {Timer, DebugAndLog} } = require("@63klabs/cache-data");

const { WeatherSvc } = require("../services");
const { WeatherView } = require("../views");

const logIdentifier = "Weather Controller GET";

/**
 * Get current weather data
 * @param {object} props - Request properties from router
 * @returns {object} data - Formatted weather response
 */
exports.get = async (props) => {
	let data = null;
	const timer = new Timer(logIdentifier, true);
	DebugAndLog.debug(`${logIdentifier}: Properties received`, props);

	return new Promise(async (resolve, reject) => {
		try {
			// Extract location parameters
			let city = props?.queryParameters?.city || props?.pathParameters?.city;
			let lat = props?.queryParameters?.lat;
			let lon = props?.queryParameters?.lon;
			let units = props?.queryParameters?.units || "metric";
			let lang = props?.queryParameters?.lang || "en";

			DebugAndLog.debug(`${logIdentifier}: Location - city: ${city}, lat: ${lat}, lon: ${lon}`);

			// Prepare query object for service
			const query = {
				city: city,
				lat: lat,
				lon: lon,
				units: units,
				lang: lang,
				calcMsToDeadline: props?.calcMsToDeadline,
				deadline: props?.deadline
			};

			// Call service and format through view
			data = WeatherView.view(await WeatherSvc.fetch(query));

			DebugAndLog.debug(`${logIdentifier}: Weather data for ${city || 'coordinates'}`, data);

		} catch (error) {
			DebugAndLog.error(`${logIdentifier}: Error: ${error.message}`, error.stack);
			// Leave data as null for error cases
		} finally {
			timer.stop();
			resolve(data);
		}
	});
};
```

#### Step 7: Create the Weather View

Create a new file `views/weather.view.js`:

```javascript
const { 
	tools: {
		Timer
	}
} = require("@63klabs/cache-data");

const utils = require('../utils');

const logIdentifier = "Weather View";

/**
 * Transform weather API response into consistent format
 * @param {object} resultsFromSvc - Raw service response
 * @returns {object} - Formatted weather response
 */
exports.view = (resultsFromSvc) => {
	const viewTimer = new Timer(`Timer: ${logIdentifier}`, true);

	let finalView = {};

	try {
		// Handle error responses
		if (resultsFromSvc?.error) {
			finalView = {
				success: false,
				error: resultsFromSvc.error,
				location: "Unknown",
				temperature: null,
				description: "Weather data unavailable"
			};
		}
		// Handle API error responses
		else if (resultsFromSvc?.cod && resultsFromSvc.cod !== 200) {
			finalView = {
				success: false,
				error: resultsFromSvc.message || "Weather API error",
				location: "Unknown",
				temperature: null,
				description: "Weather data unavailable"
			};
		}
		// Handle successful responses
		else if (resultsFromSvc?.main && resultsFromSvc?.weather) {
			// Generate a unique ID for this weather report
			const weatherId = utils.hash.takeFirst(
				`${resultsFromSvc.name}-${resultsFromSvc.coord?.lat}-${resultsFromSvc.coord?.lon}-${Math.floor(Date.now() / 600000)}`, // 10-minute intervals
				10
			);

			finalView = {
				success: true,
				id: `WX-${weatherId}`,
				location: {
					name: resultsFromSvc.name || "Unknown",
					country: resultsFromSvc.sys?.country || "Unknown",
					coordinates: {
						lat: resultsFromSvc.coord?.lat || null,
						lon: resultsFromSvc.coord?.lon || null
					}
				},
				weather: {
					main: resultsFromSvc.weather[0]?.main || "Unknown",
					description: resultsFromSvc.weather[0]?.description || "No description",
					icon: resultsFromSvc.weather[0]?.icon || null
				},
				temperature: {
					current: Math.round(resultsFromSvc.main.temp),
					feels_like: Math.round(resultsFromSvc.main.feels_like),
					min: Math.round(resultsFromSvc.main.temp_min),
					max: Math.round(resultsFromSvc.main.temp_max),
					unit: "°C" // Assuming metric units
				},
				conditions: {
					humidity: resultsFromSvc.main.humidity,
					pressure: resultsFromSvc.main.pressure,
					visibility: resultsFromSvc.visibility ? Math.round(resultsFromSvc.visibility / 1000) : null // Convert to km
				},
				wind: {
					speed: resultsFromSvc.wind?.speed || null,
					direction: resultsFromSvc.wind?.deg || null
				},
				timestamp: new Date().toISOString(),
				data_time: new Date(resultsFromSvc.dt * 1000).toISOString()
			};
		}
		// Handle unexpected response format
		else {
			finalView = {
				success: false,
				error: "Unexpected weather data format",
				location: "Unknown",
				temperature: null,
				description: "Weather data format error"
			};
		}

	} catch (error) {
		// Handle view processing errors
		finalView = {
			success: false,
			error: "Weather view processing failed",
			location: "Unknown",
			temperature: null,
			description: "Weather processing error"
		};
	}

	viewTimer.stop();
	return finalView;
};
```

#### Step 8: Update Index Files

Update `controllers/index.js`:
```javascript
const ExampleCtrl = require("./example.controller");
const EightBallCtrl = require("./eightball.controller");
const WeatherCtrl = require("./weather.controller");

module.exports = {
	ExampleCtrl,
	EightBallCtrl,
	WeatherCtrl
};
```

Update `services/index.js`:
```javascript
const ExampleSvc = require("./example.service");
const EightBallSvc = require("./eightball.service");
const WeatherSvc = require("./weather.service");

module.exports = {
	ExampleSvc,
	EightBallSvc,
	WeatherSvc
};
```

Update `views/index.js`:
```javascript
const ExampleView = require("./example.view");
const EightBallView = require("./eightball.view");
const WeatherView = require("./weather.view");

module.exports = {
	ExampleView,
	EightBallView,
	WeatherView
};
```

Update `models/index.js`:
```javascript
const ExampleDao = require("./Example.dao");
const WeatherDao = require("./Weather.dao");
const statusCodes = require("./static-data/statusCodes");

module.exports = {
	ExampleDao,
	WeatherDao,
	statusCodes
};
```

#### Step 9: Add Routing

Update `routes/index.js` to include the weather endpoint:

```javascript
// In the GET method switch statement, add:
case "api/weather":
	RESP.setBody(await Controllers.WeatherCtrl.get(props));
	break;
```


#### Step 3: Create the Weather Base DAO

Create a new file `models/ApiWeather.dao.js`:

```javascript
const { tools: {DebugAndLog, APIRequest} } = require("@63klabs/cache-data");

class ApiWeatherDao {
	constructor(connection) {
		this.request = {
			method: this._setRequestSetting(connection, "method", "GET"),
			uri: this._setRequestSetting(connection, "uri", ""),
			protocol: this._setRequestSetting(connection, "protocol", "https"),
			host: this._setRequestSetting(connection, "host", "api.openweathermap.org"),
			path: this._setRequestSetting(connection, "path", "/data/2.5/weather"),
			body: this._setRequestSetting(connection, "body", null),
			note: this._setRequestSetting(connection, "note", "Get weather data from OpenWeatherMap"),
			parameters: this._setParameters(connection),
			headers: this._setHeaders(connection),
			options: this._setOptions(connection),
			cache: this._setCache(connection)
		};
	}

	_setRequestSetting(connection, key, defaultValue) {
		if (!(key in connection)) {
			connection[key] = defaultValue;
		}
		return connection[key];
	}

	/**
	 * Set standard parameters for all weather API requests
	 * @param {object} connection - Connection configuration
	 * @returns {object} parameters
	 */
	_setParameters(connection) {
		if (!("parameters" in connection)) {
			connection.parameters = {};
		}

		// Add API key from environment variable
		connection.parameters.appid = process.env.OPENWEATHER_API_KEY;
		
		// Set default units to metric
		connection.parameters.units = "metric";

		return connection.parameters;
	}

	/**
	 * Set standard headers for all weather API requests
	 * @param {object} connection - Connection configuration
	 * @returns {object} headers
	 */
	_setHeaders(connection) {
		if (!("headers" in connection)) {
			connection.headers = {};
		}

		// Set standard headers
		connection.headers['accept'] = "application/json";
		connection.headers['user-agent'] = "atlantis-starter-02/1.0";

		return connection.headers;
	}

	_setOptions(connection) {
		if (!("options" in connection)) {
			connection.options = {};
		}
		return connection.options;
	}

	_setCache(connection) {
		if (!("cache" in connection)) {
			connection.cache = null;
		}
		return connection.cache;
	}

	/**
	 * Make the API request and parse JSON response
	 * @returns {object} response
	 */
	async get() {
		let response = null;
		try {
			response = await this._call();

			// Parse JSON response
			if (response.body !== "" && response.body !== null) {
				try {
					response.body = JSON.parse(response.body);
				} catch (parseError) {
					DebugAndLog.error(`Weather API JSON Parse Error: ${parseError.message}`, parseError.stack);
				}
			}

		} catch (error) {
			DebugAndLog.error(`Error in call to Weather API: ${error.message}`, error.stack);
		}

		return response;
	}

	/**
	 * Execute the HTTP request
	 * @returns {object} response
	 */
	async _call() {
		let response = null;
		try {
			const apiRequest = new APIRequest(this.request);
			response = await apiRequest.send();
		} catch (error) {
			DebugAndLog.error(`Error in Weather API call: ${error.message}`, error.stack);
			response = APIRequest.responseFormat(false, 500, "Weather API request failed");
		}
		return response;
	}
}

module.exports = {
	ApiWeatherDao
};
```


#### Step 4: Create the Weather Specific DAO

Create a new file `models/Weather.dao.js`:

```javascript
const { tools: {DebugAndLog, APIRequest} } = require("@63klabs/cache-data");
const { ApiWeatherDao } = require("./ApiWeather.dao");

const get = async (connection, query) => {
	return (new WeatherDao(connection, query).get());
};

class WeatherDao extends ApiWeatherDao {
	constructor(connection, query) {
		super(connection);
		this.query = query;
		this._setRequest(connection);
	}

	/**
	 * Get current weather data
	 * @returns {object} response
	 */
	async get() {
		let response = null;
		try {
			response = await super.get();
		} catch (error) {
			DebugAndLog.error(`Error in WeatherDao get: ${error.message}`, error.stack);
			response = APIRequest.responseFormat(false, 500, "Weather request failed");
		}
		return response;
	}

	/**
	 * Execute the weather API call
	 * @returns {object} response
	 */
	async _call() {
		let response = null;
		try {
			response = await super._call();
		} catch (error) {
			DebugAndLog.error(`Error in WeatherDao call: ${error.message}`, error.stack);
			response = APIRequest.responseFormat(false, 500, "Weather request failed");
		}
		return response;
	}

	/**
	 * Configure the request with query parameters
	 */
	_setRequest() {
		DebugAndLog.debug(`Weather Request: ${JSON.stringify(this.request)}`);

		// Add location parameters based on query
		if (this.query?.city) {
			this.request.parameters.q = this.query.city;
		} else if (this.query?.lat && this.query?.lon) {
			this.request.parameters.lat = this.query.lat;
			this.request.parameters.lon = this.query.lon;
		} else {
			// Default to a sample city if no location provided
			this.request.parameters.q = "London,UK";
		}

		// Override units if specified
		if (this.query?.units) {
			this.request.parameters.units = this.query.units;
		}

		// Add language if specified
		if (this.query?.lang) {
			this.request.parameters.lang = this.query.lang;
		}

		this.request.note += ` (${this.request.parameters.q || 'coordinates'})`;
		this.request.origNote = this.request.note;
	}
}

module.exports = {
	get
};
```

#### Key Features of the Weather Service

**API Key Management:**
- Securely stores API key in environment variables
- Automatically includes API key in all requests
- Uses CloudFormation parameters for deployment-time configuration

**Caching Strategy:**
- Caches weather data for 10 minutes in production, 2 minutes in development
- Uses interval-based expiration for consistent cache refresh
- Encrypts cached weather data for security

**Location Flexibility:**
- Supports city name queries (`?city=London,UK`)
- Supports coordinate queries (`?lat=51.5074&lon=-0.1278`)
- Provides default location if none specified

**Error Handling:**
- Handles API authentication errors
- Manages network timeouts and failures
- Provides meaningful error messages to clients

#### Testing the Weather Service

**Test Requests:**

```bash
# Get weather by city name
curl -X GET "https://your-api-gateway-url/api/weather?city=London,UK"

# Get weather by coordinates
curl -X GET "https://your-api-gateway-url/api/weather?lat=40.7128&lon=-74.0060"

# Get weather with different units
curl -X GET "https://your-api-gateway-url/api/weather?city=Paris,FR&units=imperial"
```

**Expected Response:**

```json
{
  "success": true,
  "id": "WX-a1b2c3d4e5",
  "location": {
    "name": "London",
    "country": "GB",
    "coordinates": {
      "lat": 51.5074,
      "lon": -0.1278
    }
  },
  "weather": {
    "main": "Clouds",
    "description": "overcast clouds",
    "icon": "04d"
  },
  "temperature": {
    "current": 15,
    "feels_like": 14,
    "min": 12,
    "max": 18,
    "unit": "°C"
  },
  "conditions": {
    "humidity": 72,
    "pressure": 1013,
    "visibility": 10
  },
  "wind": {
    "speed": 3.5,
    "direction": 230
  },
  "timestamp": "2024-01-15T10:30:00.000Z",
  "data_time": "2024-01-15T10:00:00.000Z"
}
```

#### Best Practices Demonstrated

1. **Secure API Key Management**: Uses environment variables and CloudFormation parameters
2. **Intelligent Caching**: Balances data freshness with API call efficiency
3. **Flexible Input Handling**: Supports multiple ways to specify location
4. **Comprehensive Error Handling**: Handles all failure modes gracefully
5. **Performance Monitoring**: Tracks cache hits/misses and response times
6. **Data Transformation**: Provides clean, consistent output format

The Weather service demonstrates how to implement authenticated external API calls with intelligent caching, providing a robust foundation for services that need to balance data freshness with performance and cost considerations.

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

First, let's create a utility function to generate unique tokens for our DynamoDB integration.

Create a new file `utils/token-generator.js`:

```javascript
const crypto = require('crypto');

/**
 * Generate a unique token with specified length and prefix
 * @param {string} prefix - Token prefix (default: 'GT')
 * @param {number} length - Token length excluding prefix (default: 16)
 * @returns {string} - Generated token
 */
const generateToken = (prefix = 'GT', length = 16) => {
	// Validate inputs
	if (typeof prefix !== 'string') {
		throw new Error('Prefix must be a string');
	}
	if (typeof length !== 'number' || length < 8 || length > 32) {
		throw new Error('Length must be a number between 8 and 32');
	}

	// Generate random bytes and convert to hex
	const randomBytes = crypto.randomBytes(Math.ceil(length / 2));
	const token = randomBytes.toString('hex').substring(0, length);
	
	return `${prefix}-${token}`;
};

/**
 * Generate a game token with timestamp component for uniqueness
 * @param {object} data - Data to include in token generation
 * @returns {string} - Generated game token
 */
const generateGameToken = (data = {}) => {
	const timestamp = Date.now().toString();
	const randomComponent = crypto.randomBytes(8).toString('hex');
	
	// Create a hash from the data and timestamp for additional uniqueness
	const dataString = JSON.stringify(data) + timestamp;
	const dataHash = crypto.createHash('sha256').update(dataString).digest('hex').substring(0, 8);
	
	return `GT-${dataHash}-${randomComponent}`;
};

/**
 * Calculate expiration timestamp
 * @param {number} minutesFromNow - Minutes from current time (default: 60)
 * @returns {number} - Unix timestamp for expiration
 */
const calculateExpiration = (minutesFromNow = 60) => {
	return Math.floor(Date.now() / 1000) + (minutesFromNow * 60);
};

/**
 * Validate token format
 * @param {string} token - Token to validate
 * @returns {boolean} - True if token format is valid
 */
const isValidTokenFormat = (token) => {
	if (typeof token !== 'string') {
		return false;
	}
	
	// Check for GT- prefix and proper length
	const tokenPattern = /^GT-[a-f0-9]{8}-[a-f0-9]{16}$/;
	return tokenPattern.test(token);
};

module.exports = {
	generateToken,
	generateGameToken,
	calculateExpiration,
	isValidTokenFormat
};
```

Update `utils/index.js` to include the token generator:

```javascript
const func = require('./helper-functions.js');
const hash = require('./hash.js');
const tokenGenerator = require('./token-generator.js');

module.exports = {
	func,
	hash,
	tokenGenerator
};
```

### Implement a service with a write and call to a DynamoDb table (GameTokens)

Now let's implement the complete GameTokens service that handles both storing and retrieving game tokens from DynamoDB.

Create a new file `services/gameTokens.service.js`:

```javascript
const { 
	tools: {
		DebugAndLog,
		Timer,
		AWS
	}
} = require("@63klabs/cache-data");

const utils = require('../utils');

const logIdentifier = "GameTokens Service";

/**
 * Store game data and return a shareable token
 * @param {object} data - Game data to store
 * @returns {object} - Result with token or error
 */
exports.put = async (data) => {
	return new Promise(async (resolve, reject) => {
		let results = {};
		const timer = new Timer(`${logIdentifier} PUT`, true);
		DebugAndLog.debug(`${logIdentifier} PUT: Data Received`, data);

		try {
			const tableName = process.env.GAME_TOKENS_TABLE;
			
			if (!tableName) {
				throw new Error('GAME_TOKENS_TABLE environment variable not set');
			}

			// Generate unique token
			const gameToken = utils.tokenGenerator.generateGameToken(data);
			
			// Calculate expiration (60 minutes from now)
			const expiration = utils.tokenGenerator.calculateExpiration(60);

			// Prepare DynamoDB item
			const item = {
				game_token: gameToken,
				response: data,
				created_at: Math.floor(Date.now() / 1000),
				expiry: expiration,
				metadata: {
					source: 'game-api',
					version: '1.0'
				}
			};

			const params = {
				TableName: tableName,
				Item: item,
				ConditionExpression: 'attribute_not_exists(game_token)' // Ensure uniqueness
			};

			DebugAndLog.debug(`${logIdentifier} PUT: Storing item in DynamoDB`, {
				tableName: tableName,
				gameToken: gameToken,
				expiresAt: new Date(expiration * 1000).toISOString()
			});

			// Store in DynamoDB
			await AWS.dynamo.put(params);

			results = {
				success: true,
				token: gameToken,
				expires_at: new Date(expiration * 1000).toISOString(),
				expires_in_minutes: 60
			};

			DebugAndLog.debug(`${logIdentifier} PUT: Successfully stored game token: ${gameToken}`);

		} catch (error) {
			DebugAndLog.error(`${logIdentifier} PUT: Error storing game token: ${error.message}`, error.stack);
			
			if (error.code === 'ConditionalCheckFailedException') {
				results = { success: false, error: 'Token already exists (very rare collision)' };
			} else {
				results = { success: false, error: 'Failed to store game data' };
			}
		} finally {
			timer.stop();
		}

		resolve(results);
	});
};

/**
 * Retrieve game data by token
 * @param {object} query - Query with gameToken
 * @returns {object} - Game data or error
 */
exports.fetch = async (query) => {
	return new Promise(async (resolve, reject) => {
		let results = {};
		const timer = new Timer(`${logIdentifier} GET`, true);
		DebugAndLog.debug(`${logIdentifier} GET: Query Received`, query);

		try {
			const tableName = process.env.GAME_TOKENS_TABLE;
			const gameToken = query?.gameToken;

			if (!tableName) {
				throw new Error('GAME_TOKENS_TABLE environment variable not set');
			}

			if (!gameToken) {
				results = { success: false, error: 'Game token is required' };
				resolve(results);
				return;
			}

			// Validate token format
			if (!utils.tokenGenerator.isValidTokenFormat(gameToken)) {
				results = { success: false, error: 'Invalid token format' };
				resolve(results);
				return;
			}

			const params = {
				TableName: tableName,
				Key: {
					game_token: gameToken
				}
			};

			DebugAndLog.debug(`${logIdentifier} GET: Querying DynamoDB`, params);

			const result = await AWS.dynamo.get(params);

			if (result.Item) {
				// Check if token has expired (DynamoDB TTL might not have cleaned it up yet)
				const currentTime = Math.floor(Date.now() / 1000);
				if (result.Item.expiry && result.Item.expiry < currentTime) {
					DebugAndLog.debug(`${logIdentifier} GET: Token ${gameToken} has expired`);
					results = { success: false, error: 'Token has expired' };
				} else {
					DebugAndLog.debug(`${logIdentifier} GET: Retrieved data from DynamoDB table: ${tableName}`);
					results = {
						success: true,
						data: result.Item.response,
						token: gameToken,
						created_at: new Date(result.Item.created_at * 1000).toISOString(),
						expires_at: new Date(result.Item.expiry * 1000).toISOString()
					};
				}
			} else {
				DebugAndLog.debug(`${logIdentifier} GET: Game Token ${gameToken} not found in DynamoDB table: ${tableName}`);
				results = { success: false, error: 'Token not found or has expired' };
			}

		} catch (error) {
			DebugAndLog.error(`${logIdentifier} GET: Error fetching Game Token ${query?.gameToken}: ${error.message}`, error.stack);
			results = { success: false, error: 'Failed to retrieve game data' };
		} finally {
			timer.stop();
		}

		resolve(results);
	});
};

/**
 * Delete a game token (optional cleanup method)
 * @param {object} query - Query with gameToken
 * @returns {object} - Success or error result
 */
exports.delete = async (query) => {
	return new Promise(async (resolve, reject) => {
		let results = {};
		const timer = new Timer(`${logIdentifier} DELETE`, true);
		DebugAndLog.debug(`${logIdentifier} DELETE: Query Received`, query);

		try {
			const tableName = process.env.GAME_TOKENS_TABLE;
			const gameToken = query?.gameToken;

			if (!tableName) {
				throw new Error('GAME_TOKENS_TABLE environment variable not set');
			}

			if (!gameToken) {
				results = { success: false, error: 'Game token is required' };
				resolve(results);
				return;
			}

			const params = {
				TableName: tableName,
				Key: {
					game_token: gameToken
				},
				ConditionExpression: 'attribute_exists(game_token)'
			};

			DebugAndLog.debug(`${logIdentifier} DELETE: Deleting from DynamoDB`, params);

			await AWS.dynamo.delete(params);

			results = { success: true, message: 'Token deleted successfully' };
			DebugAndLog.debug(`${logIdentifier} DELETE: Successfully deleted token: ${gameToken}`);

		} catch (error) {
			if (error.code === 'ConditionalCheckFailedException') {
				results = { success: false, error: 'Token not found' };
			} else {
				DebugAndLog.error(`${logIdentifier} DELETE: Error deleting Game Token ${query?.gameToken}: ${error.message}`, error.stack);
				results = { success: false, error: 'Failed to delete token' };
			}
		} finally {
			timer.stop();
		}

		resolve(results);
	});
};
```

### Create GameTokens Controller

Create a new file `controllers/gameTokens.controller.js`:

```javascript
const { tools: {Timer, DebugAndLog} } = require("@63klabs/cache-data");

const { GameTokensSvc } = require("../services");
const { GameTokensView } = require("../views");

const logIdentifier = "GameTokens Controller";

/**
 * Store game data and return a shareable token
 * @param {object} props - Request properties from router
 * @returns {object} data - Token response
 */
exports.post = async (props) => {
	let data = null;
	const timer = new Timer(`${logIdentifier} POST`, true);
	DebugAndLog.debug(`${logIdentifier} POST: Properties received`, props);

	return new Promise(async (resolve, reject) => {
		try {
			// Extract game data from request body
			const gameData = props?.body || {};
			
			if (Object.keys(gameData).length === 0) {
				data = GameTokensView.viewError('No game data provided');
				resolve(data);
				return;
			}

			DebugAndLog.debug(`${logIdentifier} POST: Game data`, gameData);

			// Call service to store data and get token
			const result = await GameTokensSvc.put(gameData);
			data = GameTokensView.viewPut(result);

			DebugAndLog.debug(`${logIdentifier} POST: Token creation result`, data);

		} catch (error) {
			DebugAndLog.error(`${logIdentifier} POST: Error: ${error.message}`, error.stack);
			data = GameTokensView.viewError('Failed to create game token');
		} finally {
			timer.stop();
			resolve(data);
		}
	});
};

/**
 * Retrieve game data by token
 * @param {object} props - Request properties from router
 * @returns {object} data - Game data response
 */
exports.get = async (props) => {
	let data = null;
	const timer = new Timer(`${logIdentifier} GET`, true);
	DebugAndLog.debug(`${logIdentifier} GET: Properties received`, props);

	return new Promise(async (resolve, reject) => {
		try {
			// Extract token from path parameters
			const gameToken = props?.pathParameters?.token;
			
			if (!gameToken) {
				data = GameTokensView.viewError('Game token is required');
				resolve(data);
				return;
			}

			DebugAndLog.debug(`${logIdentifier} GET: Retrieving token: ${gameToken}`);

			// Prepare query for service
			const query = {
				gameToken: gameToken,
				calcMsToDeadline: props?.calcMsToDeadline,
				deadline: props?.deadline
			};

			// Call service and format through view
			const result = await GameTokensSvc.fetch(query);
			data = GameTokensView.viewGet(result);

			DebugAndLog.debug(`${logIdentifier} GET: Token retrieval result`, data);

		} catch (error) {
			DebugAndLog.error(`${logIdentifier} GET: Error: ${error.message}`, error.stack);
			data = GameTokensView.viewError('Failed to retrieve game data');
		} finally {
			timer.stop();
			resolve(data);
		}
	});
};

/**
 * Delete a game token
 * @param {object} props - Request properties from router
 * @returns {object} data - Deletion result
 */
exports.delete = async (props) => {
	let data = null;
	const timer = new Timer(`${logIdentifier} DELETE`, true);
	DebugAndLog.debug(`${logIdentifier} DELETE: Properties received`, props);

	return new Promise(async (resolve, reject) => {
		try {
			// Extract token from path parameters
			const gameToken = props?.pathParameters?.token;
			
			if (!gameToken) {
				data = GameTokensView.viewError('Game token is required');
				resolve(data);
				return;
			}

			DebugAndLog.debug(`${logIdentifier} DELETE: Deleting token: ${gameToken}`);

			// Prepare query for service
			const query = {
				gameToken: gameToken,
				calcMsToDeadline: props?.calcMsToDeadline,
				deadline: props?.deadline
			};

			// Call service and format through view
			const result = await GameTokensSvc.delete(query);
			data = GameTokensView.viewDelete(result);

			DebugAndLog.debug(`${logIdentifier} DELETE: Token deletion result`, data);

		} catch (error) {
			DebugAndLog.error(`${logIdentifier} DELETE: Error: ${error.message}`, error.stack);
			data = GameTokensView.viewError('Failed to delete game token');
		} finally {
			timer.stop();
			resolve(data);
		}
	});
};
```

### Create GameTokens View

Create a new file `views/gameTokens.view.js`:

```javascript
const { 
	tools: {
		Timer
	}
} = require("@63klabs/cache-data");

const logIdentifier = "GameTokens View";

/**
 * Format token creation response
 * @param {object} resultsFromSvc - Service response
 * @returns {object} - Formatted response
 */
exports.viewPut = (resultsFromSvc) => {
	const viewTimer = new Timer(`Timer: ${logIdentifier} PUT`, true);

	let finalView = {};

	try {
		if (resultsFromSvc?.success) {
			finalView = {
				success: true,
				token: resultsFromSvc.token,
				share_url: `/api/tokens/${resultsFromSvc.token}`,
				expires_at: resultsFromSvc.expires_at,
				expires_in_minutes: resultsFromSvc.expires_in_minutes,
				message: "Game data stored successfully"
			};
		} else {
			finalView = {
				success: false,
				error: resultsFromSvc?.error || "Failed to create token",
				message: "Could not store game data"
			};
		}
	} catch (error) {
		finalView = {
			success: false,
			error: "View processing failed",
			message: "Token creation view error"
		};
	}

	viewTimer.stop();
	return finalView;
};

/**
 * Format token retrieval response
 * @param {object} resultsFromSvc - Service response
 * @returns {object} - Formatted response
 */
exports.viewGet = (resultsFromSvc) => {
	const viewTimer = new Timer(`Timer: ${logIdentifier} GET`, true);

	let finalView = {};

	try {
		if (resultsFromSvc?.success) {
			finalView = {
				success: true,
				token: resultsFromSvc.token,
				data: resultsFromSvc.data,
				created_at: resultsFromSvc.created_at,
				expires_at: resultsFromSvc.expires_at,
				message: "Game data retrieved successfully"
			};
		} else {
			finalView = {
				success: false,
				error: resultsFromSvc?.error || "Token not found",
				message: "Could not retrieve game data"
			};
		}
	} catch (error) {
		finalView = {
			success: false,
			error: "View processing failed",
			message: "Token retrieval view error"
		};
	}

	viewTimer.stop();
	return finalView;
};

/**
 * Format token deletion response
 * @param {object} resultsFromSvc - Service response
 * @returns {object} - Formatted response
 */
exports.viewDelete = (resultsFromSvc) => {
	const viewTimer = new Timer(`Timer: ${logIdentifier} DELETE`, true);

	let finalView = {};

	try {
		if (resultsFromSvc?.success) {
			finalView = {
				success: true,
				message: resultsFromSvc.message || "Token deleted successfully"
			};
		} else {
			finalView = {
				success: false,
				error: resultsFromSvc?.error || "Failed to delete token",
				message: "Could not delete game token"
			};
		}
	} catch (error) {
		finalView = {
			success: false,
			error: "View processing failed",
			message: "Token deletion view error"
		};
	}

	viewTimer.stop();
	return finalView;
};

/**
 * Format error response
 * @param {string} errorMessage - Error message
 * @returns {object} - Formatted error response
 */
exports.viewError = (errorMessage) => {
	return {
		success: false,
		error: errorMessage,
		message: "Request could not be processed"
	};
};
```

### Update Index Files and Routing

Update `controllers/index.js`:
```javascript
const ExampleCtrl = require("./example.controller");
const EightBallCtrl = require("./eightball.controller");
const WeatherCtrl = require("./weather.controller");
const GameTokensCtrl = require("./gameTokens.controller");

module.exports = {
	ExampleCtrl,
	EightBallCtrl,
	WeatherCtrl,
	GameTokensCtrl
};
```

Update `services/index.js`:
```javascript
const ExampleSvc = require("./example.service");
const EightBallSvc = require("./eightball.service");
const WeatherSvc = require("./weather.service");
const GameTokensSvc = require("./gameTokens.service");

module.exports = {
	ExampleSvc,
	EightBallSvc,
	WeatherSvc,
	GameTokensSvc
};
```

Update `views/index.js`:
```javascript
const ExampleView = require("./example.view");
const EightBallView = require("./eightball.view");
const WeatherView = require("./weather.view");
const GameTokensView = require("./gameTokens.view");

module.exports = {
	ExampleView,
	EightBallView,
	WeatherView,
	GameTokensView
};
```

Update `routes/index.js` to handle the new endpoints:

```javascript
// In the routing logic, add support for different HTTP methods:

if (props.method === "GET") {
	let route = REQ.getResource(2);
	
	switch (route) {
		case "api/example":
			RESP.setBody(await Controllers.ExampleCtrl.get(props));
			break;
		case "api/8ball":
			RESP.setBody(await Controllers.EightBallCtrl.get(props));
			break;
		case "api/weather":
			RESP.setBody(await Controllers.WeatherCtrl.get(props));
			break;
		case "api/tokens":
			// Handle GET /api/tokens/{token}
			RESP.setBody(await Controllers.GameTokensCtrl.get(props));
			break;
		default:
			RESP.reset({statusCode: 404});
			break;
	}
} else if (props.method === "POST") {
	let route = REQ.getResource(2);
	
	switch (route) {
		case "api/tokens":
			// Handle POST /api/tokens
			RESP.setBody(await Controllers.GameTokensCtrl.post(props));
			break;
		default:
			RESP.reset({statusCode: 404});
			break;
	}
} else if (props.method === "DELETE") {
	let route = REQ.getResource(2);
	
	switch (route) {
		case "api/tokens":
			// Handle DELETE /api/tokens/{token}
			RESP.setBody(await Controllers.GameTokensCtrl.delete(props));
			break;
		default:
			RESP.reset({statusCode: 404});
			break;
	}
} else {
	RESP.reset({statusCode: 405}); // Method not allowed
}
```

### Testing the GameTokens Service

**Create a Token (POST):**
```bash
curl -X POST "https://your-api-gateway-url/api/tokens" \
  -H "Content-Type: application/json" \
  -d '{
    "game": "tic-tac-toe",
    "players": ["Alice", "Bob"],
    "board": [["X","O","X"],["O","X","O"],["O","X","X"]],
    "winner": "Alice"
  }'
```

**Expected Response:**
```json
{
  "success": true,
  "token": "GT-a1b2c3d4-e5f6g7h8i9j0k1l2",
  "share_url": "/api/tokens/GT-a1b2c3d4-e5f6g7h8i9j0k1l2",
  "expires_at": "2024-01-15T11:30:00.000Z",
  "expires_in_minutes": 60,
  "message": "Game data stored successfully"
}
```

**Retrieve Token Data (GET):**
```bash
curl -X GET "https://your-api-gateway-url/api/tokens/GT-a1b2c3d4-e5f6g7h8i9j0k1l2"
```

**Expected Response:**
```json
{
  "success": true,
  "token": "GT-a1b2c3d4-e5f6g7h8i9j0k1l2",
  "data": {
    "game": "tic-tac-toe",
    "players": ["Alice", "Bob"],
    "board": [["X","O","X"],["O","X","O"],["O","X","X"]],
    "winner": "Alice"
  },
  "created_at": "2024-01-15T10:30:00.000Z",
  "expires_at": "2024-01-15T11:30:00.000Z",
  "message": "Game data retrieved successfully"
}
```

**Delete Token (DELETE):**
```bash
curl -X DELETE "https://your-api-gateway-url/api/tokens/GT-a1b2c3d4-e5f6g7h8i9j0k1l2"
```

### Key Features of the DynamoDB Integration

1. **Automatic Expiration**: Uses DynamoDB TTL to automatically clean up expired tokens
2. **Unique Token Generation**: Combines timestamp, data hash, and random components
3. **Comprehensive Error Handling**: Handles all DynamoDB error conditions
4. **Security**: Validates token format and checks expiration
5. **Performance Monitoring**: Tracks all database operations
6. **RESTful API**: Supports GET, POST, and DELETE operations

This DynamoDB integration demonstrates how to implement persistent storage for serverless applications, providing a foundation for features like session management, temporary data sharing, and user-generated content storage.

### Static, Sample, and Test data

The application includes several types of data files to support development, testing, and reference purposes. Understanding how to organize and use these data files is crucial for maintaining a robust application.

#### Static Data

Static data files contain reference information that doesn't change frequently, such as status codes, configuration constants, and lookup tables.

**Location**: `src/models/static-data/`

**Example: Status Codes (`models/static-data/statusCodes.js`)**

```javascript
/**
 * HTTP Status Codes and their meanings
 * Used throughout the application for consistent error handling
 */
const statusCodes = {
	200: {
		code: 200,
		message: "OK",
		description: "The request was successful"
	},
	400: {
		code: 400,
		message: "Bad Request",
		description: "The request was invalid or malformed"
	},
	401: {
		code: 401,
		message: "Unauthorized",
		description: "Authentication is required"
	},
	403: {
		code: 403,
		message: "Forbidden",
		description: "Access to the resource is forbidden"
	},
	404: {
		code: 404,
		message: "Not Found",
		description: "The requested resource was not found"
	},
	429: {
		code: 429,
		message: "Too Many Requests",
		description: "Rate limit exceeded"
	},
	500: {
		code: 500,
		message: "Internal Server Error",
		description: "An unexpected error occurred"
	},
	502: {
		code: 502,
		message: "Bad Gateway",
		description: "Invalid response from upstream server"
	},
	503: {
		code: 503,
		message: "Service Unavailable",
		description: "The service is temporarily unavailable"
	}
};

/**
 * Get status code information
 * @param {number} code - HTTP status code
 * @returns {object} - Status code information
 */
const getStatusInfo = (code) => {
	return statusCodes[code] || {
		code: code,
		message: "Unknown Status",
		description: "Unknown status code"
	};
};

/**
 * Check if status code indicates success
 * @param {number} code - HTTP status code
 * @returns {boolean} - True if success status
 */
const isSuccessStatus = (code) => {
	return code >= 200 && code < 300;
};

/**
 * Check if status code indicates client error
 * @param {number} code - HTTP status code
 * @returns {boolean} - True if client error status
 */
const isClientError = (code) => {
	return code >= 400 && code < 500;
};

/**
 * Check if status code indicates server error
 * @param {number} code - HTTP status code
 * @returns {boolean} - True if server error status
 */
const isServerError = (code) => {
	return code >= 500 && code < 600;
};

module.exports = {
	statusCodes,
	getStatusInfo,
	isSuccessStatus,
	isClientError,
	isServerError
};
```

#### Sample Data

Sample data files provide realistic test data for development and demonstration purposes.

**Location**: `src/models/sample-data/`

**Example: Sample Game Data (`models/sample-data/games.js`)**

```javascript
/**
 * Sample game data for testing and development
 */
const sampleGames = {
	gamechoices: [
		"Tic Tac Toe",
		"Chess",
		"Checkers",
		"Connect Four",
		"Battleship",
		"Scrabble",
		"Monopoly",
		"Risk",
		"Clue",
		"Trivial Pursuit"
	]
};

const sampleWeatherData = {
	coord: { lon: -0.1278, lat: 51.5074 },
	weather: [
		{
			id: 803,
			main: "Clouds",
			description: "broken clouds",
			icon: "04d"
		}
	],
	base: "stations",
	main: {
		temp: 15.32,
		feels_like: 14.85,
		temp_min: 13.89,
		temp_max: 16.67,
		pressure: 1013,
		humidity: 72
	},
	visibility: 10000,
	wind: { speed: 3.6, deg: 230 },
	clouds: { all: 75 },
	dt: 1640995200,
	sys: {
		type: 2,
		id: 2019646,
		country: "GB",
		sunrise: 1640937284,
		sunset: 1640965432
	},
	timezone: 0,
	id: 2643743,
	name: "London",
	cod: 200
};

const sample8BallResponses = [
	"It is certain",
	"Reply hazy, try again",
	"Don't count on it",
	"It is decidedly so",
	"Ask again later",
	"My reply is no",
	"Without a doubt",
	"Better not tell you now",
	"My sources say no",
	"Yes definitely",
	"Cannot predict now",
	"Outlook not so good",
	"You may rely on it",
	"Concentrate and ask again",
	"Very doubtful"
];

module.exports = {
	sampleGames,
	sampleWeatherData,
	sample8BallResponses
};
```

#### Test Data

Test data files contain structured data for unit tests and integration tests.

**Location**: `src/models/test-data/`

**Example: Test Fixtures (`models/test-data/fixtures.js`)**

```javascript
/**
 * Test fixtures for unit and integration tests
 */
const testFixtures = {
	validGameToken: "GT-a1b2c3d4-e5f6g7h8i9j0k1l2m3n4o5p6",
	expiredGameToken: "GT-expired1-2345678901234567890abcdef",
	invalidGameToken: "INVALID-TOKEN-FORMAT",
	
	validGameData: {
		game: "tic-tac-toe",
		players: ["Alice", "Bob"],
		board: [
			["X", "O", "X"],
			["O", "X", "O"],
			["O", "X", "X"]
		],
		winner: "Alice",
		moves: 9,
		duration: 180
	},
	
	validWeatherQuery: {
		city: "London,UK",
		units: "metric",
		lang: "en"
	},
	
	validCoordinateQuery: {
		lat: 51.5074,
		lon: -0.1278,
		units: "metric"
	},
	
	mockApiResponses: {
		games: {
			success: {
				statusCode: 200,
				body: JSON.stringify({
					gamechoices: ["Chess", "Checkers", "Tic Tac Toe"]
				})
			},
			error: {
				statusCode: 500,
				body: "Internal Server Error"
			}
		},
		weather: {
			success: {
				statusCode: 200,
				body: JSON.stringify({
					coord: { lon: -0.1278, lat: 51.5074 },
					weather: [{ main: "Clear", description: "clear sky" }],
					main: { temp: 20, humidity: 65, pressure: 1013 },
					name: "London",
					cod: 200
				})
			},
			notFound: {
				statusCode: 404,
				body: JSON.stringify({
					cod: "404",
					message: "city not found"
				})
			}
		}
	}
};

module.exports = testFixtures;
```

### Create a Controller and View utilizing the four data services

Now let's create a comprehensive controller that demonstrates how to coordinate multiple services and present unified responses. This controller will showcase integration patterns for combining different types of data services.

#### Create the Dashboard Controller

Create a new file `controllers/dashboard.controller.js`:

```javascript
const { tools: {Timer, DebugAndLog} } = require("@63klabs/cache-data");

const { ExampleSvc, EightBallSvc, WeatherSvc, GameTokensSvc } = require("../services");
const { DashboardView } = require("../views");

const logIdentifier = "Dashboard Controller";

/**
 * Get comprehensive dashboard data combining multiple services
 * @param {object} props - Request properties from router
 * @returns {object} data - Combined dashboard data
 */
exports.get = async (props) => {
	let data = null;
	const timer = new Timer(`${logIdentifier} GET`, true);
	DebugAndLog.debug(`${logIdentifier} GET: Properties received`, props);

	return new Promise(async (resolve, reject) => {
		try {
			// Extract parameters
			const city = props?.queryParameters?.city || "London,UK";
			const question = props?.queryParameters?.question || "Will today be a good day?";
			const includeGames = props?.queryParameters?.games !== 'false';
			
			// Prepare queries for each service
			const baseQuery = {
				calcMsToDeadline: props?.calcMsToDeadline,
				deadline: props?.deadline
			};

			// Collect promises for parallel execution
			const servicePromises = [];
			const serviceNames = [];

			// Add weather service
			servicePromises.push(WeatherSvc.fetch({
				...baseQuery,
				city: city,
				units: "metric"
			}));
			serviceNames.push('weather');

			// Add 8 ball service
			servicePromises.push(EightBallSvc.fetch({
				...baseQuery,
				question: question
			}));
			serviceNames.push('eightball');

			// Conditionally add games service
			if (includeGames) {
				servicePromises.push(ExampleSvc.fetch({
					...baseQuery,
					organizationCode: "demo"
				}));
				serviceNames.push('games');
			}

			// Execute all services in parallel
			DebugAndLog.debug(`${logIdentifier} GET: Executing ${servicePromises.length} services in parallel`);
			const results = await Promise.allSettled(servicePromises);

			// Process results
			const serviceResults = {};
			results.forEach((result, index) => {
				const serviceName = serviceNames[index];
				if (result.status === 'fulfilled') {
					serviceResults[serviceName] = {
						success: true,
						data: result.value
					};
				} else {
					serviceResults[serviceName] = {
						success: false,
						error: result.reason?.message || 'Service failed'
					};
					DebugAndLog.error(`${logIdentifier} GET: ${serviceName} service failed`, result.reason);
				}
			});

			// Format through dashboard view
			data = DashboardView.view({
				services: serviceResults,
				query: {
					city: city,
					question: question,
					includeGames: includeGames
				}
			});

			DebugAndLog.debug(`${logIdentifier} GET: Dashboard data compiled`, {
				servicesExecuted: serviceNames.length,
				successfulServices: Object.values(serviceResults).filter(r => r.success).length
			});

		} catch (error) {
			DebugAndLog.error(`${logIdentifier} GET: Error: ${error.message}`, error.stack);
			data = DashboardView.viewError('Dashboard compilation failed');
		} finally {
			timer.stop();
			resolve(data);
		}
	});
};

/**
 * Create a shareable dashboard snapshot
 * @param {object} props - Request properties from router
 * @returns {object} data - Token for shareable dashboard
 */
exports.post = async (props) => {
	let data = null;
	const timer = new Timer(`${logIdentifier} POST`, true);
	DebugAndLog.debug(`${logIdentifier} POST: Properties received`, props);

	return new Promise(async (resolve, reject) => {
		try {
			// Get dashboard data first
			const dashboardData = await exports.get(props);
			
			if (!dashboardData || !dashboardData.success) {
				data = DashboardView.viewError('Cannot create snapshot of failed dashboard');
				resolve(data);
				return;
			}

			// Create snapshot data
			const snapshotData = {
				type: 'dashboard_snapshot',
				created_at: new Date().toISOString(),
				dashboard: dashboardData,
				query: {
					city: props?.queryParameters?.city || "London,UK",
					question: props?.queryParameters?.question || "Will today be a good day?",
					includeGames: props?.queryParameters?.games !== 'false'
				}
			};

			// Store snapshot using GameTokens service
			const tokenResult = await GameTokensSvc.put(snapshotData);
			
			data = DashboardView.viewSnapshot(tokenResult, snapshotData);

			DebugAndLog.debug(`${logIdentifier} POST: Dashboard snapshot created`, {
				success: tokenResult.success,
				token: tokenResult.token
			});

		} catch (error) {
			DebugAndLog.error(`${logIdentifier} POST: Error: ${error.message}`, error.stack);
			data = DashboardView.viewError('Failed to create dashboard snapshot');
		} finally {
			timer.stop();
			resolve(data);
		}
	});
};
```

#### Create the Dashboard View

Create a new file `views/dashboard.view.js`:

```javascript
const { 
	tools: {
		Timer
	}
} = require("@63klabs/cache-data");

const utils = require('../utils');

const logIdentifier = "Dashboard View";

/**
 * Format comprehensive dashboard response
 * @param {object} resultsFromController - Combined service results
 * @returns {object} - Formatted dashboard response
 */
exports.view = (resultsFromController) => {
	const viewTimer = new Timer(`Timer: ${logIdentifier}`, true);

	let finalView = {};

	try {
		const services = resultsFromController?.services || {};
		const query = resultsFromController?.query || {};

		// Generate dashboard ID
		const dashboardId = utils.hash.takeFirst(
			`dashboard-${JSON.stringify(query)}-${Date.now()}`,
			12
		);

		// Process weather data
		let weather = null;
		if (services.weather?.success && services.weather.data) {
			const weatherData = services.weather.data;
			weather = {
				location: weatherData.name || query.city,
				temperature: weatherData.main ? Math.round(weatherData.main.temp) : null,
				description: weatherData.weather?.[0]?.description || "Unknown",
				humidity: weatherData.main?.humidity || null,
				conditions: weatherData.weather?.[0]?.main || "Unknown"
			};
		}

		// Process 8 ball data
		let prediction = null;
		if (services.eightball?.success && services.eightball.data) {
			const eightballData = services.eightball.data;
			prediction = {
				question: query.question,
				answer: eightballData.magic?.answer || "Ask again later",
				type: eightballData.magic?.type || "neutral"
			};
		}

		// Process games data
		let games = null;
		if (services.games?.success && services.games.data) {
			const gamesData = services.games.data;
			games = {
				available: Array.isArray(gamesData.gamechoices) ? gamesData.gamechoices.length : 0,
				list: Array.isArray(gamesData.gamechoices) ? gamesData.gamechoices.slice(0, 5) : []
			};
		}

		// Count successful services
		const serviceStatus = {
			total: Object.keys(services).length,
			successful: Object.values(services).filter(s => s.success).length,
			failed: Object.values(services).filter(s => !s.success).length
		};

		// Build final response
		finalView = {
			success: serviceStatus.successful > 0,
			id: `DB-${dashboardId}`,
			timestamp: new Date().toISOString(),
			query: query,
			services: serviceStatus,
			data: {
				weather: weather,
				prediction: prediction,
				games: games
			},
			errors: Object.entries(services)
				.filter(([name, result]) => !result.success)
				.map(([name, result]) => ({
					service: name,
					error: result.error
				}))
		};

	} catch (error) {
		finalView = {
			success: false,
			error: "Dashboard view processing failed",
			timestamp: new Date().toISOString()
		};
	}

	viewTimer.stop();
	return finalView;
};

/**
 * Format dashboard snapshot response
 * @param {object} tokenResult - Token creation result
 * @param {object} snapshotData - Original snapshot data
 * @returns {object} - Formatted snapshot response
 */
exports.viewSnapshot = (tokenResult, snapshotData) => {
	const viewTimer = new Timer(`Timer: ${logIdentifier} Snapshot`, true);

	let finalView = {};

	try {
		if (tokenResult?.success) {
			finalView = {
				success: true,
				snapshot: {
					token: tokenResult.token,
					share_url: `/api/tokens/${tokenResult.token}`,
					expires_at: tokenResult.expires_at,
					created_at: snapshotData.created_at
				},
				dashboard_summary: {
					services_included: Object.keys(snapshotData.dashboard?.data || {}).length,
					query: snapshotData.query
				},
				message: "Dashboard snapshot created successfully"
			};
		} else {
			finalView = {
				success: false,
				error: tokenResult?.error || "Failed to create snapshot",
				message: "Could not create dashboard snapshot"
			};
		}
	} catch (error) {
		finalView = {
			success: false,
			error: "Snapshot view processing failed",
			message: "Dashboard snapshot view error"
		};
	}

	viewTimer.stop();
	return finalView;
};

/**
 * Format error response
 * @param {string} errorMessage - Error message
 * @returns {object} - Formatted error response
 */
exports.viewError = (errorMessage) => {
	return {
		success: false,
		error: errorMessage,
		timestamp: new Date().toISOString(),
		message: "Dashboard request could not be processed"
	};
};
```

#### Update Index Files and Routing

Update `controllers/index.js`:
```javascript
const ExampleCtrl = require("./example.controller");
const EightBallCtrl = require("./eightball.controller");
const WeatherCtrl = require("./weather.controller");
const GameTokensCtrl = require("./gameTokens.controller");
const DashboardCtrl = require("./dashboard.controller");

module.exports = {
	ExampleCtrl,
	EightBallCtrl,
	WeatherCtrl,
	GameTokensCtrl,
	DashboardCtrl
};
```

Update `views/index.js`:
```javascript
const ExampleView = require("./example.view");
const EightBallView = require("./eightball.view");
const WeatherView = require("./weather.view");
const GameTokensView = require("./gameTokens.view");
const DashboardView = require("./dashboard.view");

module.exports = {
	ExampleView,
	EightBallView,
	WeatherView,
	GameTokensView,
	DashboardView
};
```

Add dashboard routes to `routes/index.js`:

```javascript
// In the GET method switch statement:
case "api/dashboard":
	RESP.setBody(await Controllers.DashboardCtrl.get(props));
	break;

// In the POST method switch statement:
case "api/dashboard":
	RESP.setBody(await Controllers.DashboardCtrl.post(props));
	break;
```

#### Testing the Dashboard

**Get Dashboard Data:**
```bash
curl -X GET "https://your-api-gateway-url/api/dashboard?city=Paris,FR&question=Should%20I%20travel%20today&games=true"
```

**Create Dashboard Snapshot:**
```bash
curl -X POST "https://your-api-gateway-url/api/dashboard?city=Tokyo,JP&question=Will%20it%20be%20sunny"
```

**Expected Dashboard Response:**
```json
{
  "success": true,
  "id": "DB-a1b2c3d4e5f6",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "query": {
    "city": "Paris,FR",
    "question": "Should I travel today",
    "includeGames": true
  },
  "services": {
    "total": 3,
    "successful": 3,
    "failed": 0
  },
  "data": {
    "weather": {
      "location": "Paris",
      "temperature": 18,
      "description": "clear sky",
      "humidity": 65,
      "conditions": "Clear"
    },
    "prediction": {
      "question": "Should I travel today",
      "answer": "Signs point to yes",
      "type": "positive"
    },
    "games": {
      "available": 10,
      "list": ["Chess", "Checkers", "Tic Tac Toe", "Connect Four", "Battleship"]
    }
  },
  "errors": []
}
```

This dashboard controller demonstrates advanced patterns for coordinating multiple services, handling parallel execution, and providing comprehensive error handling while maintaining performance through proper use of Promise.allSettled().

## Part III Summary

TODO

[Move on to Part IV](./part-04.md)