<!--
SPDX-FileCopyrightText: 2025-present Stuart Ellis <stuart@stuartellis.name>

SPDX-License-Identifier: MIT
-->

# Copier Template for TF Tasks

[![Copier](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/copier-org/copier/master/img/badge/badge-grayscale-inverted-border-orange.json)](https://github.com/copier-org/copier) [![standard-readme compliant](https://img.shields.io/badge/readme%20style-standard-brightgreen.svg?style=flat-square)](https://github.com/RichardLitt/standard-readme) [![pre-commit](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit)](https://github.com/pre-commit/pre-commit) [![styled with prettier](https://img.shields.io/badge/styled_with-prettier-ff69b4.svg)](https://github.com/prettier/prettier)

This [Copier](https://copier.readthedocs.io/en/stable/) template provides files for a [Terraform](https://www.terraform.io/) or [OpenTofu](https://opentofu.org/) project.

The tooling uses [Task](https://taskfile.dev) as the task runner for the template and the generated projects. The tasks provide an opinionated configuration for Terraform and OpenTofu. This configuration enables projects to use built-in features of these tools to support:

- Multiple separate infrastructure components ([root modules](https://opentofu.org/docs/language/modules/)) in the same code repository, as self-contained [stacks](#stacks)
- Multiple instances of the same component with different configurations with [contexts](#contexts)
- Temporary instances of a component for testing or development with [workspaces](https://opentofu.org/docs/language/state/workspaces/).
- [Switching between Terraform and OpenTofu](#using-opentofu). Use the same tasks for both.

> We use the identifier _TF_ or _tf_ for Terraform and OpenTofu. Both tools accept the same commands and have the same behavior. The tooling itself is just called `tft` in the documentation and code.

## Table of Contents

- [A Quick Example](#a-quick-example)
- [How It Works](#how-it-works)
- [Install](#install)
- [Usage](#usage)
- [Contributing](#contributing)
- [License](#license)

## A Quick Example

To start a new project:

```shell
copier copy git+https://github.com/stuartellis/tf-tasks my-project
cd my-project
TFT_CONTEXT=dev task tft:context:new
TFT_STACK=my-app task tft:new
```

The `tft:new` task creates a stack, a self-contained Terraform module. The stack includes code for AWS, so that it will work immediately once the tfvar `tf_exec_role_arn` for the context is set to the IAM role that TF will use. Enable remote state storage by adding the settings to the [context](#contexts), or use [local state](#using-local-tf-state).

You can then start working with your stack:

```shell
# Set a default context and stack
export TFT_CONTEXT=dev TFT_STACK=my-app

# Run tasks on the stack with the configuration from the context
task tft:init
task tft:plan
task tft:apply
```

```shell
# Specifically set the stack and context for one task
TFT_CONTEXT=dev TFT_STACK=my-app task tft:fmt
```

## How It Works

First, you use [Copier](https://copier.readthedocs.io/en/stable/) to either generate a new project, or to add this tooling to an existing project.

The tooling uses specific files and directories:

```shell
|- tasks/
|   |
|   |- tft/
|       |- Taskfile.yaml
|
|- tf/
|    |- .gitignore
|    |
|    |- contexts/
|    |   |
|    |   |- all/
|    |   |
|    |   |- template/
|    |   |
|    |   |- <generated contexts>
|    |
|    |- definitions/
|    |    |
|    |    |- template/
|    |    |
|    |    |- <generated stack definitions>
|    |
|    |- modules/
|
|- tmp/
|    |
|    |- tf/
|
|- .gitignore
|- .terraform-version
|- Taskfile.yaml
```

The Copier template:

- Adds a `.gitignore` file and a `Taskfile.yaml` file to the root directory of the project, if these do not already exist.
- Provides a `.terraform-version` file.
- Provides the file `tasks/tft/Taskfile.yaml` to the project. This file contains the task definitions.
- Provides a `tf/` directory structure for TF files and configuration.

The tasks:

- Generate a `tmp/tf/` directory for artifacts.
- Only change the contents of the `tf/` and `tmp/tf/` directories.
- Copy the contents of the `template/` directories to new stacks and contexts. These provide consistent structures for each component.

### Stacks

You define each set of infrastructure code as a separate component. Each of the infrastructure components in the project is a separate TF root [module](https://opentofu.org/docs/language/modules/). This tooling refers to these TF root modules as _stacks_. Each TF stack is a subdirectory in the directory `tf/definitions/`.

To create a new stack, use the `tft:new` task:

```shell
TFT_STACK=my-app task tft:new
```

The tooling creates each new stack as a copy of the files in `tf/stacks/template/`. The template directory contains a complete, working TF module for AWS resources. This means that each new stack is immediately ready to use.

You are free to change stacks as you need. For example, you can completely remove the AWS resources. The tooling only requires that a stack is a valid TF module with these tfvars:

- `environment_name` (string)
- `product_name` (string)
- `stack_name` (string)
- `variant` (string)

> This tooling does not explicitly conflict with the [stacks feature of Terraform](https://developer.hashicorp.com/terraform/language/stacks). However, I do not currently use HCP Terraform to test with the stacks feature that it provides. It is unclear when this feature will be finalised, if it will be able to be used without a HCP Terraform account, or if an equivalent will be implemented by OpenTofu.

### Contexts

This tooling uses _contexts_ to provide profiles for TF. Contexts enable you to deploy multiple instances of the same stack with different configurations.

To create a new context, use the `tft:context:new` task:

```shell
TFT_CONTEXT=dev task tft:context:new
```

Each context is a subdirectory in the directory `tf/contexts/` that contains a `context.json` file and one `.tfvars` file per stack.

The `context.json` file is the configuration file for the context. It specifies metadata and settings for TF [remote state](https://opentofu.org/docs/language/state/remote/). Each `context.json` file specifies two items of metadata:

- `description`
- `environment`

The `description` is deliberately not used by the tooling, so that you may use it however you wish. The `environment` is a string that is automatically provided to TF as the tfvar `environment_name`. You may use this tfvar in whatever way is appropriate for the project. For example, you could define multiple contexts with the same environment.

Here is an example of a `context.json` file:

```json
{
  "metadata": {
    "description": "Cloud development environment",
    "environment": "dev"
  },
  "backend_s3": {
    "tfstate_bucket": "789000123456-tf-state-dev-eu-west-2",
    "tfstate_ddb_table": "789000123456-tf-lock-dev-eu-west-2",
    "tfstate_dir": "dev",
    "region": "eu-west-2",
    "role_arn": "arn:aws:iam::789000123456:role/my-tf-state-role"
  }
}
```

To enable you to share common tfvars across all of the contexts for a stack, the directory `tf/contexts/all/` contains one `.tfvars` file for each stack. The `.tfvars` file for a stack in the `all` directory is always used, along with `.tfvars` for the current context.

The tooling creates each new context as a copy of files in `tf/contexts/template/`. It uses `standard.tfvars` to create the tfvars files that are created for new stacks.

### Variants

The variants feature creates extra copies of stacks for development and testing. A variant is a separate instance of a stack. Each variant of a stack uses the same configuration as other instances with the specified context, but has a unique identifier. Every variant is a TF [workspace](https://opentofu.org/docs/language/state/workspaces), so has separate state.

> If you do not set a variant, TF uses the default workspace for the stack.

### Resource Names

Use the `environment`, `stack_name` and `variant` tfvars in your TF code to define resource names that are unique for each instance of the resource. This avoids conflicts.

> The test in the stack template includes code to set the value of `variant` to a random string with the prefix `tt`.

The code in the stack template includes the local `standard_prefix` to help you set unique names for resources.

### Shared Modules

The project structure also includes a `tf/modules/` directory to hold TF modules that are shared between stacks in the same project. To share modules between projects, [publish them to a registry](https://opentofu.org/docs/language/modules/#published-modules).

### Dependencies Between Stacks

By design, this tooling does not specify or enforce any dependencies between infrastructure components. If you need to execute changes in a particular order, specify that order in whichever system you use to carry out deployments.

## Install

You need [Git](https://git-scm.com/) and [Copier](https://copier.readthedocs.io/en/stable/) to add this template to a project. Use [uv](https://docs.astral.sh/uv/) or [pipx](https://pipx.pypa.io/) to run Copier. These tools enable you to use Copier without installing it.

You can either create a new project with this template or add the template to an existing project. Use the same _copy_ sub-command of Copier for both cases. Run Copier with the _uvx_ or _pipx run_ commands, which download and cache software packages as needed. For example:

```shell
uvx copier copy git+https://github.com/stuartellis/tf-tasks my-project
```

I recommend that you use a tool version manager to install copies of Terraform and OpenTofu. Consider using either [tenv](https://tofuutils.github.io/tenv/), which is specifically designed for TF tools, or the general-purpose [mise](https://mise.jdx.dev/) framework. The generated projects include a `.terraform-version` file so that your tool version manager can install the Terraform version that you specify.

## Usage

To use the tasks in a generated project you need:

- A UNIX shell, such as Bash or Fish
- [Git](https://git-scm.com/)
- [Task](https://taskfile.dev)
- [Terraform](https://www.terraform.io/) or [OpenTofu](https://opentofu.org/)

The TF tasks in the template do not use Python or Copier. This means that they can be run in a restricted environment, such as a continuous integration job.

To see a list of the available tasks in a project, enter _task_ in a terminal window:

```shell
task
```

> The tasks use the namespace `tft`. This means that they do not conflict with any other tasks in the project.

Before you manage resources with TF, first create at least one context:

```shell
TFT_CONTEXT=dev task tft:context:new
```

This creates a new context. Edit the `context.json` file in the directory `tf/contexts/<CONTEXT>/` to specify the settings for the remote state storage that you want to use and set the `environment` name.

> By default, this tooling uses [remote state](https://opentofu.org/docs/language/state/remote/) for TF. The current version always uses S3 for remote state.

Next, create a stack:

```shell
TFT_STACK=my-app task tft:new
```

Use `TFT_CONTEXT` and `TFT_STACK` to create a deployment of the stack with the configuration from the specified context:

```shell
export TFT_CONTEXT=dev TFT_STACK=my-app
task tft:init
task tft:plan
task tft:apply
```

> You will see a warning when you run `init` with a current version of Terraform. This is because Hashicorp are [deprecating the use of DynamoDB with S3 remote state](https://developer.hashicorp.com/terraform/language/backend/s3#state-locking). To support older versions of Terraform, this tooling will continue to use DynamoDB for a period of time.

### The `tft` Tasks

| Name          | Description                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------- |
| tft:apply     | _terraform apply_ for a stack\*                                                                   |
| tft:check-fmt | Checks whether _terraform fmt_ would change the code for a stack                                  |
| tft:clean     | Remove the generated files for a stack                                                            |
| tft:console   | _terraform console_ for a stack\*                                                                 |
| tft:destroy   | _terraform apply -destroy_ for a stack\*                                                          |
| tft:fmt       | _terraform fmt_ for a stack                                                                       |
| tft:forget    | _terraform workspace delete_ for a variant\*                                                      |
| tft:init      | _terraform init_ for a stack                                                                      |
| tft:new       | Add the source code for a new stack. Copies content from the _tf/definitions/template/_ directory |
| tft:plan      | _terraform plan_ for a stack\*                                                                    |
| tft:rm        | Delete the source code for a stack                                                                |
| tft:test      | _terraform test_ for a stack\*                                                                    |
| tft:validate  | _terraform validate_ for a stack\*                                                                |

\*: These tasks require that you first run `tft:init` to [initialise](https://opentofu.org/docs/cli/commands/init/) the stack.

### The `tft:context` Tasks

| Name             | Description                                                                  |
| ---------------- | ---------------------------------------------------------------------------- |
| tft:context:list | List the contexts                                                            |
| tft:context:new  | Add a new context. Copies content from the _tf/contexts/template/_ directory |
| tft:context:rm   | Delete the directory for a context                                           |

### Settings for Features

Set these variables to override the defaults:

- `TFT_PRODUCT_NAME` - The name of the project
- `TFT_CLI_EXE` - The Terraform or OpenTofu executable to use
- `TFT_VARIANT` - See the section on [variants](#variants)
- `TFT_REMOTE_BACKEND` - Enables a remote TF backend

### Updating TF Tasks

To update projects with the latest version of this template, use the [update feature of Copier](https://copier.readthedocs.io/en/stable/updating/):

```shell
cd my-project
uvx copier update -A -a .copier-answers-tf-task.yaml .
```

This synchronizes the files in your project that the template manages with the latest release of the template.

> Copier only changes the files and directories that are managed by the template.

### Using Variants

Use the variants feature to deploy extra copies of stacks for development and testing. Each variant of a stack uses the same configuration as other instances with the specified context.

Specify `TFT_VARIANT` to create a variant:

```shell
export TFT_CONTEXT=dev TFT_STACK=my-app TFT_VARIANT=feature1
task tft:plan
task tft:apply
```

The tooling automatically sets the value of the tfvar `variant` to match `TFT_VARIANT`. This ensures that every variant has a unique identifier that can be used in TF code.

Only set `TFT_VARIANT` when you want to create an alternate version of a stack. If you do not specify a variant name, TF uses the default workspace for state, and the value of the tfvar `variant` is `default`.

The [test](https://opentofu.org/docs/cli/commands/test/) feature of TF creates and then immediately destroys resources without storing the state. To ensure that temporary test copies of stacks do not conflict with other copies, the test in the stack template includes code to set the value of `variant` to a random string with the prefix `tt`.

### Using Local TF State

This tooling currently uses [remote state](https://opentofu.org/docs/language/state/remote/) by default. Set `TFT_REMOTE_BACKEND` to `false` to use [local TF state](https://opentofu.org/docs/language/settings/backends/local/):

```shell
TFT_REMOTE_BACKEND=false
```

If you use the default TF code for a stack, you will also need to comment out the `backend "s3" {}` block in the `main.tf` file.

> I highly recommend that you only use TF local state for prototyping. Local state means that the resources can only be managed from a computer that has access to the state files.

### Using OpenTofu

By default, this tooling uses Terraform. To use OpenTofu, set `TFT_CLI_EXE` as an environment variable, with the value `tofu`:

```shell
TFT_CLI_EXE=tofu
```

## Contributing

This project was built for my personal use. I will consider suggestions and Pull Requests, but I may decline anything that makes it less useful for my needs. You are welcome to fork this project.

Some of the configuration files for this project template are provided by my [project baseline Copier template](https://github.com/stuartellis/copier-sve-baseline). To synchronize a copy of this project template with the baseline template, run these commands:

```shell
cd tf-tasks
copier update -A -a .copier-answers-baseline.yaml .
```

## License

MIT © 2025 Stuart Ellis
