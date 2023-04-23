---
layout: post
title:  "Turbo button for SQL Server or How I (re)learned about MAXDOP"
date:   2023-04-22 09:02:21 +0100
categories: azure sql server
---

This story starts at the beginning of September when our main Azure SQL Server database started struggling, this was not unexpected given the new customers onboarded so we didn't think too much about it and implemented the go to solution of the cloud era and bumped the sku to a maximum of 32 cores.

This bought us some time but in early october we were back to noticeable performance issues, so we bumped it up the next level, 40 cores, which this time bought us less than two weeks before we had to go all the way to 80 cores.

80 is the maximum for the serverless mode, that is where you pay for what you use. Actually, the serverless model for sql server means that you pay for what you use but there is a minimum set of cores that you have to pay for, which increases as you increase the maximum, so that for 80 cores you are actually paying for at least 10 cores.

At this point the effort to improve database performance got maximum priority by looking at costly queries, code improvements, etc, which did very little to improve things. To be clear, there were improvements but these were not enough to eliminate performance issues being experienced by our customers.


A few weeks later I did a deep dive where I learned and relearned things about database performance, e.g. cost threshold for parallelism, which gave me an idea of new things to try.


And then I had a flash of ... well memory really. I remembered a conversation in the pub about a system that had done a staggered go live (10% users followed by remaining users) and how after they've completed the full rollout, the system performance cratered, which was solved by limiting max degree of parallelism aka [MAXDOP](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/configure-the-max-degree-of-parallelism-server-configuration-option?view=sql-server-ver16).


So I checked our database and sure enough MAXDOP was set to 0, i.e. unlimited.  We changed this to the recommended value, 8, and as if by magic cpu usage went down to about 10% and after a few days of running we were able to lower the upper limit of cpu cores down to 20 cores.