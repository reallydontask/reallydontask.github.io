---
layout: post
title:  "TODOmentation. The practical alternative to documentation"
date:   2024-01-27 01:37:24 +0100
categories: best-practices
---

One of the favourite parts of my job is listening to Engineering leaders talk about how they are in an impossible bind with TODOmendation:

| Implementation is too hard

said one leader how had not baulked at migrating 2000+ servers from on-prem to cloud and then to kubernetes in the space of 3 years, who felt his company was missing out by not implementing TODOmentation

| Implementation will take too long

said another whose department had successfully finished a 7 year project and literally could not wait for TODOmentation.

| when are we going to realize any ROI?

asked the CTO of small company concerned about pouring his company's limited Engineering resources into TODOmentation.


In this article we will cover:

1. What is TODOmentation?
2. Pros and Cons of TODOmentation
3. How to implement TODOmentation 
4. Implementation Pitfalls to avoid
5. Closing Remarks

So let's get started


## 1. What is TODOmendation?

At its heart TODOmendation is simple but very powerful concept, namely:  

| Add comments to your code in-situ detailing any changes that the code might need.

That's it, no more, no less.

You will notice that there is no mention of TODO though I've yet to seen a successful implementation that doesn't use TODO as the preface to each entry, even in projects where English wasn't the spoken language!.

## 2. Pros and Cons of TODOmentation

I would like to take on the Cons first because there really is one big one, namely, lack of buy in to new initiatives.

We've all been there