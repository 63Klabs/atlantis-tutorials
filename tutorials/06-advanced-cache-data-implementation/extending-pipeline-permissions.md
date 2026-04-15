# Extending Pipeline Permissions

> NOTE: This will be included as a tutorial at a later date.

The template used for pipelines provides a limited set of permissions for CodeBuild, CloudFormation, and PostBuild activities.

This limited scope helps secure the pipeline, limiting what it can create, modify, or delete.

For example, the CodeBuild phase of the pipeline only has access to the HOST_BUCKET provided by the template parameter ``.

The CodeDeploy phase only allows the CloudFormation service to deploy a limited set of resources including CloudWatch logs, Alarms, SQS, Lambda, Lambda Layers, Step Functions, DynamoDB, Events, and S3.

## Attaching Managed Policies

If you wanted to provide any phase with access to another S3 bucket, or you want CloudFormation to be able to create Cognito resources, you can create and attach managed policies.

When you configure your pipeline (using `config.py pipeline`) at some point you will be prompted to attach managed policies. Be sure to attach the right policy to the correct stage.

> You can attach more than one managed policy ARN as a comma delimited value.

**External Resources:**
- `CloudFormationSvcRoleIncludeManagedPolicyArns`
- `CodeBuildSvcRoleIncludeManagedPolicyArns`
- `PostDeploySvcRoleIncludeManagedPolicyArns`

## Creating Managed Policies

Managed policies can be created and mainted one of two ways:
- Resource specific for shared resources: created alongside the resource it is giving permission to. Cache-Data does this when it creates the DynamoDB and S3 bucket for storing cached data. It also creates a managed policy that can be attached to Lambda function execution roles so that the functions have access to use the resource.
- Generic yet Scoped by Caller: created and maintained in a central account-wide template to be re-used by other templates to manage their resources, yet maintaining scope to provide scoped access.

## Generic yet Scoped by Caller

This approach is called Attribute-Based Access Control (ABAC).

By using the `${aws:PrincipalTag/Key}` variable and comparing it to the `aws:ResourceTag/Key` in a policy Condition, you can create a single "template" policy that scales across all projects without modification.

The policy will check if the tag on the CloudFormation Service Role (the Principal) matches the tag on the Resource it is trying to create or modify.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowActionsIfTagsMatch",
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances",
                "ec2:CreateVolume",
                "s3:CreateBucket",
                "rds:CreateDBInstance"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/Application": "${aws:PrincipalTag/Application}"
                }
            }
        }
    ]
}
```

### Critical Implementation Details

To make this work in a real-world CloudFormation environment, you must address three specific challenges:

#### 1. The "Chicken and Egg" Problem

CloudFormation creates resources. A resource doesn't have tags until it is created.

* Fix: Use the aws:RequestTag/Key variable for "Create" actions. This ensures that the tags being sent in the request match the role's tag.
* Updated Condition:

```json
"Condition": {
    "StringEquals": {
        "aws:RequestTag/Application": "${aws:PrincipalTag/Application}"
    }
}
```

#### 2. Prevent Tag Tampering
You must ensure the CloudFormation role cannot change its own Application tag or remove the Application tag from resources to bypass security.

* Action: Add a Deny statement for UntagResource or CreateTags unless the tag value matches the principal.

#### 3. Resource Support
Not all AWS resources support tagging upon creation or authorization via tags.

* Check: Verify that the specific resources your projects use (e.g., Cognito, EC2, etc.) support the aws:ResourceTag condition key.
