---
layout: post
title:  "Oracle Cloud Adapter SDK - Part 4: Actually inserting documents"
tags:
- oracle
- cloud adapter
- cloud adapter sdk
- soa suite
- osb
- mongodb
disqus: true
---
## Introduction

I visited the Oracle FMW Summercamp in Lissabon last year and I really wanted to build a Cloud Adapter using the Oracle Cloud Adapter SDK. During the Summercamp I already started, but after I ran into some errors I quit. Now I am starting again.

I want to build a Cloud Adapter for MongoDB. Perhaps Oracle or another party may also build one, but this is for my personal experience and after doing an e-learning and associate certification for MongoDB (which was really challenging!) I want to try it out for myself.

This is the first part of a small series of blogs regarding my effort and will cover the installation proces. Niall Commiskey already has a great [blog post](http://niallcblogs.blogspot.nl/2015/06/408-first-steps-with-cloud-adapter-sdk.html) about installing the Cloud Adapters, however I ran into several problems so I decided to write a post of my own describing my installation steps. I will use Windows 10 and PowerShell (my favourite Windows shell!) for these blogs.  All sources can be found on [GitHub](https://github.com/ninckblokje/MongoDBCloudAdapter).

Parts:

- [Part 1: Installation]({% post_url 2016-03-15-cloud-adapter-sdk-part1 %})
- [Part 2: Functionality]({% post_url 2016-03-29-cloud-adapter-sdk-part2 %})
- [Part 3: Building the wizard]({% post_url 2016-06-26-cloud-adapter-sdk-part3 %})
- Part 4: Actually inserting documents

Just a small warning: Always keep track of Oracle license information and the Oracle certification matrix!

## Domain configuration

The Cloud Adapter will be stored in this directory: **$ORACLE_HOME/soa/soa/modules/oracle.cloud.adapter_12.1.3**.

Several security settings need to be applied to the domain for the Cloud Adapter to access the credentials. The [Oracle documentation](http://docs.oracle.com/middleware/1213/cloudadapter-rightnow/OCAIG/GUID-57BCEAD7-F1A7-4EC8-A93B-CD05A157934C.htm) describes it, though I did not need to deploy the file **cloudsdk.ear**. I needed to configure the ones described in the documentation, but also one more:

![Extra permissions]({{ site.github.url }}/assets/cloud-adapter-sdk-part4/extra_permissions.png)

I needed to give my own classes access to the credentials to work around some Oracle code which presumes it receives a username password type of credentials, where I need something else. Add a new grant:

- Click on **Create...**
- Use as codebase: **file:${soa.oracle.home}/soa/modules/oracle.cloud.adapter_12.1.3/nl.syntouch.oracle.adapter.cloud.mongodb.jar**

## Flow

## Accessing credentials

## Input & Output

## Refactoring

So it has been a while and I had wanted to refactor some code for better reuse in both the wizard and the actual adapter. I found out that the logger used by Oracle in the wizard is a different one from the logger used in the adapter. So I had to write some wrapper code around that in the form of the class **LoggerServiceWrapper**.

Further the connection code is now shared between the wizard and the adapter. And the BSON code is now better separated from the rest of the code.


