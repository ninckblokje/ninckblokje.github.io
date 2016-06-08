---
layout: post
title:  "Oracle Cloud Adapter SDK - Part 3: Build the wizard"
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
- Part 3: Building the wizard
- Who knows!

Just a small warning: Always keep track of Oracle license information and the Oracle certification matrix!

## Apologies

It took me quite a while to figure out how the Cloud Adapter SDK worked (at least the wizard part). So I decided to only support the insert method for now. Also the code is quite messy, logging is absent and exception handling is minimum. Hopefully this will change in the future.

## What should the wizard do?

During the wizard I want the following to happen or to be supported for the insert operation:

- Specify MongoDB URL, database and collection and test the connection
- Specify the operation
- Retrieve a sample document and allow the user to edit it
- Provide supported for fully unstructered documents
- Generate a WSDL and an XSD
- Store the sample or user edited document

## Documentation

So I finally found a link to Oracle documentation regarding the Cloud Adapter SDK: [Developing Custom Oracle Cloud Adapters](https://docs.oracle.com/cloud/latest/intcs_gs/CCCDG/toc.htm).

## Project setup

A Cloud Adapter SDK implementation is based upon a JDeveloper Extension. So first we need to create a new JDeveloper Extension project. Select **File** -> **New** -> **Project** in JDeveloper. Select **Extension Project** and provide a suitable name.

![New extension project]({{ site.github.url }}/assets/cloud-adapter-sdk-part3/new_extension.png)

On the last page you can specify some additional properties, like the owner and where the resource bundle is stored. Either change it or accept the defaults. A small project is generated with the following structure:

- src
  - META-INF
    - extension.xml
    - MANIFEST.MF
  - [PACKAGE]
    - Res.properties
- [PROJECT].jpr

Now we are gonne make the following changes:

- Update **extension.xml**
- Update **MANIFEST.MF**
- Add a **cloud-adapter.xml**

*** extension.xml

First we need to edit the **extension.xml** file:

  - Change the attribute **esdk-version** to 1.0
  - Add a trigger hook under the element **triggers**:

~~~~~~~~
<sca-config-hook xmlns="http://xmlns.oracle.com/ide/extension/sca-config">
    <adapter-types>
        <adapter-type technology-keys="SOA,BPM">
            <name>MongoDBCloudAdapter</name>
            <folder-name>Cloud Adapters</folder-name>
            <category-name>Cloud</category-name>
            <description>MongoDB Cloud Adapter</description>
            <inbound>false</inbound>
            <outbound>true</outbound>
            <binding-type>jca</binding-type>
            <binding-subtype>MongoDBCloudAdapter</binding-subtype>
            <implementation-class>nl.syntouch.oracle.adapter.cloud.mongodb.wizard.MongoDBJcaEndpoint</implementation-class>
            <icon16x16>/nl/syntouch/oracle/adapter/cloud/mongodb/images/mongodb-logo.png</icon16x16>
            <icon20x20>/nl/syntouch/oracle/adapter/cloud/mongodb/images/mongodb-logo.png</icon20x20>
        </adapter-type>
        <adapter-type technology-keys="ServiceBusTechnology">
            <name>MongoDBCloudAdapter</name>
            <folder-name>Cloud Adapters</folder-name>
            <category-name>Cloud</category-name>
            <description>MongoDBCloudAdapter</description>
            <inbound>false</inbound>
            <outbound>true</outbound>
            <binding-type>sb</binding-type>
            <binding-subtype>jca/MongoDBCloudAdapter</binding-subtype>
            <implementation-class>nl.syntouch.oracle.adapter.cloud.mongodb.wizard.MongoDBSCAEndpointJcaAdapter</implementation-class>
            <icon16x16>/nl/syntouch/oracle/adapter/cloud/mongodb/images/mongodb-logo.png</icon16x16>
            <icon20x20>/nl/syntouch/oracle/adapter/cloud/mongodb/images/mongodb-logo.png</icon20x20>
        </adapter-type>
    </adapter-types>
</sca-config-hook>
~~~~~~~~

There are two different adapter-types that can be configured, one for Service Bus (and Integration Cloud) and one for SOA / BPM composites. Note the different value for the elements **binding-type** and **binding-subtype** and the attribute **technology-keys**. This is an outbound adapter only so the element **inbound** is set to **false** and the element **outbound** is set to **true**.

Service Bus and SOA / BPM composites both have a separate entry point into the wizard so both need a separate implementation class. For Service Bus this should be an implemenation of **oracle.sb.tooling.ide.sca.internal.model.jca.SCAEndpointJcaAdapter**. For SOA / BPM composites this should be an implemenation of **oracle.tip.tools.ide.adapters.designtime.adapter.JcaEndpointAbstract**. I will describe this further on.

### MANIFEST.MF

The **MANIFEST.MF** needs to be updated to load the required OSGi modules. Besides the default parameters, **Bundle-Name** is added and the parameter **Require-Bundle** is changed to refer a load of OSGi modules.

~~~~~~~~
Bundle-ManifestVersion: 2.0
Bundle-Version: 1
Bundle-Name: nl.syntouch.oracle.adapter.cloud.mongodb
Bundle-SymbolicName: nl.syntouch.oracle.adapter.cloud.mongodb
Bundle-ClassPath: .
Require-Bundle: oracle.external.oracle-jrf-webservices,oracle.external
 .oracle-jrf-wsdl,oracle.external.jaxws,oracle.external.oracle-jrf-saa
 j,oracle.sca.modeler,oracle.sca.ui.adapters,oracle.cloud.adapter.api,
 oracle.cloud.adapter.impl,oracle.javatools,oracle.jewt-core,oracle.ex
 ternal.balishare,oracle.sb.tooling.ide.sca,oracle.soa.tools.widgets,o
 racle.bpm-ide-common,oracle.uic,oracle.ide,oracle.tools.cloud.adapter
 .ide
~~~~~~~~

### cloud-adapter.xml

A a new file in the **META-INF** directory called **cloud-adapter.xml**. I have used the following content:

~~~~~~~~
<tns:AdapterPluginConfig xmlns:tns="http://xmlns.oracle.com/adapters/cloud">
    <adapterPluginID>MongoDBCloudBAdapter</adapterPluginID>
    <displayName>MongoDBCloudBAdapter</displayName>
    <description>MongoDB Cloud Adapter</description>
    <adapterFactory>nl.syntouch.oracle.adapter.cloud.mongodb.plugin.MongoDBAdapterFactory</adapterFactory>
    <supportedSecurityPolicies>
        <policy>CUSTOM</policy>
    </supportedSecurityPolicies>
    <defaultSecurityPolicy>CUSTOM</defaultSecurityPolicy>
    <propertyDefinitions>
        <propertyDefinition>
            <propertyName>MongoDB.uri</propertyName>
            <propertyType>STRING</propertyType>
            <propertyGroup>CREDENTIALS</propertyGroup>
            <required>true</required>
            <persistent>true</persistent>
            <displayName>Mongo URI, example: mongodb://localhost:27017</displayName>
        </propertyDefinition>
        <propertyDefinition>
            <propertyName>MongoDB.db</propertyName>
            <propertyType>STRING</propertyType>
            <propertyGroup>CONNECTION_PROPS</propertyGroup>
            <required>true</required>
            <persistent>true</persistent>
            <displayName>Database</displayName>
        </propertyDefinition>
        <propertyDefinition>
            <propertyName>MongoDB.collection</propertyName>
            <propertyType>STRING</propertyType>
            <propertyGroup>CONNECTION_PROPS</propertyGroup>
            <required>true</required>
            <persistent>true</persistent>
            <displayName>Collection</displayName>
        </propertyDefinition>
    </propertyDefinitions>
    <UIProviderClass>nl.syntouch.oracle.adapter.cloud.mongodb.wizard.MongoDBUIProvider</UIProviderClass>
    <resourceBundle>nl.syntouch.oracle.adapter.cloud.mongodb.definition.MongoDBCloudAdapterBundle</resourceBundle>
    <sdkVersion>2.1.0</sdkVersion>
</tns:AdapterPluginConfig>
~~~~~~~~

This file actually configures (part) of the Cloud Adapter SDK:

- The **adapterPluginID** should be unique
- The **displayName** should be same when used in code and in this file
- **adapterFactory** should point to an implementation of **oracle.tip.tools.ide.adapters.cloud.api.plugin.CloudApplicationAdapterFactory**
  - I will describe this further on
- **supportedSecurityPolicies** is set to CUSTOM since any of the other policies are not applicable here:
  - USERNAME_PASSWORD_TOKEN
  - BASIC_AUTH
  - NONE
- Then comes a list of custom properties from the property group CONNECTION_PROPS
  - These properties will be used to store the MongoDB URL, database and collection
- **UIProviderClass** should point to an implemenation of **oracle.tip.tools.adapters.cloud.api.CloudAdapterUIProvider**
  - I will describe this further on
- **resourceBundle** should point to a resource bundle (I use the same one in my extension.xml)

## Debugging

Debugging is really easy (thanks to [Wilfred van der Deijl](https://twitter.com/wilfreddeijl) for showing me how this works). Simple right click on the project and select **Debug Extension**. A new JDeveloper instance is started and you can do full debugging including brake points, step through, etc.

![Debug extension]({{ site.github.url }}/assets/cloud-adapter-sdk-part3/debug_extension.png)

Don't forget to build the project first!

## Implementation

### Implemented / extended Oracle classes

So we need quite a few implemented or extended Oracle classes. I will provide a list of the interfaces and (abstract) classes I implemented / extended:

- oracle.tip.tools.ide.adapters.cloud.api.generation.ArtifactGenerator
  - For generating additional files, in my case for storing a JSON sample document
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.plugin.bson.BSONArtifactGenerator**
- javax.activation.DataSource
  - Not a real Oracle class!
  - Used for retrieving the JSON document
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.plugin.bson.BSONDataSource**
- oracle.tip.tools.ide.adapters.cloud.impl.generation.AbstractRuntimeMetadataGenerator
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.plugin.generator.MongoDBMetadataGenerator**
- oracle.tip.tools.ide.adapters.cloud.impl.metadata.wsdl.AbstractMetadataBrowser
  - For exposing the pre-wizard model
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.plugin.metadata.MongoDBMetadataBrowser**
- oracle.tip.tools.ide.adapters.cloud.api.metadata.MetadataParser
  - For parsing data and building the pre-wizard model
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.plugin.metadata.MongoDBMetadataParser**
- oracle.tip.tools.ide.adapters.cloud.impl.plugin.AbstractCloudApplicationAdapter
  - For retrieving an **AbstractCloudConnection**, **AbstractMetadataBrowser** and **AbstractRuntimeMetadataGenerator** instance
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.plugin.MongoDBAdapter**
- oracle.tip.tools.ide.adapters.cloud.api.plugin.CloudApplicationAdapterFactory
  - For creating the **AbstractCloudApplicationAdapter** instance
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.plugin.MongoDBAdapterFactory**
- oracle.tip.tools.ide.adapters.cloud.api.connection.AbstractAuthenticationScheme
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.plugin.MongoDBAuthenticationScheme**
- oracle.tip.tools.ide.adapters.cloud.api.connection.AbstractCloudConnection
  - Used for creating and testing a connection
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.plugin.MongoDBConnection**
- oracle.tip.tools.adapters.cloud.impl.CloudAdapterConnectionPage
  - Wizard page for specifying the connection properties
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.wizard.page.MongoDBConnectionPage**
- oracle.tip.tools.adapters.cloud.api.ICloudAdapterPage
  - For implementing a new operations page
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.wizard.page.MongoDBOperationsPage**
- oracle.tip.tools.ide.adapters.designtime.adapter.JcaEndpointAbstract
  - For exposing the wizard in a SOA / BPM composite
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.wizard.MongoDBJcaEndpoint**
- oracle.sb.tooling.ide.sca.internal.model.jca.SCAEndpointJcaAdapter
  - For exposing the wizard in a Service Bus project
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.wizard.MongoDBSCAEndpointJcaAdapter**
- oracle.tip.tools.adapters.cloud.impl.AbstractCloudAdapterUIBinding
  - Specifies which pages will be shown in which order
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.wizard.MongoDBUIBinding**
- oracle.tip.tools.adapters.cloud.api.CloudAdapterUIProvider
  - For creating a new instance of **AbstractCloudAdapterUIBinding** and returning the name of the adapter
  - See **nl.syntouch.oracle.adapter.cloud.mongodb.wizard.MongoDBUIProvider**

### Context

There are basically three different context types:

- AdapterPluginContext
- DefaultRuntimeGenerationContext
- RuntimeGenerationContext

Most of the time in the wizard you can use the **AdpaterPluginContext**. This context contains all of the data (though connection properties are should be retrieved using the method **getContextObject("connectionProperties")**). The **DefaultRuntimeGenerationContext** is only used when artifacts are being generated.

I am not really sure, though I believe the **RuntimeGenerationContext** is only used before starting the wizard. I might be wrong!

### Different models

There are basically three different models in the wizard you should be aware of:

- The pre-wizard model
- The post-wizard model
- The WSDL model

It is basically modelled upon cloud services which have a standard interface which can be used to retrieve business objects. The standard interface is then modelled using the pre-wizard model. The post-wizard model contains the business objects selected by the user and the WSDL model is used internally by the Cloud Adapter SDK for transforming the post-wizard model to real WSDL and XSD files.

The pre-wizard model is stored in an instance of **oracle.tip.tools.ide.adapters.cloud.api.model.CloudApplicationModel** and is generated by an implementation of **MetadataParser** and exposed by an instance of **AbstractMetadataBrowser**.

The post-wizard model is stored in an instance of **oracle.tip.tools.ide.adapters.cloud.api.model.TransformationModel**. Data from the pre-wizard model is transformed and stored into this model (though not everything). For now I do this transformation in my operation page (**MongoDBOperationsPage**).

Once again, I am not completely sure about this.

### Wizard sequence

![Wizard start classes]({{ site.github.url }}/assets/cloud-adapter-sdk-part3/wizard_start_classes.png)



### Editing existing adapter

### Specials

targetWSDLURL

## Useful libraries

During development it can be really useful to take a look at the following JAR files:

- $ORACLE_HOME/oracle_common/modules/com.oracle.webservices.orawsdl-api_12.1.3.jar
- $ORACLE_HOME/osb/lib/uitools/oracle.tools.cloud.adapter.sdk.jar
- $ORACLE_HOME/osb/lib/uitools/oracle.tools.uiobjects.sdk.jar
- $ORACLE_HOME/osb/plugins/jdeveloper/extensions/oracle.tools.cloud.adapter.ide.jar
- $ORACLE_HOME/soa/plugins/jdeveloper/extensions/oracle.cloud.adapter.api.jar
- $ORACLE_HOME/soa/plugins/jdeveloper/extensions/oracle.cloud.adapter.impl.jar
- $ORACLE_HOME/soa/plugins/jdeveloper/extensions/oracle.sca.modeler.jar
- $ORACLE_HOME/soa/plugins/jdeveloper/extensions/oracle.sca.ui.adapters.jar
- $ORACLE_HOME/soa/soa/modules/oracle.cloud.adapter_12.1.3/EloquaCloudAdapter.jar
