---
layout: post
title:  "Azure Functions Quirks or How I stop worrying and love AzureFunctionsWebHost__hostid"
date:   2024-06-17 20:37:24 +0100
categories: azure functions 
---

A few weeks back we added a staging slot to one of our functions and it promptly stopped working, which was a bit of a surprise to say the least. As this wasn't the first function for which we had added a staging slot.

It turns out that by design the function slot names, and remember that the main one is just the produciton slot, are truncated [sic.] to 32 characters so if your function's name is longer than 32 characters and you add a slot you will have issues.

It might be the new slot or it might be the production slot (aka the main function app) but one or many will stop.

Fear not, there is a solution:

Add the following environment variable to all your slots, production included

```
AzureFunctionsWebHost__hostid
```

For reasons that would take too long to explain we use this script as part of the deployment of the code. 

It would be preferable to do it in Terraform as it handles randomness without constant regenerations, but as I said it would take too long to explain.

```powershell
$pattern = "AzureFunctionsWebHost__hostid"

$prodId='fDHTRecVwLZWaoQ98ltBEiby4gArvqxS' # Generated with:  -join ((65..90) + (97..122) + (48..57) | Get-Random -Count 32 | ForEach-Object {[char]$_})

$stagingId='cOFdDfN6zb0lumXGgPHV3Wn4aLeZk5TS' # Generated with:  -join ((65..90) + (97..122) + (48..57) | Get-Random -Count 32 | ForEach-Object {[char]$_})

#prod slot

$settings = az functionapp config appsettings list -g $rg  -n $name | ConvertFrom-Json

$exists = $json | ConvertTo-Json -Depth 100 | Select-String -Pattern $pattern

if ($exists -eq $null){

    az functionapp config appsettings set -g $rg -n $name --slot-settings "AzureFunctionsWebHost__hostid=$prodId"

}

#staging slot

$settings = az functionapp config appsettings list -g $rg -n $name --slot staging | ConvertFrom-Json

$exists = $json | ConvertTo-Json -Depth 100 | Select-String -Pattern $pattern

if ($exists -eq $null){

    az functionapp config appsettings set -g $rg -n $name --slot staging --slot-settings "AzureFunctionsWebHost__hostid=$stagingId"

}
```
