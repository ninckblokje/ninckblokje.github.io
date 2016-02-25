---
layout: post
title:  "How to create an OSB 11g Archetype"
tags:
- osb
- maven
- how_to
- long_article
disqus: true
---

In my first post ([OSB 11g Maven Archetype]({% post_url 2016-02-09-osb-11g-archetype %})) I wrote about the example I made for an OSB 11g Maven archetype.
Now I want to show how to create an archetype. This post has become quite long because of this!

Creating an OSB archetype consist out of the following steps:

1. Create an OSB project
2. Create an archetype out of an existing OSB project
3. Add variables
4. Modify files to use variables (create template file)
5. Modify file names to use variables
6. Test it!

The entire example project is placed under [Apache 2.0](https://gitlab.com/syntouch/example-osb11g-archetype/blob/master/LICENSE) license at the [GitLab](https://gitlab.com/syntouch/example-osb11g-archetype) of SynTouch (my employer).

## What should it do

I want the archetype to be able to create an OSB project with a proxy service and a business service (which points to a SOA Suite URL). The name of the project
can be split into two parts (example **TestService_2.0**):

- Name
- Major version 

The name (without the major version) should also be used as file name for the proxy and business services.

The archetype should be able to support one or multiple operations.

## Step  1 - Create an OSB project

I have created a sample OSB project with the following structure:

- Proxy service
- Business service
- WSDL and XSD
- Folder structure

![Project structure]({{ site.github.url }}/assets/how-to-create-osb-11g-archetype/project_structure.png)

The proxy service has got only one operation (though with an operational branch). The default operation throughts an error and one operation has been implemented with a pipeline pair containing a log step and a routing.

![Pipeline]({{ site.github.url }}/assets/how-to-create-osb-11g-archetype/pipeline.png)

This service should be as simple as possible and only containing the required steps. Although the archetype should support multiple operations, this service will only contain one operation. We will solve this is step 4.

## Step 2 - Create an archetype out of an existing OSB project

Luckily Maven provides an easy mechanism for creating an archetype out of something existing. First we much create a **pom.xml** in the root of the OSB project with just a few lines of XML:

    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
        <groupId>nl.syntouch.examples.servicebus</groupId>
        <artifactId>MavenArchetypeTestService_1.0</artifactId>
        <version>1.0-SNAPSHOT</version>
        <packaging>jar</packaging>
	</project>

Please note that the **artifactId** and **groupId** will be different for each customer.

Next we can tell Maven to generate the archetype.

    $ mvn archetype:create-from-project

In the **target/generated-sources/archetype** directory Maven has generated the archetype, well the first version of the archetype. It has the following structure:

- pom.xml
- src
  - main
    - resources
      - archetype-resources
      - META-INF
        - maven
          - archetype-metadata.xml

The folder **archetype-resources** contains all the (template) files, in this case the OSB project we made in step 1. The file **archetype-metadata.xml** describes how to handle these files and custom variables. It is possible to define custom variables in a template or in **archetype-metadata.xml**. More on that in step 3.

It would be wise to move the generated archetype somewhere 'safe'. Leaving it in the target directory means source control systems will ignore it and a **mvn clean** command will remove it without warning.

## Step 3 - Add variables

In order to make several things dynamic we need to specify variables. These variables can be provided at the command line or Maven will ask for them. By default Maven will require us to input the following variables:

- artifactId
- groupId
- package
- version

Now I will use the artifactId as the name of the service combined with the major version (example **TestService_2.0**). The groupId, package and version are only used for the generated **pom.xml** file. Two more pieces of information are required:

- Name of the service without major version
- One ore more operation names

Custom variables can be used to specify this. Custom variables are defined in the **archetype-metadata.xml** file. An **archetype-metadata.xml** has already been generated for us in the previous step. This is the current content:

    <archetype-descriptor xsi:schemaLocation="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.0.0 http://maven.apache.org/xsd/archetype-descriptor-1.0.0.xsd" name="    MavenArchetypeTestService_1.0"
        xmlns="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.0.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
      <fileSets>
        <fileSet filtered="true" encoding="UTF-8">
          <directory>.settings</directory>
          <includes>
            <include>**/*.xml</include>
          </includes>
        </fileSet>
        <fileSet encoding="UTF-8">
          <directory>proxy</directory>
          <includes>
            <include>**/*.proxy</include>
          </includes>
        </fileSet>
        <fileSet encoding="UTF-8">
          <directory>wsdl</directory>
          <includes>
            <include>**/*.wsdl</include>
          </includes>
        </fileSet>
        <fileSet encoding="UTF-8">
          <directory>business</directory>
          <includes>
            <include>**/*.biz</include>
          </includes>
        </fileSet>
        <fileSet encoding="UTF-8">
          <directory>xsd</directory>
          <includes>
            <include>**/*.xsd</include>
          </includes>
        </fileSet>
        <fileSet encoding="UTF-8">
          <directory>.settings</directory>
          <includes>
            <include>**/*.component</include>
            <include>**/*.prefs</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8">
          <directory></directory>
          <includes>
            <include>.project</include>
          </includes>
        </fileSet>
        <fileSet encoding="UTF-8">
          <directory></directory>
          <includes>
            <include>.gitignore</include>
          </includes>
        </fileSet>
      </fileSets>
    </archetype-descriptor>

Several filesets have already been defined, but we will come to this part later. As first child of the element **archetype-descriptor.xml** the element **requiredProperties** must be added. Each variable will be a childelement of this element and surprisingly called **requiredProperty**. In this example the requiredProperties **serviceName** and **operations** will be added:

    <archetype-descriptor
      xsi:schemaLocation="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.0.0 http://maven.apache.org/xsd/archetype-descriptor-1.0.0.xsd"
      name="example-osb11g-archetype"
      xmlns="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.0.0"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
      <requiredProperties>
        <requiredProperty key="serviceName"/>
        <requiredProperty key="operations"/>
      </requiredProperties>
      <fileSets>
        ...
      </fileSets>
    </archetype-descriptor>

The **operations** variable will contain a list of operation names (seperated by a space) and the **serviceName** variable will contain the name without the major version. Note: It is possibly to provide a value for a variable, but it is not possible to mix variables in this file!

## Step 4 - Modify files to use variables (create template file)

The next step is make sure the existing files will be using the variables in order for them to become a real template. The following files exist now in the archetype:

- .settings
  - org.eclipse.wst.common.component
  - org.eclipse.wst.common.project.facet.core.xml
  - org.eclipse.wst.validation.prefs
- business
  - MavenArchetypeTestServiceBS.biz
- proxy
  - MavenArchetypeTestServicePS.proxy
- wsdl
  - MavenArchetypeTestService.wsdl
- xsd
  - MavenArchetypeTestService.xsd
.project
pom.xml

Renaming files will be done in the next step. First we need to make sure that all the files will be processed as template. In the **requiredProperty** a list of filesets has been specified. The attribute **filtered** (with value **true**) will indicate that all the files in the directory will be processed as template. If the attribute is missing for any of the filesets it should be added. The resulting **archetype-descriptor.xml** will look like this:

    <archetype-descriptor xsi:schemaLocation="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.0.0 http://maven.apache.org/xsd/archetype-descriptor-1.0.0.xsd" name="    MavenArchetypeTestService_1.0"
        xmlns="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.0.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
      <requiredProperties>
        <requiredProperty key="serviceName"/>
        <requiredProperty key="operations"/>
      </requiredProperties>
      <fileSets>
        <fileSet filtered="true" encoding="UTF-8">
          <directory>.settings</directory>
          <includes>
            <include>**/*.xml</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8">
          <directory>proxy</directory>
          <includes>
            <include>**/*.proxy</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8">
          <directory>wsdl</directory>
          <includes>
            <include>**/*.wsdl</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8">
          <directory>business</directory>
          <includes>
            <include>**/*.biz</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8">
          <directory>xsd</directory>
          <includes>
            <include>**/*.xsd</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8">
          <directory>.settings</directory>
          <includes>
            <include>**/*.component</include>
            <include>**/*.prefs</include>
          </includes>
        </fileSet>
        <fileSet filtered="true" encoding="UTF-8">
          <directory></directory>
          <includes>
            <include>.project</include>
          </includes>
        </fileSet>
        <fileSet encoding="UTF-8">
          <directory></directory>
          <includes>
            <include>.gitignore</include>
          </includes>
        </fileSet>
      </fileSets>
    </archetype-descriptor>

Just a quick intro into the templating engine. Maven uses the Velocity template engine in an archetype. If special characters must be used in a template file they should be added as variable inside the template file. This way you can use the variable for printing the special character. Add the following lines to each file:

    #set( $symbol_pound = '#' )
    #set( $symbol_dollar = '$' )
    #set( $symbol_escape = '\' )

**#set** is a Velocity command and **$symbol_pound = '#'** defines the name of the variable and its value. So the following variables will now be available in each template file:

- $symbol_pound
- $symbol_dollar
- $symbol_escape
- $artifactid
- $groupId
- $version
- $package
- $serviceName
- $operations

### org.eclipse.wst.common.component

    <project-modules id="moduleCoreId">
        <wb-module deploy-name="MavenArchetypeTestService"/>
    </project-modules>

In this file replace the value of the attribute **deploy-name** with **$artifactId** to make the project name dynamic.

    <project-modules id="moduleCoreId">
        <wb-module deploy-name="${artifactId}"/>
    </project-modules>

### org.eclipse.wst.common.project.facet.core.xml

Nothing needs to be done here!

### org.eclipse.wst.validation.prefs

Nothing needs to be done here!

### MavenArchetypeTestServiceBS.biz

This file represents of course the business service. The following changes need to made in this file:

- The WSDL reference should use the dynamic project name and service name
  - Original: **MavenArchetypeTestService_1.0/wsdl/MavenArchetypeTestService**
  - Dynamic: **${artifactId}/wsdl/${serviceName}**
- The binding name should be dynamic
  - Will use the service name
  - Original: **MavenArchetypeTestServiceSOAP**
  - Dynamic: **${serviceName}SOAP**
- The namespace should be dynamic
  - Should use the dynamic project name
  - Original: **http://www.example.org/MavenArchetypeTestService/**
  - Dynamic: **http://www.example.org/${artifactId}/**
- The URI should be dynamic
  - Original: **http://SOA-TIER/soa-infra/services/default/MavenArchetypeTestService_1.0/MavenArchetypeTestService**
  - Dynamic: **http://SOA-TIER/soa-infra/services/default/${artifactId}/${serviceName}**

A fragment of the resulting business service:

    <xml-fragment xmlns:ser="http://www.bea.com/wli/sb/services" xmlns:tran="http://www.bea.com/wli/sb/transports" xmlns:http="http://www.bea.com/wli/sb/transports/http" xmlns:xsi="http://www.w3.    org/2001/XMLSchema-instance" xmlns:env="http://www.bea.com/wli/config/env">
      <ser:coreEntry isProxy="false" isEnabled="true">
        <ser:binding type="SOAP" isSoap12="false" xsi:type="con:SoapBindingType" xmlns:con="http://www.bea.com/wli/sb/services/bindings/config">
          <con:wsdl ref="${artifactId}/wsdl/${serviceName}"/>
          <con:binding>
            <con:name>${serviceName}SOAP</con:name>
            <con:namespace>http://www.example.org/${artifactId}/</con:namespace>
          </con:binding>
        </ser:binding>
        <ser:monitoring isEnabled="false">
          <ser:aggregationInterval>10</ser:aggregationInterval>
        </ser:monitoring>
        <ser:sla-alerting isEnabled="true">
          <ser:alertLevel>normal</ser:alertLevel>
        </ser:sla-alerting>
        <ser:ws-policy>
          <ser:binding-mode>wsdl-policy-attachments</ser:binding-mode>
        </ser:ws-policy>
      </ser:coreEntry>
      <ser:endpointConfig>
        <tran:provider-id>http</tran:provider-id>
        <tran:inbound>false</tran:inbound>
        <tran:URI>
          <env:value>http://SOA-TIER/soa-infra/services/default/${artifactId}/${serviceName}</env:value>
        </tran:URI>
        ...
      </ser:endpointConfig>
    </xml-fragment>

### MavenArchetypeTestServicePS.proxy

The proxy service will require the most work. Besides implementing **$artifactId** and **$serviceName** support must be implemented for multiple operations. This means repeating elements! First lets make sure the other variables are used:

- The WSDL reference should use the dynamic project name and service name
  - Original: **MavenArchetypeTestService_1.0/wsdl/MavenArchetypeTestService**
  - Dynamic: **${artifactId}/wsdl/${serviceName}**
- The binding name should be dynamic
  - Will use the service name
  - Original: **MavenArchetypeTestServiceSOAP**
  - Dynamic: **${serviceName}SOAP**
- The namespace should be dynamic
  - Should use the dynamic project name
  - Original: **http://www.example.org/MavenArchetypeTestService/**
  - Dynamic: **http://www.example.org/${artifactId}/**
- The **$operation** OSB variable should use the dollar variable (from the template)
  - Original: **$operation**
  - Dynamic: **${symbol_dollar}operation**
- The reference to the business service should use the dynamic project name and service name
  - Original: **MavenArchetypeTestService_1.0/business/MavenArchetypeTestServiceBS**
  - Dynamic: **${artifactId}/business/${serviceName}BS**

I want to URI to contain a slash between the service name and major version. However the major version is not defined as a variable. To do this we can use a little bit of Java code in the template to acquire the major version. By adding this a line at the top of the file a new variable called **majorVersion** will be available, but only in this file!

    #set( $majorVersion = $artifactId.replace($serviceName,"").replace("_","") )

The URI can be changed now using the new variable:

- Original: **/MavenArchetypeTestService/1.0**
- Dynamic: **/${serviceName}/${majorVersion}**

The branch name should contain the service name, but without the text **service**. Add the following to the top of the file to add the **name** variable to this file:

    #set( $name = $serviceName.replace("Service","") )

The branch name can be replaced now:

- Original: **MavenArchetypeTestBranchNode**
- Dynamic: **${name}BranchNode**

Next it is time for multiple operations. In the file **archetype-descriptor.xml** a variable call **operations** contains all the method names seperated by a space. Two pieces of XML need to be repeated for each operation:

- The element **pipeline** and child elements
- The element **branch** and child elements

Velocity provised the ability to loop over values by using the **#foreach** command. Add the **#foreach** command before the opening tag of both elements and the **#end** command after the closing tag. Within the **#foreach** Java code is used to splint the **operations** variable.

    #foreach($operation in $operations.split(" "))
    <con:pipeline type="request" name="${operation}PipelinePairNode_request">
    ...
    </con:pipeline>

    ...

    #foreach($operation in $operations.split(" "))
    <con:branch name="${operation}">
    ...
    </con:branch>

Within the for loop the variable **operation** will contain the operation name and should replace the original operation name:

- Original: NewOperation
- Dynamic: ${operation}

The resulting proxy file should look something like this (I have excluded most the static parts):

    #set( $symbol_pound = '#' )
    #set( $symbol_dollar = '$' )
    #set( $symbol_escape = '\' )
    #set( $majorVersion = $artifactId.replace($serviceName,"").replace("_","") )
    #set( $name = $serviceName.replace("Service","") )
    <xml-fragment xmlns:ser="http://www.bea.com/wli/sb/services" xmlns:tran="http://www.bea.com/wli/sb/transports" xmlns:env="http://www.bea.com/wli/config/env" xmlns:http="http://www.bea.com/wli/sb/    transports/http" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:con="http://www.bea.com/wli/sb/pipeline/config" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config"     xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/logging/config" xmlns:con4="http://www.bea.com/wli/sb/stages/routing/config">
      <ser:coreEntry isProxy="true" isEnabled="true">
        <ser:binding type="SOAP" isSoap12="false" xsi:type="con:SoapBindingType" xmlns:con="http://www.bea.com/wli/sb/services/bindings/config">
          <con:wsdl ref="${artifactId}/wsdl/${serviceName}"/>
          <con:binding>
            <con:name>${serviceName}SOAP</con:name>
            <con:namespace>http://www.example.org/${artifactId}/</con:namespace>
          </con:binding>
          <con:selector type="SOAP body"/>
        </ser:binding>
        ...
      </ser:coreEntry>
      <ser:endpointConfig>
        <tran:provider-id>http</tran:provider-id>
        <tran:inbound>true</tran:inbound>
        <tran:URI>
          <env:value>/${serviceName}/${majorVersion}</env:value>
        </tran:URI>
        ...
      </ser:endpointConfig>
      <ser:router errorHandler="_onErrorHandler-6838065459772875562--48275a7.152591420ee.-7ff4">
        <con:pipeline type="error" name="_onErrorHandler-6838065459772875562--48275a7.152591420ee.-7ff4">
          <con:stage name="ErrorLog">
            <con:context/>
            <con:actions>
              <con1:ifThenElse>
                <con2:id>_ActionId-6838065459772875562--48275a7.152591420ee.-7eef</con2:id>
                <con1:case>
                  ...
                  <con1:actions>
                    <con3:log>
                      ...
                      <con3:expr>
                        <con2:xqueryText>concat('Unknown operation ',${symbol_dollar}operation,' called')</con2:xqueryText>
                      </con3:expr>
                    </con3:log>
                  </con1:actions>
                </con1:case>
                <con1:default>
                  <con3:log>
                    ...
                    <con3:expr>
                      <con2:xqueryText>concat('Fault in ',${symbol_dollar}operation,': ',fn-bea:serialize($fault))</con2:xqueryText>
                    </con3:expr>
                  </con3:log>
                </con1:default>
              </con1:ifThenElse>
              ...
            </con:actions>
          </con:stage>
        </con:pipeline>
        #foreach($operation in $operations.split(" "))
        <con:pipeline type="request" name="${operation}PipelinePairNode_request">
          <con:stage name="${operation}RequestLog">
            <con:context/>
            <con:actions>
              <con3:log>
                ...
                <con3:expr>
                  <con2:xqueryText>concat('Received request for ',${symbol_dollar}operation,': ',fn-bea:serialize($body))</con2:xqueryText>
                </con3:expr>
              </con3:log>
            </con:actions>
          </con:stage>
        </con:pipeline>
        <con:pipeline type="response" name="${operation}PipelinePairNode_response">
          <con:stage name="${operation}ResponseLog">
            <con:context/>
            <con:actions>
              <con3:log>
                ...
                <con3:expr>
                  <con2:xqueryText>concat('Received response for ',${symbol_dollar}operation,': ',fn-bea:serialize($body))</con2:xqueryText>
                </con3:expr>
              </con3:log>
            </con:actions>
          </con:stage>
        </con:pipeline>
        #end
        <con:pipeline type="request" name="DefaultPipelinePairNode_request">
          ...
        </con:pipeline>
        <con:pipeline type="response" name="DefaultPipelinePairNode_response"/>
        <con:flow>
          <con:branch-node type="operation" name="${name}BranchNode">
            <con:context/>
            <con:branch-table>
              #foreach($operation in $operations.split(" "))
              <con:branch name="${operation}">
                <con:operator>equals</con:operator>
                <con:value/>
                <con:flow>
                  <con:pipeline-node name="${operation}PipelinePairNode">
                    <con:request>${operation}PipelinePairNode_request</con:request>
                    <con:response>${operation}PipelinePairNode_response</con:response>
                  </con:pipeline-node>
                  <con:route-node name="${operation}RouteNode">
                    <con:context/>
                    <con:actions>
                      <con4:route>
                        <con2:id>_ActionId-6838065459772875562--48275a7.152591420ee.-7fad</con2:id>
                        <con4:service ref="${artifactId}/business/${serviceName}BS" xsi:type="ref:BusinessServiceRef" xmlns:ref="http://www.bea.com/wli/sb/reference"/>
                        <con4:operation>${operation}</con4:operation>
                        <con4:outboundTransform/>
                        <con4:responseTransform/>
                      </con4:route>
                    </con:actions>
                  </con:route-node>
                </con:flow>
              </con:branch>
              #end
              <con:default-branch>
                ...
              </con:default-branch>
            </con:branch-table>
          </con:branch-node>
        </con:flow>
      </ser:router>
    </xml-fragment>

### MavenArchetypeTestService.wsdl

The WSDL file should also contain multiple operations. The same construction as with the proxy file can be used here. A **#foreach** should be used in three places:

- Messages
  - Generate an input and an output message for each operation
- Port type
  - For each operation name use it and specify the input and output messages
- Biding
  - For each operation name use it and specify the action

Besides the loop the following other parts must be made dynamic:

- The namespace should be dynamic
  - Should use the dynamic project name
  - Original: **http://www.example.org/MavenArchetypeTestService/**
  - Dynamic: **http://www.example.org/${artifactId}/**
- The name should be dynamic
  - Will use the service name
  - Used in the attribute **name**, the binding name, port type name and the XSD file name
  - Original: **MavenArchetypeTestService**
  - Dynamic: **${serviceName}**

The resulting WSDL:

    #set( $symbol_pound = '#' )
    #set( $symbol_dollar = '$' )
    #set( $symbol_escape = '\' )
    <wsdl:definitions name="${serviceName}" targetNamespace="http://www.example.org/${artifactId}/" xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns:tns="http://www.example.org/${artifactId}/"     xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/" xmlns:xsd="http://www.w3.org/2001/XMLSchema">
      <wsdl:types>
        <xsd:schema>
          <xsd:import namespace="http://www.example.org/${artifactId}/" schemaLocation="../xsd/${serviceName}.xsd"/>
        </xsd:schema>
      </wsdl:types>
    
      #foreach($operation in $operations.split(" "))
      <wsdl:message name="${operation}Request">
        <wsdl:part element="tns:${operation}" name="parameters"/>
      </wsdl:message>
      <wsdl:message name="${operation}Response">
        <wsdl:part element="tns:${operation}Response" name="parameters"/>
      </wsdl:message>
      #end
    
      <wsdl:portType name="${serviceName}">
        #foreach($operation in $operations.split(" "))
        <wsdl:operation name="${operation}">
          <wsdl:input message="tns:${operation}Request"/>
          <wsdl:output message="tns:${operation}Response"/>
        </wsdl:operation>
        #end
      </wsdl:portType>
    
      <wsdl:binding name="${serviceName}SOAP" type="tns:${serviceName}">
        <soap:binding style="document" transport="http://schemas.xmlsoap.org/soap/http"/>
        #foreach($operation in $operations.split(" "))
        <wsdl:operation name="${operation}">
          <soap:operation soapAction="http://www.example.org/${artifactId}/${operation}"/>
          <wsdl:input>
            <soap:body use="literal"/>
          </wsdl:input>
          <wsdl:output>
            <soap:body use="literal"/>
          </wsdl:output>
        </wsdl:operation>
        #end
      </wsdl:binding>
    </wsdl:definitions>

### MavenArchetypeTestService.xsd

Besides a dynamic namespace an input and output element should be generated for each operation:

    #set( $symbol_pound = '#' )
    #set( $symbol_dollar = '$' )
    #set( $symbol_escape = '\' )
    <xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema" targetNamespace="http://www.example.org/${artifactId}/" xmlns:tns="http://www.example.org/${artifactId}/" elementFormDefault="qualified">
      #foreach($operation in $operations.split(" "))
      <xsd:element name="${operation}">
        <xsd:complexType>
          <xsd:sequence>
            <xsd:element name="in" type="xsd:string"/>
          </xsd:sequence>
        </xsd:complexType>
      </xsd:element>
      <xsd:element name="${operation}Response">
        <xsd:complexType>
          <xsd:sequence>
            <xsd:element name="out" type="xsd:string"/>
          </xsd:sequence>
        </xsd:complexType>
      </xsd:element>
      #end
    </xsd:schema>

### .project

The last file is a small file again and should the variable **artifactId** for the **name** element:

    #set( $symbol_pound = '#' )
    #set( $symbol_dollar = '$' )
    #set( $symbol_escape = '\' )
    <projectDescription>
      <name>${artifactId}</name>
      ...
    </projectDescription>

### pom.xml

Nothing needs to be done here!

## Step 5 - Modify file names to use variables

Several file names should contain a dynamic value. Specifically the file name of the business service, proxy service, WSDL and XSD. Using variables as file names is possible as long as the variables are defined in the file **archetype-metadata.xml**. Instead of a dollar sign, the variable must be prefixed and suffixed by two underscores.

- Business service:
  - Original: **MavenArchetypeTestServiceBS.biz**
  - Dynamic: **__serviceName__BS.biz**
- Proxy service:
  - Original: **MavenArchetypeTestServicePS.proxy**
  - Dynamic: **__serviceName__PS.proxy**
- WSDL:
  - Original: **MavenArchetypeTestService.wsdl**
  - Dynamic: **__serviceName__.wsdl**
- XSD:
  - Original: **MavenArchetypeTestService.xsd**
  - Dynamic: **__serviceName__.xsd**

## Step 6 - Test it!

At last it is time to test the archetype. First we must build the archetype using Maven:

    $ mvn clean install

Next we can generate a project with the archetype:

    $ mvn archetype:generate -DarchetypeCatalog=local -DarchetypeGroupId=nl.syntouch.archetypes -DarchetypeArtifactId=example-osb11g-archetype

Maven will now ask for all the variables:

    Define value for property 'groupId': : io.github.ninckblokje.blog
    Define value for property 'artifactId': : ForGitHubService_1.0
    Define value for property 'version':  1.0-SNAPSHOT: :
    Define value for property 'package':  io.github.ninckblokje.blog: :
    Define value for property 'operations': : hello world
    Define value for property 'serviceName': : ForGitHubService
    Confirm properties configuration:
    groupId: io.github.ninckblokje.blog
    artifactId: ForGitHubService_1.0
    version: 1.0-SNAPSHOT
    package: io.github.ninckblokje.blog
    operations: hello world
    serviceName: ForGitHubService

Voila an OSB project has been generated and is ready for import!
