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

This blog post has also been in draft for over a month...

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

~~~~~~~~xml
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

A new file in the **META-INF** directory called **cloud-adapter.xml**. I have used the following content:

~~~~~~~~xml
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

### Login and URL

To keep things simple and flexible a decided to let the user only input a MongoDB URI (property **MongoDB.uri**). This URI contains the username, password, hostname, port and several connection properties. It is stored (encrypted) in **csf**.

### Wizard sequence

![Wizard start classes]({{ site.github.url }}/assets/cloud-adapter-sdk-part3/wizard_start_classes.png)

### Editing existing adapter

### Connection test

### Sample document

### Wizard pages

The Cloud Adapter SDK provides several pages for usage, however I only found two pages useful. The others I implemented myself. The wizard consists of the following pages:

1. Welcome page
2. Connection page
3. Operation page
4. Summary page

For the welcome page I use the Oracle provided class **CloudAdapterWelcomePage**. For the summary page I also use an Oracle provided class **CloudAdapterSummaryPage**. The connection page I implemented in the class **MongoDBConnectionPage** (which extends the class **CloudAdapterConnectionPage**). The operation page I implemented in the class **MongoDBOperationsPage** which implements the class **ICloudAdapterPage**. I had this brainwave to create an abstract class for easy page implementation (called **AbstractMongoDBPage**), though this has not been very useful.

The method **getEditPages** in the class **MongoDBUIBinding** specifies which pages are shown and in which order the pages are shown.

~~~~~~~~java
@Override
public LinkedHashMap<String, ICloudAdapterPage> getEditPages(Object object) {
    CloudAdapterFilter filter = (CloudAdapterFilter) this.context.getContextObject(CloudAdapterConstants.UI_CLOUD_ADAPTER_FILTER);

    LinkedHashMap<String, ICloudAdapterPage> editPages = new LinkedHashMap<String, ICloudAdapterPage>();

    editPages.put(CloudAdapterConstants.WELCOME_PAGE_ID, new CloudAdapterWelcomePage(this.context));
    if (filter.isAddConnection()) {
        editPages.put(CloudAdapterConstants.CONNECTION_PAGE_ID, new MongoDBConnectionPage(this.context));
    }
    editPages.put(CloudAdapterConstants.OPERATIONS_PAGE_ID, new MongoDBOperationsPage(this.context));
    editPages.put(CloudAdapterConstants.SUMMARY_PAGE_ID, new CloudAdapterSummaryPage(this.context));

    return editPages;
}
~~~~~~~~

#### Connection page

My connection page implementation is there for one reason only: The repopulate the fields when editing an existing adapter. To do this I override the method **getPageEditFields** and added two new methods:

- addMissingData
  - For adding the data back to the fields
- getConnectionProperties
  - For retrieving the connection properties from the context

~~~~~~~~java
public class MongoDBConnectionPage extends CloudAdapterConnectionPage {

    private final LoggerService logger;

    private AdapterPluginContext ctx;

    public MongoDBConnectionPage(AdapterPluginContext adapterPluginContext) {
        super(adapterPluginContext);
        ctx = adapterPluginContext;

        logger = adapterPluginContext.getServiceRegistry().getService(LoggerService.class);
    }

    protected void addMissingData(LinkedList<EditField> fields, String fieldName, String contextKey) {
        Map<String, String> connProps = getConnectionProperties();
        if (connProps == null) return;

        String value = connProps.get(contextKey);
        if (value == null) return;

        for (EditField field: fields) {
            if (fieldName.equals(field.getName())
                    && field.getObject() instanceof ITextBoxObject) {
                ITextBoxObject textBox = (ITextBoxObject) field.getObject();
                textBox.setValue(value);
            }
        }
    }

    @SuppressWarnings("unchecked")
    protected Map<String, String> getConnectionProperties() {
        // only available in edit mode
        return (Map<String, String>) ctx.getContextObject("connectionProperties");
    }

