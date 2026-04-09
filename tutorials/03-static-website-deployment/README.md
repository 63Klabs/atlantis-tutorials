# Deploying a Static Website

> This tutorial is still under development. However, the basic structure is listed below.

AWS Amplify is a powerful managed solution for deploying websites. However, if you want to get more hands-on with the development, configuration, and deployment of a static website using S3, a Pipeline, and CloudFront paired with Route53, use this tutorial to explore the solution.

From the SAM Config repository:

```bash
./cli/create_repo.py your-repo-name --profile default
# Choose 'None' for seeding
```

Clone your repository, create a dev branch, and then scaffold it with React (or your framework choice):

```bash
npm create vite@latest my-website -- --template react
cd my-website
npm install
npm run dev
```

Checkout your site on the local machine.

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

## Deploy S3 Bucket

From the SAM Config repository:

```bash
./cli/config.py storage acme my-website --profile default
./cli/deploy.py storage acme my-website --profile default
# Make note of the S3 bucket name AND S3 bucket domain
```

## Deploy your Pipeline

```bash
./cli/config.py pipeline acme my-website test --profile default
# - Use CodeBuild Only Pipeline
# - For Static Host bucket use the S3 bucket name
./cli/deploy.py pipeline acme my-website test --profile default
```

## Deploy CloudFront

```bash
./cli/config.py network acme my-website test --profile default
# - Use S3 Bucket Origin Domain from storage output
./cli/deploy.py network acme my-website test --profile default
```