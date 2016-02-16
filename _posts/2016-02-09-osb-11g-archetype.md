---
layout: post
title:  "OSB 11g Maven Archetype"
tags:
- osb
- maven
---

## Introduction
In Oracle Service Bus 12c it is possible to use templates. These templates can contain all reusable pieces of an OSB proxy service. However in 11g this feature is not available. At a customer I am currently working, the OSB proxy services are quite similar. Only a couple are really different.

Maven provides a mechanism for creating projects from a template called archetypes. Although Service Bus 11g does not use Maven, we can still use Maven archetypes to create a project from a template.

Since every customer has its own unique set of requirements for OSB proxy services, I have created an example project which shows the Maven archetype.

The entire example project is placed under [Apache 2.0](https://gitlab.com/syntouch/example-osb11g-archetype/blob/master/LICENSE) license at the [GitLab](https://gitlab.com/syntouch/example-osb11g-archetype) of SynTouch (my employer).

## Sample Service Bus project
I based the template on a dummy OSB project. In real life it is off course possible to use an existing OSB project. My projectname consists of two parts [SERVICE_NAME]_[MAJOR_VERSION].

The WSDL holds one operation, the request and response elements are defined in a separate XSD:

![WSDL]({{ site.github.url }}/assets/osb-11g-archetype/wsdl.png)

I have the following components in my OSB project:

- Business service
- Proxy service
- WSDL
- XSD

![Project structure]({{ site.github.url }}/assets/osb-11g-archetype/project_structure.png)

My proxy service has the following setup:

![Pipeline]({{ site.github.url }}/assets/osb-11g-archetype/pipeline.png)

Each request and response will be logged and some error handling is in place. The business service points to a SOA composite (which I have not implemented in this case).

## Maven archetype
Maven archetypes use the Apache Velocity template engine to parse files. It is not required for files to be parsed by Velocity, but in this case we want to replace values in the files.

A new project can be generated using the archetype with the following command, Maven will ask for all the required settings including custom variables.

    $ mvn archetype:generate -DarchetypeCatalog=local -DarchetypeGroupId=nl.syntouch.archetypes -DarchetypeArtifactId=example-osb11g-archetype
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

This will generate a new OSB project called ForGitHubService_1.0 with two operations: hello and world.

![New OSB project]({{ site.github.url }}/assets/osb-11g-archetype/new_project.png)

The following paragraphs will provide some more in-depth details.

### Archetype structure
An archetype is a Maven project with the following structure:

- pom.xml
- src
  - main
    - resources
      - archetype-resources
      - META-INF
        - maven
          - archetype-metadata.xml

The folder **archetype-resources** contains all the (template) files, in this case the template OSB project. The file **archetype-metadata.xml** describes how to handle these files and custom variables. It is possible to define custom variables in a template or in **archetype-metadata.xml**. However variables in **archetype-metadata.xml** should not reference each other!

Here is a part of my **archetype-metadata.xml**:

    ...
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
        ...
        <fileSet>
          <directory>xquery</directory>
        </fileSet>
      </fileSets>
    ...

I specify two required properties (**serviceName** and **operations**) and specify multiple file sets. If a file set is filtered (**filtered="true"**) then the file is parsed by the Velocity template engine.

### Velocity templates
Almost all of the files will be processed by Velocity in order to replace names, namespaces and go through for loops for repeating parts for each operation. It is usefull to specify several special characters above your template like $, # and \\ as variable. Velocity uses these characters so by using the variables it is still possible to use the characters.

    #set( $symbol_pound = '#' )
    #set( $symbol_dollar = '$' )
    #set( $symbol_escape = '\' )

Any custom variables used throughout the entire template should also be defined above your template.

    #set( $majorVersion = $artifactId.replace($serviceName,"").replace("_","") )
    #set( $name = $serviceName.replace("Service","") )

Now it is possible to use these variables and the variables from **archetype-metadata.xml** in the template. Since a part of my proxy will be repeated for each operation I have added a for each loop to my template:

    #foreach($operation in $operations.split(" "))
    <con:pipeline type="request" name="${operation}PipelinePairNode_request">
      <con:stage name="${operation}RequestLog">
        <con:context/>
        <con:actions>
          <con3:log>
            <con2:id>_ActionId-6838065459772875562--48275a7.152591420ee.-7f5f</con2:id>
            <con3:logLevel>debug</con3:logLevel>
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
            <con2:id>_ActionId-6838065459772875562--48275a7.152591420ee.-7f82</con2:id>
            <con3:logLevel>debug</con3:logLevel>
            <con3:expr>
              <con2:xqueryText>concat('Received response for ',${symbol_dollar}operation,': ',fn-bea:serialize($body))</con2:xqueryText>
            </con3:expr>
          </con3:log>
        </con:actions>
      </con:stage>
    </con:pipeline>
    #end

The **operations** variable is split and for each value the values between the tags **#foreach** and **#end** are repeated.

### Renaming files
It is possible to give files a dynamic name (though it is not possible to create a dynamic number of files). By placing two underscores in front of the variable name (as defined in **archetype-metadata.xml**) and two after the name is dynamically set when the archetype is executing. Example syntax:

    __serviceName__PS.proxy

If the variable **serviceName** is set to **ForGitHubService** then the final name of the proxy service file will be:

    ForGitHubServicePS.proxy

### Building
The archetype can be build by Maven with the following command:

    $ mvn clean install
