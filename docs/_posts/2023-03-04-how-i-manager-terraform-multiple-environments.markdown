---
layout: post
title:  "How I manage terraform code for multiple environments"
date:   2023-04-03 11:06:24 +0100
categories: terraform
---

TL;DR Using symlinks

If you have multiple environments for your infrastructure there are different ways you can handle them:

- branches
- workspaces
- terragrunt
- symbolic links

There is a [post from gruntwork](https://blog.gruntwork.io/how-to-manage-multiple-environments-with-terraform-32c7bc5d692) about the pros and cons of the first three methods, and perhaps, unsurprisingly they conclude that terragrunt is the best option.

While I appreciate that terragrunt can help with much more than just sharing code for multiple environments and as it happens I currently use it due to Windows woeful support for symbolic links, I still like using symbolic links mainly because it seems to run faster than terragrunt and also as it's much easier to make changes in just a single environment that go beyond config.  To be clear, I'm not saying this is not doable with terragrunt just that it's easier to do with symbolic links. 

Anyway, without further ado, this is what I normally do:


└── terraform
    ├── environments
    │   ├── dev
    │   │   ├── main.tf -> ../../main.tf    
    │   │   ├── terraform.tfvars
    │   │   ├── terraform_config.tf    
    │   │   └── variables.tf -> ../../variables.tf
    |   ├── staging
    │   │   ├── main.tf -> ../../main.tf    
    │   │   ├── terraform.tfvars
    │   │   ├── terraform_config.tf    
    │   │   └── variables.tf -> ../../variables.tf
    │   └── prod
    │       ├── main.tf -> ../../main.tf
    │       ├── terraform.tfvars
    │       ├── terraform_config.tf    
    │       └── variables.tf -> ../../variables.tf
    ├── modules
    │   ├── consumer
    |   └── producer
    ├── main.tf    
    └── variables.tf

The code is written once in the terraform directory and then by virtue of the symlinks it's available for all environments at once. If you need to add a new file, you will need to add the symlinks to all environments or all environments that you want, e.g. maybe you have a need for an integration server only in staging and prod.

terraform_config.tf contains the backend stuff as well as the provider configuration.
terraform.tfvars contains environment specific values for the variables.

This setup is very flexible when it comes to environments deviating from the others or when experiments are needed. In which case, you can delete the symlink, copy the code file from terraform directory and make the changes only in the relevant environment(s), without affecting the rest of the environments.

You can do this with terragrunt but if the code you don't want everywhere is not in a module, you are going to have to add feature flags everywhere, which might mess up the references as they might be arrays rather than a single. Obviously this is manageable but if you can avoid it, why go through it?


As I said previously, there are other good reasons to use terragrunt but if you are solely using it to share code across environments and are on a *nix based system, consider using symbolic links instead.