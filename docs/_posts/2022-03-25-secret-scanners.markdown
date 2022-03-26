---
layout: post
title:  "Secret Scanning Tools"
date:   2022-03-25 08:37:14 +0100
categories: security
---

The Ministry of Justice has a [code in the open policy](https://mojdigital.blog.gov.uk/2017/02/21/why-we-code-in-the-open/), which for various reasons our team was not adhering too, so recently I scanned our repositories with various open source and commercial tools to try to find out whether we'd let any secrets slip or not.

When I set out to scan the repositories I was aware of two issues:

 - A very obvious [hardcoded password](https://github.com/ministryofjustice/staff-infrastructure-azure-landing-zone/blob/233df6fa3192a06d2945f595fced8d581327bd3d/terraform/environments/pullrequest/terraform-azurerm-vmstopstart/main.tf#L84) however this isn't a massive issue as the VMs are ephemeral and not accessible from the internet. In any case this is good test case for the tools

- A Teams webhook in a tfvars file. This is a less obvious test case as it's mostly composed of guids.


As part of the scanning we also found a powershell script that had a storage account key hardcoded that had been deleted so it is in an old commit.

The process for selecting the tools was relatively simple, somebody in the security team suggested a bunch of tools that we could try so I tried a few of them, not all, for instance couldn't try github secret scanning as my org doesn't allow it .

## Results

I must admit that I was disappointed with the results found, summarized in the table below:


|Tool| Commercial | hardcoded password? |  Teams Webhook? | Storage Account Key? | command |
|----|------------|--------------------|----------------|---------------------| -------|
|[TruffleHog](https://github.com/trufflesecurity/truffleHog)| No | No |  No | No |```trufflehog --regex file://<full path to repo>``` |
|[repo-security-scanner](https://github.com/techjacker/repo-security-scanner)| No | No |  No | No |```git log -p \| scanrepo ``` |
|[gittyleaks](https://github.com/kootenpv/gittyleaks)| No | Yes |  No | No |```gittyleaks --find-anything``` |
|[detect-secrets](https://github.com/Yelp/detect-secrets)| No | Yes |  No | No |```detect-secrets scan``` |
|[shhgit](https://github.com/eth0izzle/shhgit)| No | No |  No | No |```./shhgit --local <path to repo>``` |
|[gitGuardian](https://gitguardian.com)| Yes | No |  Yes | Yes |```This required the installation of the GitGuardian app in Github and grant read access to the relevant repositories``` |
|[spectralOps](https://spectralops.io)| Yes | Yes |  Yes | No |```spectral scan --include-tags base,audit``` |


A lot of the tools seem to do very basic checks, which will help you to recover from obvious mistakes that should've been spotted in a PR process. I should say that I'm not disparaging the tools, these mistakes happen all the time, so they do provide some value.


## TruffleHog

This tool was extremely dissappointing as it just found High Entropy items on the readme files and a random arm template, which for reasons unknown to me contain a png file.  It found nothing on the other repos of consequence.

## Repo-Security-Scanner

Another disappointing tool, it just seems to just match file names, so we get a lot matches for terraform.tfvars and some hits around keywords, e.g. backup but overall this was not great.

## Gittyleaks

This tool does find the password and it also matches a lot of keywords like key or token in the code, which are largely false positives in our case but it doesn't find the other two issues (Webhook and storage account key)

## Detect-Secrets

This tool does find the password and it also finds a key that I didn't even know was there, ASPNETCORE_AUTO_RELOAD_WS_KEY. This is actually commented out and I'm not 100% sure what it does, though I can speculate.  Google is not very helpful.

Unfortunately, it doesn't find the teams webhook nor the storage account key, though I don't think this scans the log

## shhgit

Another disappointing tool, it just seems to find terraform.tfvars

## Git Guardian

This is a paid tool, so you get a nice frontend and integration with things like Slack, anyway. I'm disappointed that it hasn't found the password but pleasently surprised that it has found the Teams webhook and this is how I learned about the storage account key.

## SpectralOps

This is another paid tool, similar to git guardian. Instead of installing the github app, I have used their CLI tool, which works really well and will post the results to their portal.

It's disappointing that it didn't catch the storage account key even when running on history mode.

It does provide quite a lot of guidance, which is good but I guess it could also become annoying, so for instance it classed as an error that we're using the latest version of a custom github action, which I think it should be a warning, specially as in this case it's from the same organization as the repos that are being scanned.

Overall, I think this has so far been my favourite tool, but it's very early days.