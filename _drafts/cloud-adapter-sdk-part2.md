---
layout: post
title:  "Oracle Cloud Adapter SDK - Part 2: Functionality"
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
So this is part 2 of a series of blog post describing building a Cloud Adapter for MongoDB. In this part I want to discuss the functionality I want to achieve.

I will use Windows 10 and PowerShell (my favourite Windows shell!) for these blogs.

Parts:

- [Part 1: Installation]({% post_url 2016-03-15-cloud-adapter-sdk-part1 %})
- Part 2: Functionality
- Part 3
- Who knows!

Just a small warning: Always keep track of Oracle license information and the Oracle certification matrix!

## What should it do?
MongoDB has a lot of features we might want to expose in the Cloud Adapter. However I want to start relative simple and I might expend the functionality in the feature. So I want to start with inserting data. A second step will be to find the data by querying it.

## MongoDB
MongoDB is a NoSQL database and stores documents. These documents are basically JSON documents (actually BSON) and can have a maximum size of 16 megabytes. Basically there are no foreign keys. Of course you refer to other documents, however these is no foreign key like in a relational database. Each document does have a primary key called **ObjectId**. A document is stored in a collection and a MongoDB database can have multiple collections. A document is also never structured like in a relational database. This means that basically each field could be empty or may not even exists. New fields can also be added when ever needed by simply updating the document.

MongoDB supports basic querying and advanced aggregation features. Indexes can be used to speed up queries and aggregations and MongoDB comes with a sophisticated query analyzer.

There is no support for transactions in MongoDB. So the data is either stored or not. Multiple inserts or other modifications will be made visible immediatly and there is no rollback feature. Therefor any application must expect to see an inconsistent state at any given time and any application must be made robust. A good document design will minimize the need for transactions.

MongoDB supports replication (for backup) and sharding for scalability. In case of a shard a predefined field will determine to which member of a shard the data goes. Special config servers maintain the locations of each document in the shard cluster.

MongoDB offers several configuration options which can increase or decrease potential data reliability.

## MongoDB API
MongoDB offers a Javascript commandline tool (like SQLPlus from Oracle) for working with a MongoDB database instance. Simple commands or complete scripts written in Javascript can be executed here.

