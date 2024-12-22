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

## Rationale

We have quite a lot of environments that are **mostly** the same, however unlike in other places I've worked the differences are not necessarily the obvious ones, e.g. use different SKUs in prod or ignore region resilience requirements in non production but are actually a bit more subtle, e.g. function app X is needed in dev, qa1 and prod1 but not qa2 or prod2 or prod3 o servicebus namespace is needed only dev/qa2/prod2.

As the configuration is/was managed, by and large, in a catch all terraform.tfvars adding cloud resources that are not needed across all environments was tedious, as we needed to add the same terraform object up to 10 times (more if we include the failover environments), and error prone, as we often ended up with resources not being added to all needed environments, which was a combination of how the resources were requested (piecemeal) and how we promoted stuff into production.

## Configuration in Code


## Advantages

## Disadvantages
