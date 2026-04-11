# Atlantis SAM Config Scripts In-Depth

For additional documentation, review the [Atlantis SAM Configuration Scripts repository](https://github.com/63klabs/atlantis-sam-config-scripts).

Each script accepts the `-h` flag to provide guidance on their use, arguments, options, and flags.

Open your organization's SAM Configuration repository and for each script below, use the `-h` option to learn more about the script. 

## create_repo.py

Create a GitHub or CodeCommit repository and seed it from an application starter, GitHub repo, or a ZIP stored S3 using the create_repo.py script

For usage info:

```bash
./cli/create_repo.py -h
```

Any tag you add to a CodeCommit repository will be used during the tag configuration phase of the `config.py` script for `pipeline` stacks. This assists in tag propagation.

## config.py

Create and maintain a CloudFormation stack utilizing stack options stored in a samconfig file and a central template repository managed by your organization (or 63Klabs for training and getting started).

The config script will walk you through selecting a template and filling out stack parameters, options, and tags.

At the end of configuration it will present the option to move right into the deploy script.

For usage info:

```bash
./cli/config.py -h
```

## deploy.py

After configuring a stack, you must deploy it. Although the scripts maintain a samconfig file, it utilizes a template repository hosted in S3, which, oddly, samconfig does not support even though CloudFormation templates can easily import Lambda code and nested stacks from S3.

Behing the scenes the deploy script runs the `sam deploy` command after obtaining the template from S3 and providing it to the command as a temporary, local file.

For usage info:

```bash
./cli/deploy.py -h
```

## import.py

You probably have some stacks that were deployed prior to using the CLI scripts to manage them.

The `import.py` script allows you to import current stack configurations to a samconfig file.

You can also import the template that was used (if you need a copy).

From there you can tweak the `samconfig` file, upload the template to a central location (or apply a different one), and after formatting and saving to the proper location for the scripts to use, utilize the CLI scrips.

For usage info:

```bash
./cli/import.py -h
```

## update.py

Various enhancements and fixes will be released over time so the update script is essential in making sure you have the most current scripts.

> **NOTE:** Refer to your organization's policy reguarding WHO should be performing the updates and WHEN the updates should be performed.

The script is very friendly as it will kindly remind, and perform a pull of the configuration repository before proceeding. It will then push all changes back after completion. This ensures that the push/pull step is not forgotten and all subsequent pulls by developers have the new files!

For usage info:

```bash
./cli/update.py -h
```

## delete.py

Deleting applications and their pipelines require a specific order to be followed. The application stack must be deleted first, followed by the pipeline, SSM parameters that were created during the build process, and finally clean-up of the SAM configuration file.

Lucky, the `delete.py` script takes care of all of this.

For usage info: 

```bash
./cli/delete.py -h
```

The delete.py script has the following features to aid in clean-up:

- Pipeline destruction workflow:
	1. Prompts for git pull
	2. Validates pipeline and application stack ARNs
	3. Checks DeleteOnOrAfter tag with date validation (supports both local and UTC dates) and ensures termination protection is disabled.
	4. Final confirmation by entering prefix, project_id, and stage_id
	5. Deletes application stack first, then pipeline stack
	6. Deletes associated SSM parameters
	7. Updates/deletes samconfig entries
	8. Performs git commit and push
- Safety features:
	- Multiple validation steps before deletion
	- Proper error handling and logging
	- Stack deletion waiter to ensure completion
- Batch processing for SSM parameter deletion
- Future extensibility: Framework in place for storage, network, and iam types (currently shows not implemented message)

## Summary

Congratulations! You have completed Tutorial #04!

- [Next Tutorial: Tutorial #05: Run CodeBuild on a Schedule for Operations](../05-run-codebuild-on-a-schedule-for-operations/README.md)
- [All Tutorials](../../README.md)