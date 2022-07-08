---
layout: post
title:  "Validate Azure Pipelines with pre-commit hook"
date:   2022-06-07 14:06:24 +0100
categories: azurepipelines azuredevops
---

One of the most downsides of storing your pipelines in a repository is that for most CI/CD platforms you need to commit the code to the repository in order to run it. This isn't generally a massive issue but when you are trying new things or are unsure of the syntax, it can be really frustrating as the feedback cycle is rather longer than say running a linter locally.

If you are using Github Actions, you can use [act](https://github.com/nektos/act) but as far as I am aware there isn't anything like this for Azure Pipelines, enter the Azure Pipelines REST API and [preview](https://docs.microsoft.com/en-us/rest/api/azure/devops/pipelines/preview/preview?view=azure-devops-rest-7.1).

This method can be used to validate the yaml by using the _yamlOverride_ parameter and passing in the new yaml, if this is combined with the pre-commit git hook, we have a seamless ish experience that will catch syntax issues on the pipeline definitions and any templates the pipeline might use, which while obviously not as good as [act](https://github.com/nektos/act) it's an improvement over the commit, push, run pipeline workflow.


The pre-commit hook, can be found in [this gist](https://gist.github.com/reallydontask/7e7e01695b13635946dafe77c65fab01#file-pre-commit)

This hook works by firing a request at the pipeline preview endpoint of the Azure DEVOPS REST API and it relies on having an existing [configuration file](#pipelines-configuration-file), which can be generated using the [pre-commit-pipelines.ps1](https://gist.github.com/reallydontask/7e7e01695b13635946dafe77c65fab01#file-pre-commit-pipelines-ps1) script, see [instructions](#pipelines-configuration-file).

The validation will stop when it finds an error, which I think is the most likely use case, namely modifying a single pipeline at the time, however if you modify two or more it will stop at the first failure, thus if you have say two issues, in two different pipelines it will only flag the first issue and then once you fix it, it will flag the second one.

Finally, note that if you've not set an environment variable as per step 3 [below](#installation), the hook will not do anything.


## Installation

Pre-Requisites

- WSL Installed
- Powershell Core Installed in WSL
- .githooks directory exists

These are the steps needed to configure it.

1. Generate PAT in Azure Devops, see instructions [here](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows)
2. Ensure that you only allow Read Permission on the build scope. 

![Permissions](/assets/2022-07-06-azure_pat_permissions.png)

3. Create AZDO_PAT variable in your .bashrc file

Add this to the bottom of your .bashrc file

```bash
export AZDO_PAT=pat generated in step 1
```
4. Limit Permissions for .bashrc, 

```bash
chmod 600 ~/.bashrc
```

5. Configure git to use .githooks path for hooks

```bash
git config core.hooksPath .githooks
```

6. Create [pre-commit file](https://gist.github.com/reallydontask/7e7e01695b13635946dafe77c65fab01#file-pre-commit) with no extension in .githooks directory.

7. Run [pre-commit-pipelines.ps1](https://gist.github.com/reallydontask/7e7e01695b13635946dafe77c65fab01#file-pre-commit-pipelines-ps1) to populate the config file

```pwsh
pre-commit-pipelines.ps1 -configFile "<path to .githooks>/pipelines.csv"
```

Note that pre-commit expects the config file to be located in .githooks and named pipelines.csv


## Pipelines Configuration File

 The [pre-commit-pipelines.ps1](https://gist.github.com/reallydontask/7e7e01695b13635946dafe77c65fab01#file-pre-commit-pipelines-ps1) script can be used to create a configuration file that lists all the pipelines and files, note that because I'm too lazy it lists all pipelines in the Azure DevOps Organization, which might not be desirable, e.g. you might only want to list the ones in your own project.

 Anyway, run the pre-commit-pipelines.ps1 script from the scripts directory, this will generate the new configuration file if you've set up a token as dicussed in the [installation section](#installation)

 The old file will be stored as pipelines.csv.bak and you can add the following line to your .gitignore file to prevent it from being committed:
```
 *.csv.bak
```