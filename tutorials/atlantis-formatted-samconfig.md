# Atlantis Formatted samconfig File

The scripts in the SAM Config Repository utilize the `samconfig` file format, adhering to structure and properties used nativly by SAM to deploy applications.

For the most part, you could take the `samconfig` file and perform a `sam deploy` against it except for one crucial difference:

AWS SAM config DOES NOT support S3 locations for the `template_file` property.

This is a HUGE limitation since S3 is a great way to manage shared templates (and versions) across an organizations.

CloudFormation can import templates from S3 locations, Lambda can deploy ZIP files from S3, whey can't `samconfig`?

Since this limitation is a show stopper for most to adopt the `sam` cli for day-to-day needs beyond a tutorial, and we still need a way to solve the chicken and egg problem for deploying a pipeline stack that will in turn automate deployments, utilizing scripts such as `config.py` and `deploy.py` came into being.

While still maintaining the overall `samconfig` structure and not coming up with a proprietary or custom solution for storing stack configuration, the scripts only add missing features such as:

1. Duplication and management of stack parameters across deployments 
2. S3 template file location (with versioning)
3. Easier formatting of `parameter_overrides` and `tags`

While `config.py` reads in a `samconfig` file, it also reads in the corresponding template file in order to provide helpful prompts for stack parameters. It also copies "global" configurations from the non-standard `atlantis.deploy.parameters` section and applies it to the other `deploy.parameter` sections. It then saves the `samconfig` back to a standard `toml` file.

For deployments, `deploy.py` reads the file, obtains the location and version of the template used, downloads it to a temporary directory and performs a `sam deploy` in the background while pointing to the local, temporary template file.

These scripts incorporate Infrastructure as Code while still maintaining native AWS SAM configuration and deployment support.