    @Override
    public LinkedList<EditField> getPageEditFields() {
        LinkedList<EditField> fields = super.getPageEditFields();

        addMissingData(fields, Constants.MONGO_COLLECTION_KEY, Constants.MONGO_COLLECTION_KEY);
        addMissingData(fields, Constants.MONGO_DB_KEY, Constants.MONGO_DB_KEY);

        return fields;
    }
}
~~~~~~~~

#### Operation page

![Sample document]({{ site.github.url }}/assets/cloud-adapter-sdk-part3/sample_json.png)

The operations page contains a lot more logic then the connection page. I want to provide some switches there and provide the ability for the user to change the sample BSON / JSON document. I decided to implement just the Oracle interface **ICloudAdapterPage** in the class **MongoDBOperationsPage**. The interface specifies I must implement the following methods:

- getPageId
- getPageName
- getPageTitle
- getHelpId
- getWelcomeText
- getPageEditFields
- getChildrenEditPages
- updateBackEndModel
- getUpdatedEditPages
- validatePage

Thie first five methods are just there for returning some static stuff. The others are more interesting.

##### getPageEditFields

Lets start with the method **getPageEditFields**. This method return a **LinkedList** of **EditField** instances. Basically just the fields we show on this page. I will need the following fieds (with label of course):

- SelectObject
  - For selecting the operation
  - For selecting the mode
- TextAreaObject
  - For providing the ability to edit a sample BSON / JSON document

Creating a field is quite easy, but only when you know how! First create the object, then wrap it in an **EditField** and add it to the list.

~~~~~~~~java
// String[] values, String[] formattedValues, String selected, int displayMode, boolean hasEvents
SelectObject operationsSelect = UIFactory.createSelectObject(operationNamesArray, operationNamesArray,
                                                         operationNamesArray[0], 2, true);
// String name, String label, String description, boolean isrequired, boolean isDisabled, UIObject object, EditField.LabelFieldLayout labelFieldLayout, String helpText, EditField.LabelFieldAlignment oneRowlabelFieldAlignment
fields.add(UIFactory.createEditField(operationsEditField, getText(operationsLabelKey),
                                     getText(operationsDescriptionKey), true, false,
                                     operationsSelect, EditField.LabelFieldLayout.ONE_ROW_LAYOUT,
                                     getText(operationsHelpKey), EditField.LabelFieldAlignment.LEFT_LEFT));
~~~~~~~~

In this method I also check for context values and retrieve the sample document from the document. This can either be retrieved from the database (when a new adapter is created) or retrieved from a file (when an existing adapter is being edited).

### Specials

#### Storing addtional properties

When you define additional properties they will not get stored in the JCA file automatically. Credentials are automatically stored in the **csf**. For connection properties only the property **targetWSDLURL** is automatically stored. However I wanted the following additional properties:

- MongoDB.collection
  - For storing the collection name
- MongoDB.db
  - For storing the database name
- MongoDB.mode
  - For storing the operation mode, which can be structured (strongly typed) or unstructered (any type)
  - Not sure if I will actually use it
- MongoDB.sampleBsonFile
  - The filename of a sample BSON / JSON file
  - Only used in the wizard

The class **MongoDBConnection** extends the Oracle class **AbstractCloudConnection**. The method **getPersistentPropertyNames** returns all conncetion property names. By overriding this method we can tell the Cloud Adapter SDK to store more properties. Do note though they are stored as connection properties!

~~~~~~~~java
@Override
public Set<String> getPersistentPropertyNames() {
    Set<String> propNames = super.getPersistentPropertyNames();

    propNames.add(Constants.MONGO_DB_KEY);
    propNames.add(Constants.MONGO_COLLECTION_KEY);

    propNames.add(Constants.CONTEXT_MODE_KEY);
    propNames.add(Constants.CONTEXT_SAMPLE_FILE_KEY);

    return propNames;
}
~~~~~~~~

In the class **MongoDBUIBinding** (which extends the Oracle class **AbstractCloudAdapterUIBinding**) I added a method called **updateContext** for transfering the properties from the connection properties to the **AdapterPluginContext**. This method is called from the constructor.

