# Deploying a Static Website

> This tutorial is still under development. However, the basic structure is listed below. Please be advised that the content is short, may be missing, and may have inaccuracies or typos. If you would like to contribute updates, please submit an [issue via this repository on GitHub](https://github.com/63Klabs/atlantis-tutorials/issues). Be sure to include the page and what content should be added/updated. If you'd be willing to write a few sentences (or more, but be clear and succinct), please do. Thank you for your understanding.

[AWS Amplify](https://aws.amazon.com/amplify) is a powerful, managed solution for deploying websites. It is recommended for those who need a dead simple method of deploying a static website or web application as it handles the deployment, certificates, custom domains, hosting, and more for you.

**However**, if you want to get more hands-on with the development, configuration, and deployment of a static website using S3, a Pipeline, and CloudFront paired with Route53, use this tutorial to explore the solution.

## 1. reate Your Repository
From the SAM Config repository:

```bash
./cli/create_repo.py your-repo-name --profile default
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
./cli/config.py storage acme my-website --profile default
./cli/deploy.py storage acme my-website --profile default
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
./cli/config.py pipeline acme my-website test --profile default
# - Use CodeBuild Only Pipeline
# - For Static Host bucket use the S3 bucket name
./cli/deploy.py pipeline acme my-website test --profile default
```

After the pipeline has deployed, use the Output link to see it in action. Check for any errors and follow along in the CodeBuild console as it builds and copies your site to S3.

Once the deployment has finished, go to the S3 console and view the contents of your S3 bucket. You should see `test/public/<your-files>`

## 4. Configure and Deploy the `test` CloudFront Distribution (Route53 optional)

We will now create the CloudFront Distribution that uses your bucket with the `test/public` object prefix as an origin.

From the SAM Config repository:

```bash
./cli/config.py network acme my-website test --profile default
# - There are a lot of parameters, for most you will accept the defaults
# - Use S3 Bucket Origin Domain from storage output
# - A custom domain for Route53 is optional, you can just use the provided CloudFront domain for the tutorial and development
./cli/deploy.py network acme my-website test --profile default
```

From the output section you should see the CloudFront distribution domain. Follow the link and you should see your site.

## 5. Add `beta` and `prod`

Go through the same steps above, this time creating a `beta` and `prod` deployment.

You may need to add a `beta` branch to your repository.

## 6. Architecture Overview

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

    users(("Users")) --> dns_prod
    users --> dns_beta
    users --> dns_test
```

