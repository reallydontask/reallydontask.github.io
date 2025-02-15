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

In this post I'd like to introduce a configuration pattern for terraform that we've called Configuration In Code, I couldn't use Configuration As Code, right?

## Rationale and Background

We have quite a lot of environments that are **mostly** the same, however unlike in other places I've worked the differences are not necessarily the obvious ones, e.g. use different SKUs in prod or ignore region resilience requirements in non production but are actually a bit more subtle, e.g. function app X is needed in dev, qa1 and prod1 but not qa2 or prod2 or prod3 or servicebus namespace is needed only in dev/qa2/prod2.

As the configuration is/was managed in a catch all terraform.tfvars (though we do have some more thematic tfvars, e.g. entra or aks) adding cloud resources to the correct environments was tedious, as we needed to add the same tfvars ect a lot of times, and error prone, as we often ended up with resources not being added to all needed environments or more often, the opposite problem of adding resources where they were not needed. This is to say nothing of the failover environments.

## Configuration in Code

The idea behind Configuration in Code (CiC, pronounced ``/sɪk/``) is pretty simple, move your tfvars configuration into a terraform file as a local set of objects and toggle the objects on/off based on the environment name, ``var.env`` in our case, variable.

To be clear, this doesn't mean a complete elimination of the use of tfvars, at the very least we need a single variable to differentiate between environments but in practice there are a few other variables that don't really fit this pattern, e.g. vnet  ip ranges.

Having said that, sometimes you might want to choose not to use this pattern because the environment differences are too complex or time consuming to capture.

So how does this work in practice?  Well, below is an example of the config for functions that are deployed to aks.  We then loop through these to create the azure resources, e.g. app insights, storage accounts, etc ...


```hcl
locals {
    aks_functions = merge(
    contains(["dev", "qa", "prod1"], var.env) ? { clientlookup = {} } : {},
    contains(["dev", "qa", "prod1"], var.env) ? { client = { create_app_insights = true } } : {},
    contains(["dev", "qa", "prod1"], var.env) ? { provider = { create_app_insights = true, storage_account_name = "provider" } } : {},
    contains(["dev", "qa2", "prod2"], var.env) ? { operator = { create_app_insights = true, storage_account_name = "operator" } } : {},
  )
}
```

This approach also works to control things like the database SKU or whether a database is created for a particular environment (Registration Database is a in separate server in non dev environments)

```hcl
locals {
  database = merge({
    Adaptor = {
      sku_tier = contains(["dev", "qa2"], var.env) ? "Basic" : "S0"
      db_size  = 2
    },
    Session = {
      sku_tier = contains(["dev", "qa2"], var.env) ? "Basic" : "S0"
      db_size  = 2
    }    },
    contains(["dev"], var.env) ? {
      Registration = {
        sku_tier = "Basic"
        db_size  = 2
      }
  } : {})
}
```

This approach is not limited to maps/objects, it works also for lists/tuples. **is_regional** is a type of environment and is used instead of the more common: ```contains(["dev"], var.env)```

```hcl
locals {
    secrets = concat(
    ["JobUserSql-Password"],
    local.is_regional ? ["ServiceBus-ConnectionString"] : [],
    contains(["dev", "qa", "prod1"], var.env) ? ["KEDA-ConnectionString"] : []
  )
}
```

You can also have individual values for a each environments by using a map like this:

```
locals {

database_sku = {
  prod1 = {sku = "S0"}
  prod1 = {sku = "S0"}
  prod3 = {sku = "S2", size = 250}
}

database = merge({
  Adaptor = {
      sku_tier = try(local.database_sku[var.env].sku, "Basic")
      db_size  = try(local.database_sku[var.env].size, 2)
    },
  Session = {
      sku_tier = try(local.database_sku[var.env], "Basic")
      db_size  = try(local.database_sku[var.env].size, 2)
    }    },
  contains(["dev"], var.env) ? {
      Registration = {
        sku_tier = try(local.database_sku[var.env], "Basic")
       db_size  = try(local.database_sku[var.env].size, 2)
      }
  } : {})

}

```

It's probably worth emphasizing the fact that we have not got rid of our tfvars files, we've just massively reduced their scope and removed the obvious repetition in them.

A full sample can be found in this [gist](https://gist.github.com/reallydontask/a530d642b36027497857cb5ee9b6013d)

## Is this for Me?

If you have a lot of environments that are subtly different then you could benefit from this approach. This could also be really useful if the number of environments is increasing, 
e.g. you've finally decided that resilience is important or your company is expanding or you have one environment per customer.

If on the other hand, your environments mostly differ by SKU then this approach probably makes little sense as your code will probably just need to take in a different SKU right?, right?

I think the most annoying facet of this approach, is the lack of defaults in the locals versus variables. This can normally be remedied by using try in the resource/module call but sometimes this is not possible, e.g. using an object property in a for_each.
