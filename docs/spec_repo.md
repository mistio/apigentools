# Spec Repo

A "Spec Repo" is a repository (or, more generally, a directory), which contains files necessary for apigentools to generate your client code. The layout is as follows:

```
.
├── .gitignore
├── config
│   ├── config.json                 # general config for apigentools
│   └── languages
│       └── java_v1.json            # openapi-generator config for Java client for v1 API
├── downstream-templates
│   └── java
│       └── LICENSE                 # a file/template to add to generated client
├── generated                       # generated code will end up in this directory
│   └── .gitkeep
├── spec
│   └── v1                          # directory with spec of v1 of the API
│       ├── accounts.yaml           # example: spec of the accounts API
│       ├── header.yaml             # header of the OpenAPI spec
│       └── shared.yaml             # entities shared between multiple parts of the OpenAPI spec
├── template-patches
│   └── java-01-minor-change.patch  # a patch to apply to openapi-generator template
└── templates                       # openapi-generator templates with applied patches end up here
    └── .gitkeep
```

You can create a basic scaffold of a Spec Repo using the `init` subcommand, for example:

```
apigentools init myspecrepo
```

Most of the paths in this layout can be overriden by commandline arguments passed to apigentools executable. More details about listed files follow.

## .gitignore

This is a standard [.gitignore file](https://git-scm.com/docs/gitignore).

## config/

This is a directory containing [config.json](#configconfigjson), which is a configuration file for apigentools.

## config/languages/

This directory contains openapi-generator configuration files. There must be one file for every language/major-API-version configuration configured in your `config/config.yaml`. For information on what keys can go in these files, use `openapi-generator config-help -g <language>`.

## downstream-templates/

This directory contains "downstream" (in the sense that they're provided by you, not openapi-generator) templates. These need to go into a per-language directories, e.g. `downstream-templates/java`. Usually, you would put files here like a top-level `README.md`, top-level `pom.xml` and similar.

## generated/

This directory contains code generated by apigentools, it's usually empty when you clone this repository, as the generated clients are supposed to have their own repositories and be git-ignored from the Spec Repo.

## spec/

This directory contains per-major-API-version OpenAPI specifications of your API. For example, if your API has `v1` and `v2`, the `spec` directory would have `v1` and `v2` subdirectories. Each of these would contain at least `header.yaml` and `shared.yaml` "section files". Refer to [section files reference](section-files) for more information.

## template-patches/

This directory contains your downstream template patches; these are applied to openapi-generator templates before the generation process using the `apigentools templates` command, thus allowing you to customize the upstream templates.

## templates/

This directory contains processed upstream templates, IOW templates from openapi-generator with your template patches applied on top. These are usually git-ignored from the Spec Repo and you generate them locally using the `apigentools templates` command.

# File Formats

This document serves as a reference of formats of various files used by apigentools.

## `config/config.json`

This is a configuration file for apigentools and it must be present in the Spec Repo for apigentools to work.

Example:

```
{
    "codegen_exec": "openapi-generator",
    "languages": {
        "java": {
            "github_org_name": "my-github-org",
            "github_repo_name": "my-java-client",
            "spec_versions": ["v1"],
            "version_path_template": "myapi_{{spec_version}}"
        }
    },
    "server_base_urls": {
        "v1": "https://api.myserver.com/v1"
    },
    "spec_sections": {
        "v1": ["accounts.yaml"]
    },
    "spec_versions": ["v1"]
}
```

The structure of the general config file is as follows (starting with top-level keys):

* `codegen_exec` - name of the executable of the code generating tool
* `languages` - settings for individual languages, contains a mapping of language names to their settings
  * individual language settings:
    * `commands` - commands to execute before/after code generation; commands are executed in two *phases* - `pre` (executed before code generation) or `post` (executed after code generation); note that each command is run once for each of the langauge's `spec_versions` and inside the directory with code for that spec version
       * *phase* commands - each phase can contain a list of commands to be executed; each command is represented as a map:
         * `description` - a description of the command
         * `commandline` - the command itself, the items in the list are strings and potentially *functions* that represent callbacks to the Python code; each function is represented as a map:
            * `function` - name of the function (see below for list of recognized functions)
            * `args` - list of args to pass to the function in Python code (as in `*args`)
            * `kwargs` - mapping of args to pass to the function in Python code (as in `**kwargs`)
            * result of the *function* call is then used in the actual commandline call
    * `command_env` - additional environment values with which both commands from `commands` and code generation itself will be executed (mapping of environment variable names to their values)
    * `github_org_name` - name of the Github organization of the client for this language
    * `github_repo_name` - name of the Github repository of the client for this language
    * `spec_versions` - list of spec versions to generate client modules for, has to be subset of top-level `spec_versions`
    * `upstream_templates_dir` - name of the directory in openapi-generator that holds templates for this language; this is optional and by default the name of the language is used
    * `version_path_template` - Mustache template for the name of the subdirectory in the Github repo where code for individual major versions of the API will end up, e.g. with `myapi_{{spec_version}}` as value and `github_repo_name` of value `my-java-client`, the code for `v1` of the API will end up in `myapi-java-client/myapi_v1`
* `server_base_urls` - mapping of major spec versions (these have to be in `spec_versions`) to URLs of servers that provide them
* `spec_sections` - mapping of major spec versions (these have to be in `spec_versions`) to lists of files with paths/tags/components definitions to merge when creating full spec (files not explicitly listed here will be ignored)
* `spec_versions` - list of major versions currently known and operated on (these have to be subdirectories of `spec` directory)

### Functions in Language Phase Commands

This section lists currently recognized functions for language phase commands as described in the section above:

* `glob` - runs Python's `glob.glob`

## Section Files

By design, apigentools doesn't operate on full OpenAPI specification files, but on so-called "section files". These are pretty much an exploded OpenAPI spec - this improves the development experience when working on these individual files instead of one huge file. The rules for these files are as follows:

* The below rules apply specification of every major API version.
* `spec/<MAJOR_API_VERSION>/header.yaml` is expected to be present. This is a header file that should only contain OpenAPI keys `openapi`, `info`, `externalDocs` (note that `servers` is filled in dynamically by apigentools based on your `config/config.yaml`).
* `spec/<MAJOR_API_VERSION>/shared.yaml` is expected to be present. This is a file containing any OpenAPI objects that are referenced from more than one "section" - `components` (which includes `schemas` and `security_schemes`), `security` and `tags`.
* Any other `.yaml` files with arbitrary `components`, `paths`, `security` and `tags` may be added (note that these have to be explicitly listed in `config/config.json` in order to be used).

On apigentools invocation, these are merged into a single OpenAPI spec file and used as input for further operations, e.g. for openapi-generator.