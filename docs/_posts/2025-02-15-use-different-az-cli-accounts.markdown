---
layout: post
title:  "Using different Azure Accounts in AZ CLI"
date:   2025-02-15 17:07:24 +0100
categories: azure terraform az-cli
---

Every so often I find myself needing to switch azure accounts and it can be a bit painful as the logged in account changes for ALL terminal windows

It turns out there is a very easy way of ensuring that each terminal gets its own account

In my case, I have two accounts:  prod and nonprod

My default account is nonprod but occassionally I do need the prod one so I've got this little function in my bash profile

```bash
function az_prod_login (){

export AZURE_CONFIG_DIR=~/.tmpprod
az login
az account set --subscription "<my prod subscription id>"
}

```

In theory you could also do something like this if you have want a unique directory each time you login.

```bash
function az_prod_login (){
RANDOM_STRING=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 8 | head -n 1)
export AZURE_CONFIG_DIR=~/.tmp_$RANDOM_STRING
az login
az account set --subscription "<my prod subscription id>"
}

```

You could take it up one level by passing the username or the subscription.

Left as an exercise for the reader or chatGTP 😉