# Creating Your Own Atlantis Compatible Applications

Whether you are creating your own application from scratch, or modifying someone else's code to deploy using Atlantis, there are some key requirements to ensure deployments work withh the Atlantis Platform.

## 1. Project Structure

It is highly recommended that documentation, AI specs, IDE configs, etc, are at the root of the project/repository. This keeps the build and source directory clean.

This can be seen in the Atlantis Application Starters where all build and source code is in the `application-infrastructure` directory:

### Why?

It keeps it clean.

If you are used to a simple application, this can seem overkill, but fully-documented, complex applications used by teams with a long lifespan can include not only the README in the root directory, but also `DEPLOYMENT`, `AGENTS`, `LICENSE`, `docs/`, `.kiro/` and more.

### Is it Required?

No, when you configure your pipeline you can choose where your `buildspec.yml` can be found. So you can really use any structure you want, even renaming the `application-infrastructure` directory as long as you ensure your pipeline knows where to find it.

## 2. Parameters and Naming

Permissions, tagging, deployments, and stack exports rely on dependable naming conventions.

The Atlantis Platform uses its standard `<PREFIX>-<PROJECT_ID>-<STAGE_ID>` naming convention and parameters to assist in managing resources.

All templates deployed by Atlantis require the following parameters:

- Prefix
- ProjectId
- StageId (unless it is a shared template)
- DeployEnvironment
- S3BucketNameOrgPrefix (if deploying an S3 bucket)
- RolePath
- PermissionsBoundaryArn

All application templates deployed by Atlantis Pipelines are required to accept the following Parameters:

- Prefix
- ProjectId
- StageId
- S3BucketNameOrgPrefix
- RolePath
- DeployEnvironment
- ParameterStoreHierarchy
- DeployRole
- AlarmNotificationEmail
- PermissionsBoundaryArn

### Why?

This 