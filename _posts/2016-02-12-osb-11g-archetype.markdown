---
layout: post
title:  "OSB 11g Maven Archetype"
date:   2016-02-09 14:41:00 +0100
categories: jekyll update
---

## Introduction
In Oracle Service Bus 12c it is possible to use templates. These templates can contain all reusable pieces of an OSB proxy service. However in 11g this feature is not available. At a customer where I am currently working our OSB proxy services are quite similar. Only a couple are really different.

Maven provides a mechanism for creating projects from a template called archetypes. Although Service Bus 11g does nog use Maven, we can still use Maven archetypes to create a project from a template.

Since every customer has its own unique set of requirements for OSB proxy services, I have created an example project which shows the Maven archetype.

## Sample Service Bus project
I based the template on a dummy OSB project. In real life it is off course possible to use an existing OSB project.

The WSDL holds one operation, the request and response elements are defined in a separate XSD:

![WSDL]({{ site.github.url }}/assets/2016-02-12-osb-11g-archetype/wsdl.png)

I have the following components in my OSB project:

![Project structure]({{ site.github.url }}/assets/2016-02-12-osb-11g-archetype/project_structure.png)

My proxy service has the following setup:

![Pipeline]({{ site.github.url }}/assets/2016-02-12-osb-11g-archetype/pipeline.png)

Each request and response will be logged and some error handling is in place. The business service points to a SOA composite (which I have not implemented in this case).

## Maven archetype
Maven archetypes use the Apache Velocity templating engine to parse files. It is not a required for files to be parsed by Velocity, but in this case we want to replace values in the files.

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

The folder **archetype-resources** contains all the (template) files. The file **archetype-metadata.xml** describes how to handle these files and custom variables. It is possible to define custom variables in a template or in **archetype-metadata.xml**. However variables in **archetype-metadata.xml** should not reference each other!

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

I specify two requires properties (**serviceName** and **operations**) and specify multiple file sets. If a file set is filtered (**filtered="true"**) then the file is parsed by the Velocity template engine.

### Velocity templates
Almost all of the files will be processed by Velocity in order to replace names, namespaces and go through for loops for repeating parts for each operation. It is usefull to specify several special characters above your template like $, # and \\.

    #set( $symbol_pound = '#' )
    #set( $symbol_dollar = '$' )
    #set( $symbol_escape = '\' )