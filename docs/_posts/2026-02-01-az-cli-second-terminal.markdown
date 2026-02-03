---
layout: post
title:  "Using multiple accounts with az cli"
date:   2026-02-01 11:37:24 +0100
categories: azure cli
---

Unlike aws cli, az cli does not have the concept of a profile, namely you can't do

```bash
az storage account list --profile bob
```

instead you are forced to do something like this


```bash
export AZURE_CONFIG_DIR=~/.tmpaccount

az login 
```

You can add this as function to your `.bashrc` file or your favourite's shell configuration as a function, e.g.

```bash
function c_az_prod {
 
        export AZURE_CONFIG_DIR=~/.prod
        az login
        az account set -s 40e300f1-dead-c0de-beef-d6295431e6d6
 
}
```