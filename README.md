# sfmt
Apache Ant build scripts for the Salesforce Force.com Migration Tool.

## Installation
To use these build scripts you may first need to install [Apache Ant](https://ant.apache.org/) and add the [Salesforce Force.com Migration Tool plugin](https://developer.salesforce.com/page/Migration_Tool_Guide) to you Ant library path.


## Usage
As an Apache Ant build script, this tool is intended to be invoked form a command line with the basic form of
```
ant <target> [-D<property>=<value> ...]
```

Properties must be passed in with the -D flag to be recognized within the ant build scripts. The advantage of defining everything within the build scripts rather than some other wrapper is that this tool should be platform independent and not require any additional software to run.

> Note: Some properties names have periods in them, which can cause issues in Windows Powershell. There are short properties names for the most common properties, but if you need to define another you can enclose the entire option in quotes, e.g. `ant new -Dwork=old "-Dsfmt.api.version=38.0"`

### Terms and Definitions
A *Work Item* (`work` for short) refers to any set of changes to be made in a migration. The term is intentionally ambiguous to be open to different work flows, e.g.
* as "projects", where a single package list all related components and migrates the complete project whether individual components are changed or not.
* as "tasks", where individual packages are created to response to specific tasks, e.g. from a bug tracker.

A `patch` is an update to the work item package prior to completing the full migration.

The `source` is the Salesforce Org where the metadata components originates. An *environment* (`env` for short) is a Salesforce Org targeted for an action.

### Configuration
To interact with a Salesforce Org you must provide a username and password (or a session ID, but this tool hasn't been built for that option yet) to the tool.

These values can be set with the `auth.username` and `auth.password` properties in the `build.properties` file or as options on the command line.

Set the username to your Production Org username and the build script will automatically append the environment name when another Org is targeted.

This tool was built for a single Production Org with one or more Sandboxes, and assumes your Sandbox usernames and passwords are the same as your Production Org with the Sandbox name appened to the end of the username.
If you need to work with another Org you can override these properties on the command line.
```
ant <target> -Dsfdc.conf.auth.username=me@my-other.org -Dsfdc.conf.auth.password=notmypassword -Dsfdc.conf.auth.serverurl="https://login.salesforce.com"
```

> Note that this means you are providing or saving your username and password in plain text (which I'm not happy about and am looking for alternatives), so take care sharing these files (i.e. don't add `build.properties` to a git commit).

### Creating a New Migration
There are two types of migrations:
1. Work Item
2. Patch

A Work Item Migration is a complete migration from source to final destination. A Patch is a modification to a Work Item, for example to update a component that had issues or add a new component.

When starting our with a migration you should always create a Work Item migration first, and create Patches only as needed.

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
A Salesforce Migration consist of at least 2 steps:
1. Download Salesforce Metadata Components from a Salesforce Organization.
2. Upload Salesforce Metadata Components to a Salesforce Organizaton.

In the Migration Tool these and referred to as a *retrieve* and *deploy*, respectively. A deploy task can also run with with the checkOnly option which runs any pre-deployment tests but rollsback without making changes to the environment; refered to as a *validation*. Each deploy task, whether as a full deploy or just a validation, will generate a Request ID, recognizable as a 15-18 character Salesforce ID, which can be used to check on or complete the deployment at a later time.

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
Every new Work Item will have its own `build.xml` file which links back to the primary `build.xml` script. This allows all commands to be run from within the work item directory and, when paired with the work items `build.properties` that should set `work=<work_item_name>`, can be run without the additional command line argument. E.g.
```
ant new -Dpatch=<patch_name>
ant fetch-from-source [-Dpatch=<patch_name>]
ant validate-in-env [-Dpatch=<patch_name>] -Denv=<environment_name>
ant deploy-in-env [-Dpatch=<patch_name>] -Denv=<environment_name> -Drequest=<request_id>
```

Additionally, this work item build script can be modified to for special behavior specific to work item.

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
