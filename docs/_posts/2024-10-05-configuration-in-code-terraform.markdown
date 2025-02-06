---
layout: post
title:  "Terraform: Configuration In Code"
date:   2024-10-05 20:37:24 +0100
categories: terraform
---

- [Introduction](#introduction)
- [Rationale](#rationale)
- [Configuration in Code](#configuration-in-code)
- [Advantages](#advantages)
- [Disadvantages](#disadvantages)



## Introduction

In this post I'd like to introduce a configuration pattern for terraform that we've called Configuration In Code, I couldn't use Configuration As Code, right?

## Rationale and Background

We have quite a lot of environments that are **mostly** the same, however unlike in other places I've worked the differences are not necessarily the obvious ones, e.g. use different SKUs in prod or ignore region resilience requirements in non production but are actually a bit more subtle, e.g. function app X is needed in dev, qa1 and prod1 but not qa2 or prod2 or prod3 o servicebus namespace is needed only dev/qa2/prod2.

As the configuration is/was managed, by and large, in a catch all terraform.tfvars (though we have some specific tfvars, e.g. entra or aks) adding cloud resources to the correct environments only was tedious, as we needed to add the same terraform object up to 10 times (more if we include the failover environments), and error prone, as we often ended up with resources not being added to all needed environments, which was a combination of how the resources were requested (piecemeal) and how we promoted stuff into production.

## Configuration in Code

The idea is pretty simple, move your tfvars configuration into a terraform file, within reason, and toggle configuration on/off with the env variable

This is an example of the config for functions that are deployed to aks.  We then loop through these to create the azure resources, e.g. app insights, storage accounts, etc ...


```hcl
locals {
    aks_functions = merge(
    contains(["dev", "qa", "uksprod1"], var.env) ? { clientlookup = {} } : {},
    contains(["dev", "qa", "uksprod1"], var.env) ? { client = { create_app_insights = true } } : {},
    contains(["dev", "qa", "uksprod1"], var.env) ? { provider = { create_app_insights = true, storage_account_name = "mgrprov" } } : {},
    contains(["dev", "qa2", "uksprod2"], var.env) ? { operator = { create_app_insights = true, storage_account_name = "mgrop" } } : {},
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

```
locals {

    secrets = concat(
    ["JobUserSql-Password"],
    local.is_regional ? ["ServiceBus-ConnectionString"] : [],
    contains(["dev", "qa", "uksprod1"], var.env) ? ["KEDA-ConnectionString"] : []
  )
}
```

## When to Use


## When not to use

Unfortuntely, there are downsides to this approach that it's probably worth bearing in mind.

Firstly, there is probably little need if your environments are the same apart from sku changes.

I think the most annoying one is the lack of defaults in the locals, which can be normally be remedied by using try in the resource/module call but sometimes this is not possible.
