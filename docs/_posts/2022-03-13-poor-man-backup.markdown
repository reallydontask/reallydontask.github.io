---
layout: post
title:  "Poor Man's Azure Backup"
date:   2022-03-13 11:37:24 +0100
categories: azure cloud 
---

Azure offers a backup service, which following Azure naming conventions is simply called [Azure Backup](https://azure.microsoft.com/en-gb/pricing/details/backup/#overview). There is nothing wrong with this service and I've used it in the past but I wondered if it would be possible to _backup_ directly to a storage account.

It turns out that it is possible.

My main use case is to backup photos from, my and my partner's google accounts, essentially via a regular/ad-hoc takeout, so one of the nice features of this approach is that you can mount the backup as a network drive in any PC/laptop thus making it very easy to share, the main downside is that unless Active Directory auth is used, it seems that only admin access is possible, which I appreciate won't work for everyone but it allows me to create a second share that my family uses and dump there a curate list of photos.


## Azure File Share

Azure Storage accounts offer file shares that can be mounted using SMB 3.0, this version is needed to enable encryption.

Once an Azure file share is mounted/mapped it just behaves like a regular network drive and it is possible to copy any files to that drive to, in effect, back it up.

This is not much of a backup but if we couple it with xcopy and a scheduled task/cronjob then we have the makings of a backup solution.

## Xcopy

The command I use is this:

```
xcopy.exe 'C:\Users\john\Documents\Photos\' 'Z:\Photos\' /M /E /G /H /Y
```
The flag descriptions are from the [official microsoft xcopy page](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/xcopy)

_/M_

Copies Source files that have their archive file attributes set. Unlike /a, /m turns off archive file attributes in the files that are specified in the source. 

For me this is the most important flag as it essentially makes xcopy behave a bit like a backup command.

/E

Copies all subdirectories, even if they are empty.

/G

Creates decrypted Destination files when the destination does not support encryption.

/H

Copies files with hidden and system file attributes.

/Y

Suppresses prompting to confirm that you want to overwrite an existing destination file.

## An Example

This script can be used to create a resource group, storage account and file share, then mount the file share as the X drive.

```
$rg = "Poor Mans Backup"
$location= "uk south"
$storageAccountName="pmb1983"
$shareName = "backup"

az group create -n $rg -l $location
az storage account create -g $rg -n $storageAccountName -l $location --min-tls-version TLS1_2 --sku Standard_GRS --https-only true
$connectionString=(az storage account show-connection-string --name $storageAccountName -g $rg -o tsv)
az storage share create --name $shareName --account-name $storageAccountName --connection-string $connectionString
$key1=$(az storage account keys list -g $rg -n $storageAccountName --query "[?keyName=='key1'].value" -o tsv)
$connectTestResult = Test-NetConnection -ComputerName $storageAccountName.file.core.windows.net -Port 445
if ($connectTestResult.TcpTestSucceeded) {
    # Save the password so the drive will persist on reboot
    cmd.exe /C "cmdkey /add:`"$storageAccountName.file.core.windows.net`" /user:`"localhost\$storageAccountName`" /pass:$key1"
    # Mount the drive
    New-PSDrive -Name X -PSProvider FileSystem -Root "\\$storageAccountName.file.core.windows.net\$shareName" -Persist
} else {
    Write-Error -Message "Unable to reach the Azure storage account via port 445. Check to make sure your organization or ISP is not blocking port 445, or use Azure P2S VPN, Azure S2S VPN, or Express Route to tunnel SMB traffic over a different port."
}
```