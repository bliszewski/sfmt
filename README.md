# sfmt
Apache Ant build scripts for the Salesforce Force.com Migration Tool.

## Terms and Definitions
A *Migration* is any set of changes made in a Salesforce Org to be moved into another Salesforce Org. This tool attempts to account for both Metadata Components, which are specified in the `package.xml` file or manual configurations, listed in the `README.txt` file.

A *Work Item* (`work` for short) refers to any set of changes to be made in a migration. The term is intentionally ambiguous to be open to different work flows, e.g.
* as "projects", where a single migration list all related changes and migrates the complete project whether individual components are changed or not.
* as "tasks", where individual migrations are created to response to specific tasks, e.g. from a bug tracker.

A `patch` is an update to the work item package prior to completing the full migration.

An *environment* (`env` for short) is a Salesforce Org targeted for an action.

The `source` is the Salesforce Org where the metadata components originates.

## Installation
To use these build scripts you may first need to install [Apache Ant](https://ant.apache.org/) and add the [Salesforce Force.com Migration Tool plugin](https://developer.salesforce.com/page/Migration_Tool_Guide) to you Ant library path.

## Configuration
Edit the `*.properties` files with configuration settings.

* `build.properties` defines settings for the build scripts.
* `sfdc.properties` defined settings for the Force.com Migration Tool.

### Username, Password, and Server URL
A username, password, and server URL needs to be provided to connected to a Salesforce Org. 

> Session IDs as another option but are not currently supported with this tool.

Set `auth.username` and `auth.password` in the `build.properties` file or as options on the command line. 

> Be careful about adding `build.properties` in your repository if you put your username and password there. I'm not happy about keeping these values in plaintext, but that seems to be standard practice.

These properties should be set to your Production Org username and password; sandbox names are appened by the script when another environment is targeted.

The Server URL is defined by Salesforce. While you can provide a unique URL to your Salesforce Org (rather, the Salesforce instance), Salesforce provides 2 generic URLs:

1. `https://login.salesforce.com` for Production Orgs.
2. `https://test.salesforce.com` for Sandboxes.

Generally, You will not need to specify a server URL; the tool will do select it for you.

> This tool was built around a single Production Org with one or more Sandboxes, and assumes your Sandbox usernames and passwords match your production environment, following the standard behavior to appened a sandbox name to the end of the username. If you need to work with another Org you can override these properties on the command line.
```
ant <target> -Dsfdc.conf.auth.username=me@my-other.org -Dsfdc.conf.auth.password=notmypassword -Dsfdc.conf.auth.serverurl="https://login.salesforce.com"
```

## Usage
As an Apache Ant build script, this tool is intended to be invoked form a command line with the basic form of
```
ant <target> [-D<property>=<value> ...]
```

`target` is the name of a `<target>` item within the build script. Common targets are:
```
new                Create a new migration work item or patch.
fetch-from-source  Retrieve the package.xml from source environment.
validate-in-env    Perform a checkOnly=true deploy task in an environment.
deploy-in-env      Perform a deployRecentValidation task in an environment.
```

Properties are passed to Apache Ant with the `-D` flag. 

The most common properties are:
```
work     Name of the migration work item.
patch    Name of the migration patch within a work item.
env      Name of the environment to target.
source   Name of the environment to use as a source.
request  Request ID of a recent deploy (or validation) task.
```

> Note: Some properties names have periods in them, which can cause issues in Windows Powershell. There are short properties names for the most common properties, but if you need to define another you can enclose the entire option in quotes, e.g. `ant new -Dwork=old "-Dsfmt.api.version=38.0"`

### Creating a New Migration
There are two types of migrations:
1. Work Item Migration, a standard migration of components and configurations from one Org to another.
2. Patch Migration, a modification to a migration, e.g. to fix a bug that was discovered during migration.

Always start with a Work Item migration and use patches to update the package if needed.

Using the build script to create a new Work Item migration will
1. Make a directory with the work item name in the work items directory.
2. Write a `package.xml` file in the above directory.
3. Write a `README.txt` file in the above directory.
4. Write a `build.xml` file in the above directory, linking back to sfmt's `build.xml`.
5. Write a `build.properties` file in the above directory where you can store properties for the migration.

Creating a new Patch migration will
1. Write a patch `package.xml` file in the above directory.
2. Write a patch `README.txt` file in the above directory.
3. Write a patch `build.properties` file in the above directory where you can store properties for the migration.

#### Work Item Migration
```
ant new -Dwork=<workitem_name>
```
#### Patch Migration
```
ant new -Dwork=<workitem_name> -Dpatch=<patch_name>
```

### Performing a Migration
A Salesforce Migration always consist of at least 2 steps:
1. Download Salesforce Metadata Components from a Salesforce Organization.
2. Upload Salesforce Metadata Components to a Salesforce Organizaton.

In the Force.com Migration Tool these and referred to as a *retrieve* and *deploy*, respectively. A deploy task can also run with with the `checkOnly` option to run pre-deployment tests but rollsback without making changes to the environment; this is referred to as a *validation*. 

Each deploy task, whether as a full deploy or just a validation, will generate a Request ID, recognizable as a 15-18 character Salesforce ID, which can be used to check on or complete the deployment at a later time.

> The 15- and 18-character IDs are effectively the same, except that the 15-character ID is case-sensitive while the 18-character ID is not; the extra 3 characters encode the proper casing. Using the 18-character ID is recommended anywhere case may be lost or distorted.

A deploy task may run Apex tests which must all pass before the task can successfully complete. With the stock migration tool, running tests is default when deploying into Production but not when deploying to a Sandbox.

sfmt makes validation a necessary step for all migrations, thus the sfmt migration is performed with 3 steps:
1. Retrieve Salesforce Metadata Components from a Salesforce Organization.
2. Validate Salesforce Metadata Components in a Salesforce Organizaton.
3. Deploy the validated Salesforce Metadata Components in a Salesforce Organizaton.

#### Retrieve Work Item Package from Source
```
ant fetch-from-source -Dwork=<workitem_name> [-Dpatch=<patch_name>]
```

#### Validate Work Item Package in Destination
```
ant validate-in-env -Dwork=<workitem_name> [-Dpatch=<patch_name>] -Denv=<environment_name>
```

#### Deploy Work Item Package in Destination
```
ant deploy-in-env -Dwork=<work_item_name> [-Dpatch=<patch_name>] -Denv=<environment_name> -Drequest=<request_id>
```

### Alternative Work Item Build Script
A new `build.xml` file is copied into the Work Item directory which imports the primary `build.xml` script. This allows all commands to be run from within the work item directory, and if the `work` properties is set in the local `build.properties` can be run without specifying the work item on the command line argument. E.g.
```
ant new -Dpatch=<patch_name>
ant fetch-from-source [-Dpatch=<patch_name>]
ant validate-in-env [-Dpatch=<patch_name>] -Denv=<environment_name>
ant deploy-in-env [-Dpatch=<patch_name>] -Denv=<environment_name> -Drequest=<request_id>
```

This `build.xml` can also be modified to create special tasks specific to the work item.

## TODO
- [x] runnable targets to create required migration resources.
- [x] runnable targets to retrieve, validate, and deploy migration packages.
- [x] basic support for most Force.com Migration Tool targets.
- [x] extensible build script and targets.
- [ ] logging and output parsing.
- [ ] full environment download.
- [ ] Salesforce Data Loader hooks.
- [ ] environments backups and component diffs.
- [ ] version controlled packages and environments.