~~~~~~~~java
protected void updateContext() {
    if (getConnectionProperties() ==null) return;

    if (getConnectionProperties().containsKey(Constants.CONTEXT_MODE_KEY)) context.setContextObject(Constants.CONTEXT_MODE_KEY, getConnectionProperties().get(Constants.CONTEXT_MODE_KEY));
    if (getConnectionProperties().containsKey(Constants.CONTEXT_SAMPLE_FILE_KEY)) context.setContextObject(Constants.CONTEXT_SAMPLE_FILE_KEY, getConnectionProperties().get(Constants.CONTEXT_SAMPLE_FILE_KEY));
}
~~~~~~~~

#### Creating additional files

I wanted to give the user the ability to specify a custom BSON / JSON document which gets used in creating the WSDL and XSD's. However this document needs to be stored so the used can see it when editing the adapter.

![Sample document]({{ site.github.url }}/assets/cloud-adapter-sdk-part3/sample_json.png)

To do this I created a new class **BSONArtifactGenerator** (which implements the Oracle class **ArtifactGenerator**). This class will generate the file.

~~~~~~~~java
public class BSONArtifactGenerator implements ArtifactGenerator {

    public void generate(DefaultRuntimeGenerationContext defaultRuntimeGenerationContext) throws CloudApplicationAdapterException {
        try {
            String dirLocation = (String) defaultRuntimeGenerationContext.getContextObject("generatedBaseDir");
            URI dirUri = new URI(dirLocation);

            String fileName = (String) defaultRuntimeGenerationContext.getContextObject(Constants.CONTEXT_SAMPLE_FILE_KEY);
            Document bson = (Document) defaultRuntimeGenerationContext.getContextObject(Constants.CONTEXT_PARSE_DOCUMENT_KEY);

            File dir = new File(dirUri);
            dir.mkdirs();

            File f = new File(dir, fileName);

            Files.write(Paths.get(f.toURI()), bson.toJson().getBytes("UTF-8"), StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
        } catch (IOException | URISyntaxException ex) {
            throw new CloudApplicationAdapterException(ex);
        }
    }
}
~~~~~~~~

Three properties are read from the context:

- generatedBaseDir
  - Oracle property which points to the **adapter** folder in the project
- MongoDB.parseBsonDocument
  - Where I store the document specified in the wizard
- MongoDB.sampleBsonFile
  - Unique name for storing the document

In the class **MongoDBMetadataGenerator** (extends the Oracle class **AbstractRuntimeMetadataGenerator**) I override the method **addArtifactGenerators** so I can add my own artifact generator.

~~~~~~~~java
@Override
protected void addArtifactGenerators(List<ArtifactGenerator> generators) {
    super.addArtifactGenerators(generators);
    generators.add(new BSONArtifactGenerator());
}
~~~~~~~~

The filename should only be generated when a new adapter is created (though should be unique for each adapter). To do this I decided to use a GUID as a filename. In the method **initializeContext** which is part of the **MongoDBMetadataGenerator** (extends the Oracle class **AbstractRuntimeMetadataGenerator**) a new unique filename is generated when no name has been stored.

~~~~~~~~java
@Override
protected void initializeContext(RuntimeGenerationContext runtimeGenerationContext) {
    CloudConnection connection = getCloudConnection();

    connection.getConnectionProperties().setProperty(Constants.CONTEXT_MODE_KEY, (String) ctx.getContextObject(Constants.CONTEXT_MODE_KEY));

    if (ctx.getContextObject(Constants.CONTEXT_SAMPLE_FILE_KEY) == null) {
        ctx.setContextObject(Constants.CONTEXT_SAMPLE_FILE_KEY, UUID.randomUUID().toString() + ".json");
    }
    connection.getConnectionProperties().setProperty(Constants.CONTEXT_SAMPLE_FILE_KEY, (String) ctx.getContextObject(Constants.CONTEXT_SAMPLE_FILE_KEY));
}
~~~~~~~~

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
