# Part III: Application Starter 02 API Gateway with Lambda using Cache-Data (Node.js)

## 1, Implement a basic service with a direct call to an endpoint with no caching (8 Ball)

Now let's implement a simple service that makes direct API calls without caching. This demonstrates how to handle services that need real-time data or where caching isn't appropriate.

The 8 Ball service will provide random answers to questions, similar to a Magic 8 Ball toy. Since we want fresh, random responses each time, we won't use caching.

### 1.1: Create the 8 Ball Controller

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

	return new Promise(async (resolve) => {
		try {
			
			DebugAndLog.debug(`${logIdentifier}: Called`);

			// Call service and format through view
			data = EightBallView.view(await EightBallSvc.fetch(query));

			DebugAndLog.debug(`${logIdentifier}: 8 Ball response`, data);

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

### 1.2: Create the 8 Ball Service

Create a new file `services/eightball.service.js`:

```javascript
const {
	tools: {
		DebugAndLog,
		Timer
	},
	endpoint // use endpoint.get() to directly call an endpoint
} = require("@63klabs/cache-data");

const logIdentifier = "EightBall Service GET";

/**
* Fetch 8 Ball prediction without caching
* @param {object} query - Query parameters from controller
* @returns {object} results - Raw API response
*/
exports.fetch = async (query) => {
	return new Promise(async (resolve, reject) => {
		let resultBody = {};
		const timer = new Timer(logIdentifier, true);
		DebugAndLog.debug(`${logIdentifier}: Query Received`, query);

		try {

			// Make direct API call without caching
			results = await endpoint.get({url: "https://api.chadkluck.net/8ball"});

			resultBody = results.body || {};

		} catch (error) {
			DebugAndLog.error(`${logIdentifier}: Error: ${error.message}`, error.stack);
			resultBody = { error: "Service unavailable" };
		} finally {
			timer.stop();
			resolve(resultBody);
		}
	});
};
```

### 1.3: Create the 8 Ball View

Create a new file `views/eightball.view.js`:

```javascript
const {
  tools: {
    Timer
  }
} = require("@63klabs/cache-data");

const logIdentifier = "EightBall View";

/**
* Transform 8 Ball API response into consistent format
* @param {object} resultsFromSvc - Raw service response
* @returns {object} - Formatted 8 Ball response
*/
exports.view = (resultsFromSvc) => {
  const viewTimer = new Timer(`Timer: ${logIdentifier}`, true);

  let finalView = null;

  try {
    finalView = resultsFromSvc;
  } catch (error) {
    // Handle view processing errors
    finalView = {
      success: false,
      error: "View processing failed",
	  prediction: "The magic 8 ball is broken"
    };
  } finally {
    viewTimer.stop();
    return finalView;
  };

};
```

### 1.4: Add Routing Configuration

Update the routing in `routes/index.js` to include the 8 Ball endpoint. Add this case to the switch statement:

```javascript
// In the GET method switch statement, add:
case "api/8ball":
	RESP.setBody(await Controllers.EightBallCtrl.get(props));
	break;
```

### 1.5: Update Controller Index

Update `controllers/index.js` to export the new controller:

```javascript
const ExampleCtrl = require("./example.controller");
const EightBallCtrl = require("./eightball.controller");

module.exports = {
	ExampleCtrl,
	EightBallCtrl
};
```

### 1.6: Update Service Index

Update `services/index.js` to export the new service:

```javascript
const ExampleSvc = require("./example.service");
const EightBallSvc = require("./eightball.service");

module.exports = {
	ExampleSvc,
	EightBallSvc
};
```

### 1.7: Update View Index

Update `views/index.js` to export the new view:

```javascript
const ExampleView = require("./example.view");
const EightBallView = require("./eightball.view");

module.exports = {
	ExampleView,
	EightBallView
};
```

### 1.8: Update Template

```yaml
GetEightBallData:
		  Type: Api
		  Properties:
			Path: /api/8ball
			Method: get
			RestApiId: !Ref WebApi
 
```

### 1.9: Update Template Open API Spec

```yaml
/api/8ball/:
	get:
	  description: "GET 8Ball API"
	  responses:
		$ref: '#/components/schemas/DataResponses'
	  x-amazon-apigateway-integration:
		httpMethod: post
		type: aws_proxy
		uri:
		  Fn::Sub: arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${AppFunction.Arn}/invocations
```

### 1.10: Deploy

Merge `dev` into `test` and push.

Watch the pipeline through the console and fix any errors.

Once deployed check out the new `8ball` endpoint.

### 1.11: Update `eightball.service` to use Connections

When we created the service, just to keep it simple, we passed a hard coded URL to the `endpoint.get()` method:

```javascript
results = await endpoint.get({url: "https://api.chadkluck.net/8ball"});
```

However, we can use the existing connection configuration system to manage the endpoint details. This allows us to centrally store all connection configurations in one place, making it easier to review and understand the endpoints our application is connecting too.

The connections object contains fields similar to a fetch request, breaking down the URL into `host`, `path`, and other HTTP request properties.

Update `config/connections.js` to add the 8 Ball connection after the `games` connection:

```js
{
	// ... games
},
{
	name: "8ball",
	host: "api.chadkluck.net",
	path: "/8ball",
}
```

Unlike the `games` connection, we won't add any cache properties. We will also keep it simple and not add query string parameters, headers, or method.

Next, we will update `services/eightball.service.js`:

```javascript
// ADD the following require near the top of the script
const { Config } = require("../config"); // configured connections
```

Further down, in the try block, update:

```javascript
	// Make direct API call without caching
	results = await endpoint.get({url: "https://api.chadkluck.net/8ball"});

	resultBody = results.body || {};
```

To:

```javascript
	// Make direct API call without caching
	let conn = Config.getConn("8ball"); // by name defined in connections.js
	results = await endpoint.get(conn);
	resultBody = results.body || {};
```

### 1.12: Deploy Again

After making these changes, redeploy the application to apply the updates.

```bash
git add --all
git commit -m "changed 8Ball service to use config/connections"
git push
git switch test
git merge dev
git push
git switch dev
```

Then test using `curl`:

```bash
# Basic request
curl -X GET "https://your-api-gateway-url/api/8ball"
```

## 2. Implement a service using Cache Data Access Object (Games)

TODO

## 3. Implement a Route, Controller, and View that brings two services together

Now let's create a comprehensive controller that demonstrates how to coordinate multiple services and present unified responses. This controller will showcase integration patterns for combining different types of data services.

### 3.1 Create the Dashboard Controller

Create a new file `controllers/suggest.controller.js`:

```javascript
const { tools: {Timer, DebugAndLog} } = require("@63klabs/cache-data");

const { ExampleSvc, EightBallSvc } = require("../services");
const { SuggestView } = require("../views");

const logIdentifier = "Suggest Controller";

/**
 * Get comprehensive data combining multiple services
 * @param {object} props - Request properties from router
 * @returns {object} data - Combined dashboard data
 */
exports.get = async (props) => {
	let data = null;
	const timer = new Timer(`${logIdentifier} GET`, true);
	DebugAndLog.debug(`${logIdentifier} GET: Properties received`, props);

	return new Promise(async (resolve, reject) => {
		try {

			// Collect promises for parallel execution
			const servicePromises = [];
			const serviceNames = [];

			// Add 8 ball service
			servicePromises.push(EightBallSvc.fetch());
			serviceNames.push('eightball');

			// Conditionally add games service
			servicePromises.push(ExampleSvc.fetch());
			serviceNames.push('games');

			// Execute all services in parallel
			DebugAndLog.debug(`${logIdentifier} GET: Executing ${servicePromises.length} services in parallel`);
			const results = await Promise.all(servicePromises);

			// Process results
			const serviceResults = {};
			results.forEach((result, index) => {
				const serviceName = serviceNames[index];
				serviceResults[serviceName] = results[index];
			});

			// Format through dashboard view
			data = Suggest.view(serviceResults);

			DebugAndLog.debug(`${logIdentifier} GET: Service data compiled`, {
				servicesExecuted: serviceNames.length
			});

		} catch (error) {
			DebugAndLog.error(`${logIdentifier} GET: Error: ${error.message}`, error.stack);
			data = DashboardView.viewError('Suggest compilation failed');
		} finally {
			timer.stop();
			resolve(data);
		}
	});
};
```

### 3.2 Create the Dashboard View

Create a new file `views/suggest.view.js`:

```javascript
const { 
	tools: {
		Timer
	}
} = require("@63klabs/cache-data");

const logIdentifier = "Suggest View";

/**
 * Format comprehensive dashboard response
 * @param {object} resultsFromService - Combined service results
 * @returns {object} - Formatted dashboard response
 */
exports.view = (resultsFromService) => {
	const viewTimer = new Timer(`Timer: ${logIdentifier}`, true);

	let finalView = {};

	try {
		const services = resultsFromService || {};

		// Process 8 ball data
		let prediction = null;
		if (services?.eightball?.success && services.eightball.data) {
			const eightballData = services.eightball.data;
			prediction = eightballData.magic?.answer || "Ask again later";
		}

		// Process games data
		let game = null;
		if (services?.games?.success && services.games.data) {
			const gamesData = services.games.data;
			const len = gamesData.gamechoices.length : 0;
			// get random game from gamechoices
			game = gamesData.gamechoices[Math.floor(Math.random() * len)];
		}

		// Build final response
		finalView = {game, prediction};

	} catch (error) {
		finalView = {
			error: "Suggest processing failed",
		};
	}

	viewTimer.stop();
	return finalView;
};
```

### 3.3 Update Index Files and Routing

Update `controllers/index.js`:
```javascript
const ExampleCtrl = require("./example.controller");
const EightBallCtrl = require("./eightball.controller");
const SuggestCtrl = require("./suggest.controller");

module.exports = {
	ExampleCtrl,
	EightBallCtrl,
    SuggestCtrl
};
```

Update `views/index.js`:
```javascript
const ExampleView = require("./example.view");
const EightBallView = require("./eightball.view");
const SuggestView = require("./suggest.view");

module.exports = {
	ExampleView,
	EightBallView,
	SuggestView
};
```

Add `suggest` route to `routes/index.js`:

```javascript
// In the GET method switch statement:
case "api/suggest":
	RESP.setBody(await Controllers.DashboardCtrl.get(props));
	break;
```

### 3.4 Add to template

TODO

### 3.5 Add to Open API Spec

TODO

### 3.6 Test

```bash
curl -X GET "https://your-api-gateway-url/api/suggest"
```

This "suggest" controller demonstrates advanced patterns for coordinating multiple services, handling parallel execution, and providing comprehensive error handling while maintaining performance through proper use of Promise.all().

## Part III Summary

TODO

[Move on to Part IV](./part-04.md)