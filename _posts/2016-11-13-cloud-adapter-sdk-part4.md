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

Several security settings need to be applied to the domain for the Cloud Adapter to access the credentials. The [Oracle documentation](http://docs.oracle.com/middleware/1213/cloudadapter-rightnow/OCAIG/GUID-57BCEAD7-F1A7-4EC8-A93B-CD05A157934C.htm) describes it, though I did not need to deploy the file **cloudsdk.ear**. I needed to configure the ones described in the documentation.

## Oracle classes

According to the Oracle documentation these are the classes used in the runtime part of the adapter:

![Runtime classes]({{ site.github.url }}/assets/cloud-adapter-sdk-part4/adapter_runtime_classes.png)

## Refactoring

So it has been a while and I had wanted to refactor some code for better reuse in both the wizard and the actual adapter. I found out that the logger used by Oracle in the wizard is a different one from the logger used in the adapter. So I had to write some wrapper code around that in the form of the class **LoggerServiceWrapper**.

Further the connection code is now shared between the wizard and the adapter. And the BSON code is now better separated from the rest of the code.

## Accessing URI

The wizard stores the URI (which might contain credentials) in CSF. Additional permissions are required for the Cloud Adapter to access them. By Default the Cloud Adapter SDK expects a username password.

![Stored URI]({{ site.github.url }}/assets/cloud-adapter-sdk-part4/credentials.png)

### Required grants

I needed to give my own classes access to the credentials to work around some Oracle code which presumes it receives a username password type of credentials, where I need something else. Add a new grant:

- Click on **Create...**
- Use as codebase: **file:${soa.oracle.home}/soa/modules/oracle.cloud.adapter_12.1.3/nl.syntouch.oracle.adapter.cloud.mongodb.jar**

Add three new permissions:

| Permission Class | Resource Name | Permission Actions   |
|---|---|---|
| oracle.security.jps.service.credstore.CredentialAccessPermission | context=SYSTEM,mapName=SOA,keyName=* | * |
| java.lang.RuntimePermission | oracle.wsm.policyaccess |  |
| oracle.security.jps.service.credstore.CredentialAccessPermission | context=SYSTEM,mapName=oracle.wsm.security,keyName=* | * |

![Extra permissions]({{ site.github.url }}/assets/cloud-adapter-sdk-part4/extra_permissions.png)

### Retrieve URI

The Cloud Adapter SDK wants to access there credentials during initializing,but failes to do so. This will log an error message, but does not hinder the Cloud Adapter:

~~~~~~~~
java.lang.Exception: java.security.PrivilegedActionException: java.lang.Exception: Invalid object stored in CSF for GenericCredential. Only Map<String,String> is supported.
	at oracle.tip.adapter.fw.security.CredentialStoreFrameworkWrapperImpl.getCSFCredentials(CredentialStoreFrameworkWrapperImpl.java:83)
	at oracle.cloud.connector.impl.BasicAuthenticationManager.getCSFCredentials(BasicAuthenticationManager.java:88)
	at oracle.cloud.connector.impl.BasicAuthenticationManager.getAuthenticationProperties(BasicAuthenticationManager.java:79)
	at oracle.cloud.connector.impl.BasicAuthenticationManager.<init>(BasicAuthenticationManager.java:50)
	at oracle.cloud.connector.impl.CloudInvocationContextImpl.<init>(CloudInvocationContextImpl.java:49)
	at oracle.tip.adapter.cloud.CloudAdapterInteraction.createCloudOperation(CloudAdapterInteraction.java:220)
	at oracle.tip.adapter.cloud.CloudAdapterInteraction.execute(CloudAdapterInteraction.java:135)
...
~~~~~~~~

To fix this I would probarly need to implement a custom implementation of the **oracle.tip.adapter.cloud.CloudInteractionSpec** and possibly other classes. I was not really feeling like that, so I created a new class which I will use further on in the code. This class (**MongoDBCredentialStore**) reads the credentials from CSF and returns them. The permissions configured above will give this class the rights to do so.

~~~~~~~~java
public class MongoDBCredentialStore {

    private final CloudAdapterLoggingService logger;

    private String csfkey;
    private String csfmap;

    public MongoDBCredentialStore(CloudInvocationContext cloudInvocationContext) {
        logger = cloudInvocationContext.getLoggingService();

        csfkey = (String) cloudInvocationContext.getCloudConnectionProperties().get("csfkey");
        csfmap = (String) cloudInvocationContext.getCloudConnectionProperties().get("csfMap");
    }

    public Map<String, String> getCredentials() {
        Map<String, String> credentialMap = new HashMap<>();

        try {
            JpsContextFactory jpsContextFactory = JpsContextFactory.getContextFactory();
            JpsContext jpsContext = jpsContextFactory.getContext();
            final CredentialStore store = jpsContext.getServiceInstance(CredentialStore.class);

            credentialMap.putAll(AccessController.doPrivileged(new PrivilegedAction<Map<String, String>>() {
                public Map<String, String> run() {
                    Map<String, String> credentials = new HashMap<>();

                    try {
                        GenericCredential credential = (GenericCredential) store.getCredential(csfmap, csfkey);
                        credentials.put(Constants.MONGO_URI_KEY, (String) credential.getCredential());
                    } catch (CredStoreException ex) {
                        logger.logError("Unable to retrieve csfkey [" + csfkey + "] from csfmap [" + csfmap + "]", ex);
                    }

                    return credentials;
                }
            }));
        } catch (JpsException ex) {
            logger.logError("Unable to retrieve csfkey [" + csfkey + "] from csfmap [" + csfmap + "]", ex);
        }

        return credentialMap;
    }

    public String getUrl() {
        return getCredentials().get(Constants.MONGO_URI_KEY);
    }
}
~~~~~~~~

## Input & output

Normally you would implement the Oracle class **oracle.cloud.connector.api.CloudMessageHandler** to manipulate the request, response and error messages. But I could not figure out how to easily change XML into BSON using this interface. So instead I did the transformation from XML to BSON and vice versa in my implementation of **oracle.cloud.connector.api.Endpoint**.

To inject a custom **Endpoint** implementation the **AbstractCloudApplicationConnection** must provide an alternative **EndpointFactory**.

The class **MongoDBApplicationConnection** extends the Oracle class **AbstractCloudApplicationConnection** and in the constructors sets the class **MongoDBEndpointFactory** as **EndpointFactory**.

~~~~~~~~java
public MongoDBApplicationConnection() {
    super();

    setEndpointFactory(new MongoDBEndpointFactory());
}
~~~~~~~~

The class **MongoDBEndpoint** is instantiated by the class **MongoDBEndpointFactory** and implements four methods of the Oracle interface:

* initialize
* invoke
* addObserver
* destroy

### initialize

This method will initialize the **Endpoint** and will do the following things:

* Create a connection to the MongoDB instance
  * Based upon properties stored in the JCA file
  * Credentials are retrieved from CSF through the class MongoDBCredentialStore
* The operation name is stored
* The root and type namespace are determined and set

~~~~~~~~java
@Override
public void initialize(CloudInvocationContext cloudInvocationContext) throws CloudInvocationException {
    logger = cloudInvocationContext.getLoggingService();
    
    logger.log(CloudAdapterLoggingService.Level.DEBUG, "Initializing endpoint for MongoDB");
    connect(cloudInvocationContext);
    
    operationName = cloudInvocationContext.getTargetOperationName();
    initializeNamespaces(cloudInvocationContext);
}
~~~~~~~~

### invoke

This method will handle the actual invoke. Based upon the operation name a separate method is called to handle the actual work. In this case I only implemented the invoke operation so the **invokeInsert** method is called. This is not a method defined in the Oracle interface, but a way to keep separate future operations.

~~~~~~~~java
protected CloudMessage invokeInsert(Document requestBson) throws CloudInvocationException {
    connection.getCollection().insertOne(requestBson);
    
    Document responseBson = new Document()
            .append("_id", requestBson.getObjectId("_id"));
    
    Node responseXml = new BsonToXmlTransformer()
        .setRootNamespace(rootNamespace)
        .setDataNamespace(typeNamespace + "/response")
        .setWrapperElementName("insertResponse")
        .transform(responseBson);
    return CloudMessageFactory.newInstance().createCloudMessage(responseXml);
}

@Override
public CloudMessage invoke(CloudMessage requestMsg) throws RemoteApplicationException, CloudInvocationException {
    Node xml = requestMsg.getMessagePayloadAsDocument();
    Document bson = new XmlToBsonTransformer(
            .transform(xml)
            .get("Document", Document.class);
    
    CloudMessage responseMsg;
    switch(operationName) {
        case "insert":
            responseMsg = invokeInsert(bson);
            break;
        default:
            logger.log(CloudAdapterLoggingService.Level.ERROR, "Unknown operation [" + operationName + "]");
            throw new CloudInvocationException("Unknown operation [" + operationName + "]");
    }
    
    return responseMsg;
}
~~~~~~~~

A new BSON document is created here (note that the **CloudMessageHandler** is between the actual caller and the **Endpoint** implementation) by the class **XmlToBsonTransformer**. The BSON documented is insert and a new BSON document containing just the **_id** is returned and transformed back to XML.

### addObserver

To add an **EndpointObserver**.

### destroy

This method will close the connection to the MongoDB instance.

## Dependencies

Since the MongoDB Cloud Adapter uses the MongoDB driver it must be on the classpath. To keep things simple I decided to package the MongoDB driver with the MongoDB Cloud Adapter.

I created a new custom JDeveloper library called **MongoDB** which contained three JAR's:

* bson-3.2.2.jar
* mongodb-driver-3.2.2.jar
* mongodb-driver-core-3.2.2.jar

This library has been added to the Cloud Adapter project as dependency.

![JDeveloper MongoDB library]({{ site.github.url }}
/assets/cloud-adapter-sdk-part4/jdev_library.png)

In the deployment profile of the Cloud Adapter project I created a new file group called **MongoDB** of type **Packaging Type**. As contributors I added the three JAR files and in the filters I included everything except the **MANIFEST.MF**.

![JDeveloper MongoDB library]({{ site.github.url }}
/assets/cloud-adapter-sdk-part4/file_group_contributors.png)

![JDeveloper MongoDB library]({{ site.github.url }}
/assets/cloud-adapter-sdk-part4/file_group_filters.png)

This will package the classes inside the JAR files together with the MongoDB Cloud Adapter.

## Testing the Cloud Adapter

Copy the Cloud Adapter binary to the directory **$ORACLE_HOME/soa/soa/modules/oracle.cloud.adapter_12.1.3** and (re)start the domain.

In JDeveloper create a new composite and drag the MongoDB Cloud Adapter to the right side (of course have a running MongoDB instance). Configure the adapter and deploy the composite.

Now run the composite:

![Instance]({{ site.github.url }}
/assets/cloud-adapter-sdk-part4/instance.png)

![Instance]({{ site.github.url }}
/assets/cloud-adapter-sdk-part4/invoke.PNG)

And the data is stored in MongoDB (by searching for the **_id** returned by the Cloud Adapter):

~~~~~~~~
> db.test.find(ObjectId("582cdc4c9745d41c94b6b586"))
{ "_id" : ObjectId("582cdc4c9745d41c94b6b586"), "field1" : "Blog1234567890" }
~~~~~~~~

I have added the composite to the [GitHub](https://github.com/ninckblokje/MongoDBCloudAdapter) repository.

## Execution agent

Although I have not tested the MongoDB Cloud Adapter yet with Integration Cloud, I have written a Docker container for installing and running the Execution Agent required for the MongoDB Cloud Adapter and the on-premise integration. Please check my [GitHub](https://github.com/ninckblokje/ics-execution-agent-docker) reposity for that.

The Docker container is of early beta quality
