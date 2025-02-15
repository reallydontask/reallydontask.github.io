---
layout: post
title:  "Terraform: Configuration In Code"
date:   2024-10-05 20:37:24 +0100
categories: terraform
---

- [Introduction](#introduction)
- [Rationale](#rationale)
- [Configuration in Code](#configuration-in-code)
- [Is this for Me?](#is-this-for-me)


## Introduction

In this post, I’ll introduce a Terraform configuration pattern we’ve dubbed **Configuration In Code** (CiC). While "Configuration As Code" might sound more familiar, CiC offers a unique approach to managing complex environments with subtle differences.

## Rationale and Background

We manage numerous environments that are **mostly** similar, but unlike other places where I’ve worked, the differences aren’t always obvious. For example:

- Function App X might be needed in `dev`, `qa1`, and `prod1`, but not in `qa2`, `prod2`, or `prod3`.
- A Service Bus namespace might only be required in `dev`, `qa2`, and `prod2`.

These subtle variations make managing configurations tedious, as we needed to add the same tfvars ect a lot of times, and error prone, as we often ended up with resources not being added to all needed environments or more often, the opposite problem of adding resources where they were not needed. This is to say nothing of the failover environments.


## Configuration in Code

The idea behind Configuration in Code (CiC, pronounced ``/sɪk/``) is pretty simple, move your tfvars configuration into a terraform file as a local set of objects and toggle the objects on/off based on the environment name, ``var.env`` in our case, variable.

To be clear, this doesn't mean a complete elimination of the use of tfvars, at the very least we need a single variable to differentiate between environments but in practice there are a few other variables that don't really fit this pattern, e.g. vnet  ip ranges.

Having said that, sometimes you might want to choose not to use this pattern because the environment differences are too complex or time consuming to capture.

So how does this work in practice?  Well, below is an example of the config for functions that are deployed to aks.  We then loop through these to create the azure resources, e.g. app insights, storage accounts, etc ...


```hcl
locals {
  aks_functions = merge(
    # Enable clientlookup function in dev, qa, and prod1
    contains(["dev", "qa", "prod1"], var.env) ? { clientlookup = {} } : {},
    
    # Enable client function in dev, qa, and prod1 with App Insights
    contains(["dev", "qa", "prod1"], var.env) ? { client = { create_app_insights = true } } : {},
    
    # Enable provider function in dev, qa, and prod1 with App Insights and custom storage account
    contains(["dev", "qa", "prod1"], var.env) ? { provider = { create_app_insights = true, storage_account_name = "provider" } } : {},
    
    # Enable operator function in dev, qa2, and prod2 with App Insights and custom storage account
    contains(["dev", "qa2", "prod2"], var.env) ? { operator = { create_app_insights = true, storage_account_name = "operator" } } : {},
  )
}
```

This approach also works to control things like the database SKU or whether a database is created for a particular environment (Registration Database is a in separate server in non dev environments)

```hcl
locals {
  database = merge(
    {
      Adaptor = {
        sku_tier = contains(["dev", "qa2"], var.env) ? "Basic" : "S0"
        db_size  = 2
      },
      Session = {
        sku_tier = contains(["dev", "qa2"], var.env) ? "Basic" : "S0"
        db_size  = 2
      }
    },
    # Enable Registration database only in dev
    contains(["dev"], var.env) ? {
      Registration = {
        sku_tier = "Basic"
        db_size  = 2
      }
    } : {}
  )
}
```

This approach is not limited to maps/objects, it works also for lists/tuples. The **is_regional** local represents a type of environment and is used instead of the more common: ```contains(["dev"], var.env)```

```hcl
locals {
  secrets = concat(
    # Common secret for all environments
    ["JobUserSql-Password"],
    
    # Regional-specific secret
    local.is_regional ? ["ServiceBus-ConnectionString"] : [],
    
    # Environment-specific secret for dev, qa, and prod1
    contains(["dev", "qa", "prod1"], var.env) ? ["KEDA-ConnectionString"] : []
  )
}
```

You can also have individual values for a each environments by using a map like this:

```
locals {
  database_sku = {
    prod1 = { sku = "S0", size = 2 },
    prod2 = { sku = "S0", size = 2 },
    prod3 = { sku = "S2", size = 250 }
  }

  database = merge(
    {
      Adaptor = {
        sku_tier = try(local.database_sku[var.env].sku, "Basic")
        db_size  = try(local.database_sku[var.env].size, 2)
      },
      Session = {
        sku_tier = try(local.database_sku[var.env].sku, "Basic")
        db_size  = try(local.database_sku[var.env].size, 2)
      }
    },
    # Enable Registration database only in dev
    contains(["dev"], var.env) ? {
      Registration = {
        sku_tier = try(local.database_sku[var.env].sku, "Basic")
        db_size  = try(local.database_sku[var.env].size, 2)
      }
    } : {}
  )
}

```

It's probably worth emphasizing the fact that we have not got rid of our tfvars files, we've just massively reduced their scope and removed the obvious repetition in them.

A full sample can be found in this [gist](https://gist.github.com/reallydontask/a530d642b36027497857cb5ee9b6013d)

## Is this for Me?

If you manage multiple environments with subtle differences, **Configuration In Code** (CiC) can significantly streamline your workflow. It’s particularly useful when:
- The number of environments is growing (e.g., for resilience or customer-specific setups).
- Environment-specific configurations are complex and error-prone.

If on the other hand, your environments mostly differ by SKU then this approach probably makes little sense as your code will probably just need to take in a different SKU right?, right?

I think the most annoying facet of this approach, is the lack of defaults in the locals versus variables. This can normally be remedied by using try in the resource/module call but sometimes this is not possible, e.g. using an object property in a for_each.
