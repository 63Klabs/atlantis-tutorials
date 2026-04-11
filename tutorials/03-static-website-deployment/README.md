# Deploying a Static Website

> This tutorial is still under development. However, the basic structure is listed below. Please be advised that the content is short, may be missing, and may have inaccuracies or typos. If you would like to contribute updates, please submit an [issue via this repository on GitHub](https://github.com/63Klabs/atlantis-tutorials/issues). Be sure to include the page and what content should be added/updated. If you'd be willing to write a few sentences (or more, but be clear and succinct), please do. Thank you for your understanding.

[AWS Amplify](https://aws.amazon.com/amplify) is a powerful, managed solution for deploying websites. It is recommended for those who need a dead simple method of deploying a static website or web application as it handles the deployment, certificates, custom domains, hosting, and more for you.

**However**, if you want to get more hands-on with the development, configuration, and deployment of a static website using S3, a Pipeline, and CloudFront paired with Route53, use this tutorial to explore the solution.

## 1. reate Your Repository
From the SAM Config repository:

```bash
./cli/create_repo.py your-repo-name --profile YOUR_PROFILE
# Choose 'None' for seeding
```

Clone your repository, create a dev branch, and then scaffold it with React (or your framework of choice):

```bash
npm create vite@latest my-website -- --template react
cd my-website
npm install
npm run dev
```

View your site on the local machine by going to the provided `localhost`/`127.0.0.1` URL.

In the `my-website` directory, create a buildspec file with commands that will build and then copy the contents of the build to S3.

```yaml
version: 0.2
# my-website/buildspec.yml

phases:
  install:
    runtime-versions:
      nodejs: latest
      python: latest
    commands:

      - ls -l -a

      # Build Environment information for debugging purposes
      - python3 --version
      - node --version
      - aws --version
      - echo "NODE_ENV is $NODE_ENV"

      # Set npm caching (This 'offline' cache is still tar zipped, but it helps.) - https://blog.mechanicalrock.io/2019/02/03/monorepos-aws-codebuild.html
      - npm config -g set prefer-offline true
      - npm config -g set cache /root/.npm
      - npm config get cache

  pre_build:
    commands:
      
      # Application Environment: Install NPM dependencies needed for application environment
      - ls -l -a
      - cd my-website
      - npm install --production

      # FAIL the build if npm audit has vulnerabilities it can't fix
      # Perform a fix to move us forward, then check to make sure there were no unresolved high fixes
      - npm audit fix --force
      - npm audit --audit-level=high
      
  build:
    commands:

	  - npm run build

	  # S3_STATIC_HOST_BUCKET and STAGE_ID are environment variables in our CodeBuild deployment
	  - aws s3 sync dist "s3://${S3_STATIC_HOST_BUCKET}/${STAGE_ID}/public/" --delete

      # list files in the build
	  - cd dist # sometimes build/
      - ls -l -a

# add cache
cache:
  paths:
    - '/root/.npm/**/*'
```

Commit and push your code to the `dev` branch. Then merge and push into `test`.

## 2. Configure and Deploy an S3 Bucket for Hosting

The S3 Bucket will be shared among all your deployments for your project (`test`, `beta`, `prod`) so you will not need to specify a `StageId`.

From the SAM Config repository:

```bash
./cli/config.py storage acme my-website --profile YOUR_PROFILE
./cli/deploy.py storage acme my-website --profile YOUR_PROFILE
# Make note of the S3 bucket name AND S3 bucket domain
```

If you check the contents of your S3 bucket you'll see that it is empty. However, as you add and run deployment pipelines, the following structure will emerge:

```
|- test/public
|- beta/public
|- prod/public
```

This S3 bucket will be shared among all the `StageId`s of your site and each will be mapped to its own CloudFront distribution which we will create after the pipeline.

Organizations used to serve objects directly from S3 using a Static Web Hosting option. However, that is no longer the best practice.

Today, modern best practices are to create an S3 bucket, keep it out of public view, and place CloudFront in front of it. The storage template you deployed locks access to the bucket and only allows public access through CloudFront. This is called "Object Access Control" or OAC.

View the [template-storage-s3-oac-for-cloudfront.yml](https://github.com/63Klabs/atlantis-sam-templates/blob/main/templates/v2/storage/template-storage-s3-oac-for-cloudfront.yml) on GitHub to see how the bucket is defined.

## 3. Configure and Deploy your `test` Pipeline to Build and Copy Your Content to S3

Now that we have S3 set up, we can set up the CI/CD pipeline to build and copy the files over.

From the SAM Config repository:

```bash
./cli/config.py pipeline acme my-website test --profile YOUR_PROFILE
# - Use CodeBuild Only Pipeline
# - For Static Host bucket use the S3 bucket name
./cli/deploy.py pipeline acme my-website test --profile YOUR_PROFILE
```

After the pipeline has deployed, use the Output link to see it in action. Check for any errors and follow along in the CodeBuild console as it builds and copies your site to S3.

Once the deployment has finished, go to the S3 console and view the contents of your S3 bucket. You should see `test/public/<your-files>`

## 4. Configure and Deploy the `test` CloudFront Distribution (Route53 optional)

We will now create the CloudFront Distribution that uses your bucket with the `test/public` object prefix as an origin.

From the SAM Config repository:

```bash
./cli/config.py network acme my-website test --profile YOUR_PROFILE
# - There are a lot of parameters, for most you will accept the defaults
# - Use S3 Bucket Origin Domain from storage output
# - A custom domain for Route53 is optional, you can just use the provided CloudFront domain for the tutorial and development
./cli/deploy.py network acme my-website test --profile YOUR_PROFILE
```

From the output section you should see the CloudFront distribution domain. Follow the link and you should see your site.

## 5. Add `beta` and `prod`

Go through the same steps above, this time creating a `beta` and `prod` deployment.

You may need to add a `beta` branch to your repository.

## 6. Architecture Overview

Below is the final architecture diagram of what we build in this tutorial.

```mermaid
graph TB
    subgraph repo["Git Repository"]
        direction LR
        dev["dev branch"]
        test["test branch"]
        beta["beta branch"]
        main["main branch"]
        dev -->|merge| test
        test -->|merge| beta
        beta -->|merge| main
    end

    subgraph pipelines["CodePipeline + CodeBuild<br/><i>Per Stage</i>"]
        direction LR
        pipe_test["acme-my-website-test-Pipeline<br/>CodeBuild Only"]
        pipe_beta["acme-my-website-beta-Pipeline<br/>CodeBuild Only"]
        pipe_prod["acme-my-website-prod-Pipeline<br/>CodeBuild Only"]
    end

    test -->|triggers| pipe_test
    beta -->|triggers| pipe_beta
    main -->|triggers| pipe_prod

    subgraph s3["S3 Origin Bucket <i>Shared across stages — No public access</i> acme-my-website-origin-{AccountId}-{Region}-an"]
        direction LR
        s3_test["test/public/"]
        s3_beta["beta/public/"]
        s3_prod["prod/public/"]
    end

    pipe_test -->|"aws s3 sync dist<br/>→ s3://{bucket}/test/public/"| s3_test
    pipe_beta -->|"aws s3 sync dist<br/>→ s3://{bucket}/beta/public/"| s3_beta
    pipe_prod -->|"aws s3 sync dist<br/>→ s3://{bucket}/prod/public/"| s3_prod

    subgraph cdn["CloudFront Distributions <i>Per Stage — OAC Access to S3</i>"]
        direction LR
        cf_test["acme-my-website-test<br/>CloudFront Distribution<br/><i>CachingDisabled</i>"]
        cf_beta["acme-my-website-beta<br/>CloudFront Distribution"]
        cf_prod["acme-my-website-prod<br/>CloudFront Distribution"]
    end

    s3_test -.->|OAC| cf_test
    s3_beta -.->|OAC| cf_beta
    s3_prod -.->|OAC| cf_prod

    subgraph dns["Route 53 / Custom Domains <i>Optional</i>"]
        direction LR
        dns_test["my-website-test.example.com"]
        dns_beta["my-website-beta.example.com"]
        dns_prod["my-website.example.com"]
    end

    cf_test -.-> dns_test
    cf_beta -.-> dns_beta
    cf_prod -.-> dns_prod
```

Here's what it shows:

- **Git Repository**: The dev → test → main branch merge strategy
- **Pipelines**: Each stage (test, beta, prod) gets its own CodeBuild-only pipeline, triggered by its corresponding branch
- **S3 Origin Bucket**: A single shared bucket (no StageId in the name) with {StageId}/public/ prefixes isolating each stage's content. CodeBuild syncs build output here. No public access.
- **CloudFront Distributions**: One per stage, using Origin Access Control (OAC) to read from the corresponding S3 prefix. Test/dev environments use CachingDisabled.
- **Route 53 / Custom Domains**: Optional custom domains — non-prod stages get a stage suffix (e.g., my-website-test.example.com), while prod uses the clean subdomain

## 7. Advanced Concepts

As you deployed the CloudFormation distribution, you may have noticed the ability to add an API Gateway as an endpoint alongside the static website.

This would provide a site map such as:

```
|- /    <-- uses S3 as origin
|- /api <-- uses API Gateway as origin
```

The S3 origin would map to `s3://yourbucket/test/public/*

The API Gateway origin would map to `apigwid.execute-api.us-east-2.amazonaws.com/acme-my-api-test/*`

So, if your API Gateway Endpoint was:

```
apigwid.execute-api.us-east-2.amazonaws.com/acme-my-api-test/v1/users
```

The CloudFront location would be:

```
distid.cloudfront.net/api/v1/users
```

And static content would be:

```
distid.cloudfront.net/
```

### 7.1 Deploy an API Behind CloudFront

1. In the SAM Config repository, use the `create_repo.py` script to create a new serverless application repository seeding it with Atlantis Starter 00 Basic Node.js
2. `clone` the repo to your machine and `merge` the `dev` branch into `test`, and `push`.
3. Use `config.py pipeline` and `deploy.py pipeline` to create a pipeline for your `test` branch. You **must** name the project with the SAME `ProjectId` as your website project.
4. After the pipeline has deployed, and you have checked that your application is reachable, re-configure and deploy the `network` stack.
5. Use `config.py network` and `deploy.py network` to add the API Gateway ID to the `test` CloudDistribution.

Once the distribution has deployed, you should be able to access your endpoint using the distribution URL.

### 7.2 Display Your API Content on Your Site

Modify your web site to fetch data from the endpoint and display it on your site.

### 7.3 Production Use

> The following concepts of CloudFront Cache Invalidation and Logging are beyond the scope of this tutorial but may be explored on your own in the future. They are not required to finish this tutorial unless you are otherwise instructed by your supervisor or instructor.

While the stacks you deployed have best-practices built in, typical production deployments also include Logging and CloudFront cache invalidation.

#### Access Logging

Logging can be set at two levels:

- CloudFront (Web Logs)
- S3 access

CloudFront web access logs will include information about the request from the client, such as:

```
#Version: 1.0
#Fields: date time x-edge-location sc-bytes c-ip cs-method cs(Host) cs-uri-stem sc-status cs(Referer) cs(User-Agent) cs-uri-query cs(Cookie) x-edge-result-type x-edge-request-id x-host-header cs-protocol cs-bytes time-taken x-forwarded-for ssl-protocol ssl-cipher x-edge-response-result-type cs-protocol-version fle-status fle-encrypted-fields c-port time-to-first-byte x-edge-detailed-result-type sc-content-type sc-content-len sc-range-start sc-range-end
2026-03-30	02:33:55	MSP50-P1	9754	xx.xx.41.39	GET	d16wymzfzyj9xi.cloudfront.net	/	200	-	Mozilla/5.0%20(Macintosh;%20Intel%20Mac%20OS%20X%2010.15;%20rv:149.0)%20Gecko/20100101%20Firefox/149.0	-	-	Miss	cOJMEmKzoOA6fcqv-Yd4NlargWoqUENHMC4mBo3IXtue3zK_cvBV4A==	d16wymzfzyj9xi.cloudfront.net	https	301	0.276	-	TLSv1.3	TLS_AES_128_GCM_SHA256	Miss	HTTP/2.0	-	-	63617	0.276	Miss	text/html	9398	-	-
2026-03-30	02:33:55	MSP50-P1	9754	xx.xx.41.39	GET	d16wymzfzyj9xi.cloudfront.net	/favicon.ico	200	https://d16wymzfzyj9xi.cloudfront.net/	Mozilla/5.0%20(Macintosh;%20Intel%20Mac%20OS%20X%2010.15;%20rv:149.0)%20Gecko/20100101%20Firefox/149.0	-	-	Miss	V3av-BbJHF-LY3jIQ7FJM3f-IpfrPrDcrJ3ess4SsUt_rJpxG5IYsw==	d16wymzfzyj9xi.cloudfront.net	https	146	0.187	-	TLSv1.3	TLS_AES_128_GCM_SHA256	Miss	HTTP/2.0	-	-	63617	0.186	Miss	text/html	9398	-	-
2026-03-30	02:34:08	MSP50-P1	17851	xx.xx.41.39	GET	d16wymzfzyj9xi.cloudfront.net	/docs/use-cases	200	https://d16wymzfzyj9xi.cloudfront.net/	Mozilla/5.0%20(Macintosh;%20Intel%20Mac%20OS%20X%2010.15;%20rv:149.0)%20Gecko/20100101%20Firefox/149.0	-	-	Miss	EMMBfgaG4UIG93jLWH1GVrN5b-Y1kJiw5YhI2bq5_9I8UzW-rdesMg==	d16wymzfzyj9xi.cloudfront.net	https	36	0.159	-	TLSv1.3	TLS_AES_128_GCM_SHA256	Miss	HTTP/2.0	-	-	63617	0.158	Miss	text/html	17476	-	-
2026-03-30	02:34:08	MSP50-P1	5981	xx.xx.41.39	GET	d16wymzfzyj9xi.cloudfront.net	/docs/css/style.css	200	https://d16wymzfzyj9xi.cloudfront.net/docs/use-cases	Mozilla/5.0%20(Macintosh;%20Intel%20Mac%20OS%20X%2010.15;%20rv:149.0)%20Gecko/20100101%20Firefox/149.0	-	-	Miss	6JfsQxpixVR5bUMFYD1gKVQDL0-yh_mzz0mTOlakFbkqDVnPYPxnfg==	d16wymzfzyj9xi.cloudfront.net	https	104	0.126	-	TLSv1.3	TLS_AES_128_GCM_SHA256	Miss	HTTP/2.0	-	-	63617	0.126	Miss	text/css	5635	-	-
```

S3 access logs will include information about any resource or client that accessed or tried to access objects:

```
efd-somestring-df11 acme-my-website-origin-123456789012-us-east-2-an [30/Mar/2026:01:09:38 +0000] xx.xx.41.39 - C36TRGH4ZJ7R8RHZ REST.OPTIONS.PREFLIGHT - "OPTIONS /acme-my-website-origin-123456789012-us-east-2-an HTTP/1.1" 200 - - - 5 - "https://123456789012-z2xzmo4z.us-east-2.console.aws.amazon.com/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:149.0) Gecko/20100101 Firefox/149.0" - z6uFa0AzIZyZEnXwUERc/uZizyBwFwzP36L7dyKP4Wf84zTwlvwCJrzT+dhyY9GbBW/h+eIVLIgtYF2448YcoVsf6kdzPUv5 - TLS_CHACHA20_POLY1305_SHA256 - s3.us-east-2.amazonaws.com TLSv1.3 - - -
efd-somestring-df11 acme-my-website-origin-123456789012-us-east-2-an [30/Mar/2026:02:30:57 +0000] 10.0.71.123 arn:aws:sts::123456789012:assumed-role/acme-Worker-my-website-test-DeployServiceRole/AWSCodeBuild-bbd32a55 ASNF4YJCAHYN6TT3 REST.PUT.OBJECT test/public/docs/use-cases/index.html "PUT /test/public/docs/use-cases/index.html HTTP/1.1" 200 - - 17476 54 17 "-" "aws-cli/2.34.11 md/awscrt#0.31.2 ua/2.1 os/linux#4.14.355-280.714.amzn2.x86_64 md/arch#x86_64 lang/python#3.13.11 md/pyimpl#CPython exec-env/AWS_ECS_EC2 m/b,E,W,G,Z cfg/retry-mode#standard md/installer#exe md/distrib#amzn.2023 md/prompt#off md/command#s3.sync" - 0e...B4lA= SigV4 TLS_AES_128_GCM_SHA256 AuthHeader acme-my-website-origin-123456789012-us-east-2-an.s3.us-east-2.amazonaws.com TLSv1.3 - - us-east-2
```

In this example you'll see the first line demonstrates access from CloudFront. The second line demonstrates access from the CodeBuild stage during deployment.

Logs are useful for troubleshooting and security, and should be retained for a specified amount of time as required by law or organizational guidelines.

The logs are also most useful when they can be queried using [Amazon Athena](https://aws.amazon.com/athena/).

To utilize logging you will need to:

1. Set up a logging storage bucket (`config.py storage` and choose `template-storage-s3-access-logs`)
2. Configure your `storage` and `network` stacks to send logs to the logging bucket.

You can examine the [logging storage template](https://github.com/63Klabs/atlantis-sam-templates/blob/main/templates/v2/storage/template-storage-s3-access-logs.yml) on the Atlantis SAM Templates GitHub repository.

#### CloudFront Cache Invalidation

CloudFront acts as a CDN (Content Delivery Network) wich caches your pages so they are able to be quickly accessed by a global audience.

If you have experienced caching behavior before, you know that it can pose issues when content is updated and the cache has not yet expired. To overcome this, CloudFront allows cache invalidation requests.

Ideally you want to invalidate only the content that has been updated. 

One of the methods to do this is to use S3 Events and send the context of the event (what object was added, modified, deleted) through a process that will submit an invalidation request to CloudFront.

Luckily the process to accomplish this is ready to deploy using the [Serverless CloudFront Cache Invalidation](https://github.com/63Klabs/atlantis-starter-03-serverless-cloudfront-cache-invalidation) available as an application starter.

To utilize the cache invalidator you will need to:

1. Have a site infrastructure set up (repo, pipeline, storage, network)
2. Install the cache invalidator application (`create_repo.py` and choose Starter 03 `serverless-cloudfront-cache-invalidation` to seed the repository. Then be sure to perform the config and deploy steps)
3. Configure your site storage stack to send events to the invalidator
4. Configure your site network stack to accept invalidation requests

Follow the instructions for the Cache Invalidator to ensure the configurations meet your needs.

> Note: The invalidator only works with `PROD` instances (`beta`, `prod`, etc). By default the `network` stack does not perform caching for non-`PROD` environments so you will not see any activity on a `test` branch.

## 8. Clean-Up

To avoid ongoing charges to your AWS account, delete the resources created in this tutorial.

> Note: This step is optional and is dependent upon your user permissions and whether or not you wish or are required to delete the stacks created in this tutorial. It is recommended, for practice and if you have the proper permissions, to delete your "PROD" (beta and prod) stages. This helps reduce cost and re-enforce your knowledge of stack management. You can always go through the steps of creating and deploying the stage later. That's the nice thing about automation!

The `delete.py` script is provided to perform clean-up operations in proper order.

You will be required to provide the ARN for each stack you delete. You may obtain these from the Stack Info tab in the CloudFormation web console.

As the accidental deletion of stacks can be devastating, the delete script requires several confirmation steps.

We will delete:

- `prod` pipeline stack
- `prod` network stack
- `beta` pipeline stack
- `beta` network stack
- `test` pipeline stack
- `test` network stack
- `test` application and pipeline stacks
- shared storage stack

> Proceed with caution! Double check your work and make sure you are deleting the correct stack!

There are 2 manual steps that need to take place **ON THE PIPELINE AND APPLICATION STACKS** prior to running the delete script. Some organizations may restrict who can perform these steps to ensure proper checks and balances.

1. Manually add a tag to the stack with the key `DeleteOnOrAfter` and a value of a date in `YYYY-MM-DD` format. (Add `Z` to end for UTC. Example `2026-07-09Z`). This can be done using the AWS CLI:
  - `aws resourcegroupstaggingapi tag-resources --resource-arn-list "arn:aws:cloudformation:region:account:stack/stack-name/stack-id" --tags DeleteOnOrAfter=YYYY-MM-DD --profile your-profile`
  - Be sure to replace the ARN in the command with the pipeline stack ARN.
  - Successful completion will result in receiving an empty `FailedResourcesMap`
  - You can double check by going to the pipeline stack in the CloudFormation console.
2. Disable termination protection: `aws cloudformation update-termination-protection --stack-name STACK_NAME --no-enable-termination-protection`
    - Be sure to replace `STACK_NAME` with the name of the stack. You do not need the full ARN for this command.
  - Do this for each `pipeline`, `network`, and `storage` stack you wish to delete.

The delete script is now ready to be ran from the SAM config repository:

```bash
# Perform this command in the SAM Config Repo
./cli/delete.py pipeline acme my-website beta --profile YOUR_PROFILE
```

You will have the chance to either retain the stage's environment settings in the `samconfig` file for later re-deployment, or to delete it completely. Once all stage environments of a `samconfig` file are deleted the file and directory for that project is also deleted.

Be sure to perform this operation for any unwanted stages of your site (`test`, `beta`, `prod`, etc.).

Performing the delete does not delete the repository. Since the size of the repository is minimal you will not incur charges and may leave the repository as-is for future reference.

## Summary

Congratulations! You have completed Tutorial #03! You have successfully deployed a static website using an automated CI/CD pipeline.

You can now use this knowledge to deploy more complex applications and services using the same principles and techniques.

- [Next Tutorial: Tutorial #04: Atlantis SAM Config Scripts In-Depth](../04-atlantis-sam-config-scripts-in-depth/README.md)
- [All Tutorials](../../README.md)