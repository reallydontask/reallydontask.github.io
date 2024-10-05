---
layout: post
title:  "Terraform: Configuration In Code"
date:   2024-10-05 20:37:24 +0100
categories: terraform
---

## Introduction

In this post I'd like to introduce a configuration pattern for terraform that we've called Configuration In Code, I couldn't use Configuration As Code, right?

## Rationale

If you have many environments, you have probably found yourself copying and pasting configuration from main terraform.tfvars file into new environments and you have also probably forgot to copy some of these across environments.

This is tedious and can be error prone, as the environments are similar enough but not equal, e.g. prod environments will have different SKUs or will likely be the only environments with failover environments.