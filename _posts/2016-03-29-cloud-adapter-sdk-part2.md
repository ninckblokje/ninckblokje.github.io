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

I will use Windows 10 and PowerShell (my favourite Windows shell!) for these blogs. All sources can be found on [GitHub](https://github.com/ninckblokje/MongoDBCloudAdapter).

Parts:

- [Part 1: Installation]({% post_url 2016-03-15-cloud-adapter-sdk-part1 %})
- Part 2: Functionality
- [Part 3: Building the wizard]({% post_url 2016-06-26-cloud-adapter-sdk-part3 %})
- [Part 4: Actually inserting documents]({% post_url 2016-11-13-cloud-adapter-sdk-part4 %})

Just a small warning: Always keep track of Oracle license information and the Oracle certification matrix!

## What should it do?
MongoDB has a lot of features we might want to expose in the Cloud Adapter. However I want to start relative simple and I might expend the functionality in the feature. So I want to start with inserting data. A second step will be to find the data by querying it.

## MongoDB
MongoDB is a NoSQL database and stores documents. These documents are basically JSON documents (actually BSON):

    {
    	"_id" : ObjectId("56fa75781f1378215c215709"),
    	"field1" : "value1"
    }

Basically there are no foreign keys. Of course you refer to other documents, however these is no foreign key like in a relational database. Each document does have a primary key called **_id** (which is of type **ObjectId**). A document is stored in a collection and a MongoDB database can have multiple collections. A single instance of MongoDB can host multiple databases.

A document is never structured like in a relational database. This means that basically each field could be empty or may not even exists. New fields can also be added when ever needed by simply updating the document. A document can have a maximum size of 16 megabytes.

MongoDB supports [basic querying](https://docs.mongodb.org/manual/core/read-operations-introduction/) and [advanced aggregation](https://docs.mongodb.org/manual/aggregation/) features. Indexes can be used to speed up queries and aggregations and MongoDB comes with a sophisticated query analyzer.

There is no support for transactions in MongoDB. So the data is either stored or not. Multiple inserts or other modifications will be made visible immediatly and there is no rollback feature. Therefor any application must expect to see an inconsistent state at any given time and any application must be made robust. A good document design will minimize the need for transactions.

MongoDB supports [replication](https://docs.mongodb.org/manual/replication/) (for backup) and [sharding](https://docs.mongodb.org/manual/sharding/) for scalability. In case of a shard a predefined field will determine to which member of a shard the data goes. Special config servers maintain the locations of each document in the shard cluster. I won't go into any details regarding these setups, since my focus is the Cloud Adapter.

MongoDB offers several configuration options which can increase or decrease potential data reliability. For example it is possible to specify on how many nodes the data must be stored in a replication cluster before returning after an insert.

## MongoDB API
MongoDB offers a Javascript commandline tool (like SQLPlus from Oracle) for working with a MongoDB database instance. Simple commands or complete scripts written in Javascript can be executed here. Here is an example script:

    // switch to test db
    use test
    
    // drop test collection
    db.test.drop()
    
    // optionaly create test collection
    db.createCollection('test')
    
    // create JSON document and save it (save will add the _id to the original JSON document)
    testDoc={'field1':'value1'}
    db.test.save(testDoc)
    
    // query for _id
    db.test.find({'_id':testDoc._id})
    
    // query for field1
    db.test.find({'field1':'value1'})

For quite a lot of languages an API is available. Since the Cloud Adapter is build using Java I am gonne use the [Java API](https://docs.mongodb.org/ecosystem/drivers/java/). The Java API supports building BSON documents using the builder pattern. The **_id** is automatically added to an inserted document. This is some sample code which implements the same logic as the above Javascript in Java:

    // connect to MongoDB at localhost:27017
    MongoClient mongoClient = new MongoClient();
    
    // switch to test db
    MongoDatabase database = mongoClient.getDatabase("test");
    
    // drop test collection (note the collection will always be created by the getCollection command)
    MongoCollection<Document> collection = database.getCollection("test");
    collection.drop();
    
    // create a new collection
    collection = database.getCollection("test");
    
    // create BSON document and save it
    Document doc = new Document()
            .append("field1", "value1");
    collection.insertOne(doc);
    System.out.println(doc.toString());
    
    Document queryDoc1 = new Document()
            .append("_id", doc.get("_id"));
    System.out.println(collection.find(queryDoc1).first().toString());
    
    Document queryDoc2 = new Document()
            .append("field1", doc.get("field1"));
    System.out.println(collection.find(queryDoc2).first().toString());

## BSON

MongoDB uses [BSON](http://bsonspec.org/) to store and query document. This is Binary JSON which supports like JSON embedding documents and arrays but also supports the representation of data types. This is not part of JSON. This allows MongoDB to use dates, numbers, booleans, ObjectId and other [data types](https://docs.mongodb.org/manual/reference/bson-types/) in order to improve querying, sorting, etc.

For example to find all documents in a collection with a **qty** larger then **20** the **$gt** operator can be used:

    db.test2.insert({'qty':10})
    db.test2.insert({'qty':20})
    db.test2.insert({'qty':30})
    
    db.test2.find({'qty':{$gt:20}})

This will output one document: `{ "_id" : ObjectId("56fa806b1f1378215c21570d"), "qty" : 30 }`

There is a lot more possible with BSON, but this is just a small introduction to provide some context.

## Running MongoDB

Running MongoDB is really easy. Download the installer from the [website](https://www.mongodb.org/downloads#production) and install it. Create a folder named `C:\data\db` and run `mongod`. The MongoDB shell can be executed by the command `mongo`.

There are many advanced options for starting MongoDB and using different ports, credentials, etc. But for this setup I will use the default setup.