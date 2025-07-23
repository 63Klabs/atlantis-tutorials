
# Part II: Application Starter 02 API Gateway with Lambda using Cache-Data (Node.js)

> Uses [Atlantis App Starter - 02 - API Gateway and Lambda using @63Klabs/Cache-Data (Node)](https://github.com/63Klabs/atlantis-starter-02-apigw-lambda-cache-data-nodejs)

Refer to the README in the app starter GitHub repository above for an overview of the code.

## 1. Seed repository and create pipeline

Using the `create_repo` script in your organization's SAM Config repository, create and seed the repository with application starter 02.

```bash
./cli/create_repo.py tutorial-games-proxy --profile ACME_DEV_PROFILE
```

Choose application starter 02 (`atlantis-starter-02-apigw-lambda-cache-data-nodejs`) when prompted.

Clone the application's repository to your local environment and merge the `dev` branch into the `test` branch without making any changes.

> When staring new projects it is a good idea to start off with known, working code and get the initial "Hello" deployment working before making changes.

In the SAM Config repository, create the pipeline for your application.

```bash
./cli/config.py pipeline acme games-proxy test --profile ACME_DEV_PROFILE
```

Copy, paste and execute the deploy command from the config output.

```bash
# Perform this command in the SAM Config Repo
./cli/deploy.py pipeline acme games-proxy test --profile ACME_DEV_PROFILE
```

After the pipeline has been created successfully, a link to the pipeline will be displayed in the Output. Follow the link to view the pipeline in the console. (You may need to log into the console first before following the link.)

Once your CloudFormation application stack has deployed, view the endpoint in your browser. You should see a list of games.

## 2. Inspect Parameters and Environment Variables

Just like passing parameters or arguments into a method or function in your code, you can pass parameters and variables between AWS resources.

For example, your SAM configuration passed Parameters to your stack to use as it deployed resources for the Pipeline. You entered these in the `config.py` prompt and they were stored in the SAM config file like so:

```toml
parameter_overrides = "\"Prefix\"=\"acme\" \"ProjectId\"=\"starter-02\" \"StageId\"=\"test\" \"S3BucketNameOrgPrefix\"=\"\" \"RolePath\"=\"/sam-apps/\" \"PermissionsBoundaryArn\"=\"\" \"DeployEnvironment\"=\"TEST\" \"S3ArtifactsBucket\"=\"cf-templates-aaaaaa123-us-east-2\" \"S3StaticHostBucket\"=\"\" \"BuildSpec\"=\"application-infrastructure/buildspec.yml\" \"ParameterStoreHierarchy\"=\"/sam-apps/\" \"AlarmNotificationEmail\"=\"chad@63klabs.net\" \"Repository\"=\"acme-02-starter\" \"RepositoryBranch\"=\"test\""
```

When the stack was deployed it used these parameters to make decisions, name resources, and even pass on to the CodeBuild environment and the Application stack.

```yaml
# Relevant code from template-pipeline.yml
# Full template: https://github.com/63Klabs/atlantis-cfn-template-repo-for-serverless-deployments/blob/main/templates/v2/pipeline/template-pipeline.yml

Resources:

  # ....

  CodeBuildProject:
    Type: AWS::CodeBuild::Project
	Properties:
	  Environment:
	  	# These environment variables are available for use by bash commands in your buildspec
        EnvironmentVariables:
        - Name: AWS_PARTITION
          Value: !Ref 'AWS::Partition'
        - Name: AWS_REGION
          Value: !Ref 'AWS::Region'
		  # ...
		- Name: ROLE_PATH
          Value: !Ref 'RolePath'
        - Name: PERMISSIONS_BOUNDARY_ARN
          Value: !Ref 'PermissionsBoundaryArn'
        - Name: NODE_ENV
          Value: "production"

  # ...
  ProjectPipeline:
    Type: AWS::CodePipeline::Pipeline
	Properties:
	  Stages:
	  # ...
      -
        Name: Deploy
        Actions:
        -
		Name: GenerateChangeSet
		Configuration:
			# These are passed to your application stack
			ParameterOverrides: !Sub '{
				"Prefix": "${Prefix}",
				"ProjectId": "${ProjectId}",
				"StageId": "${StageId}",
				"S3BucketNameOrgPrefix": "${S3BucketNameOrgPrefix}",
				"RolePath": "${RolePath}",
				"DeployEnvironment": "${DeployEnvironment}",
				"ParameterStoreHierarchy": "${ParameterStoreHierarchy}${DeployEnvironment}/${Prefix}-${ProjectId}-${StageId}/",
				"DeployRole": "${CodeDeployServiceRole.Arn}",
				"AlarmNotificationEmail": "${AlarmNotificationEmail}",
				"PermissionsBoundaryArn": "${PermissionsBoundaryArn}"
				}'
```

> You can override these values, and add additional parameter values for your Application stack by modifying the `template-configuration.yml` file in your application infrastructure directory.

Your application stack also passes Environment variables to your Lambda function:

```yaml
# application-infrastructure/template.yml

Resources:
  AppFunction:
    Type: AWS::Serverless::Function
    Properties:
	  Environment:
      Variables:
        NODE_ENV: "production"
        DEPLOY_ENVIRONMENT: !Ref DeployEnvironment
        LOG_LEVEL: !If [ IsProduction, "0", "5"]
        PARAM_STORE_PATH: !Ref ParameterStoreHierarchy 
		CACHE_DATA_TIME_ZONE_FOR_INTERVAL: !Ref CacheDataTimeZoneForInterval 
        CACHE_DATA_AWS_X_RAY_ON: true
        CACHE_DATA_USE_TOOLS_HASH_METHOD: true
```

Which you can then use as such:

```js
// Node
const loglevel = process.env.LOG_LEVEL;
```

```py
# Python
import os
my_var = os.environ.get('LOG_LEVEL')
```

Understanding how these values are passed from SAM config, to the Pipeline stack, to CodeBuild, to the Application Stack, and to your Lambda function is essential for troubleshooting and getting the most out of existing values.

Here are the AWS CLI commands to list environment variables and parameters for various resources:

### CodeBuild Project Environment Variables

```bash
aws codebuild batch-get-projects --names CODEBUILD_PROJECT_NAME | jq '.projects[0].environment.environmentVariables' --profile ACME_DEV_PROFILE
```
Replace `CODEBUILD_PROJECT_NAME` with the CodeBuild resource name. (You can get this from the Resources section of your Pipeline stack)

### CloudFormation Stack Parameters

```bash
aws cloudformation describe-stacks --stack-name STACK_NAME | jq '.Stacks[0].Parameters' --profile ACME_DEV_PROFILE
```

### Lambda Function Environment Variables

```bash
aws lambda get-function-configuration --function-name FUNCTION_NAME | jq '.Environment.Variables' --profile ACME_DEV_PROFILE
```

## 3. Check endpoint and cache

Hit refresh in the browser a few times. You should notice that while the first request took a second or two, the following requests are much shorter. This is because upon your first request the cache had to load from the original source. Subsequent requests only need to get the data from DynamoDb.

In the web console go to the Lambda Execution logs in CloudWatch. (There is a link in your application stacks outputs section.)

You'll notice at the very start of the logs is a `COLDSTART` indicator. You'll also notice log information about what actions your application took to fulfill the request to your browser. Some logs will include cache access information. You'll notice on the first request there wasn't a cache so it had to get it from the original source.

You'll also notice a final response log that gives information about the request and the response.

Next, we'll check out the data in the DynamoDb table. 

> Note: You may not have access to the table to view the data. If you do not have access to actually see that the cache exists, please just take my word, and your CloudWatch logs entry, as proof that the data exists.

You can get the link to the DynamoDb table either from the Cache-Data storage stack output section or by heading to DynamoDb in the web console and searching for `<prefix>-cache-data-CacheData`

When you access the table you can examine the table info and click on Explore table items. 

When you explore table items you'll see the cache records. If you click into the records you can examine the meta data, and that the data itself is encrypted.

## 3. Understand components of application template

Let's step back a moment and explore the application resources by examining the template.

## Metadata

TODO

## Parameters and overrides

TODO

## Utilize conditionals for resource creation and properties

TODO

## Mapping

TODO

## Using `ImportValue` instead of parameters

TODO
