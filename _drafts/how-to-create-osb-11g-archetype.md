---
layout: post
title:  "How to create an OSB 11g Archetype"
tags:
- osb
- maven
---

In my first post ([OSB 11g Maven Archetype]({% post_url 2016-02-09-osb-11g-archetype %})) I wrote about the example I made for an OSB 11g Maven archetype.
Now I want to show how to create an archetype. Creating an OSB archetype consist out of the following steps:

1. Create an OSB project
2. Create an archetype out of an existing OSB project
3. Add variables
4. Modify files to use variables (create template file)
5. Modify file names to use variables
6. Test it!

## What should it do

I want the archetype to be able to create an OSB project with a proxy service and a business service (which points to a SOA Suite URL). The name of the project
can be split into two parts:

- Name
- Major version 

The name (without the major version) should also be used as file name for the proxy and business services.

## Step  1 - Create an OSB project

It is important to keep the OSB project as simple as possible, since we need to manually create an template out of it later. My OSB project only has the required parts:

- Proxy service
- Business service
- WSDL and XSD
- Folder structure

![Project structure]({{ site.github.url }}/assets/how-to-create-osb-11g-archetype/project_structure.png)

The proxy service has got only one operation (though with an operational branch). The default operation throughts an error and one operation has been implemented with a pipeline pair containing a log step and a routing.

![Pipeline]({{ site.github.url }}/assets/how-to-create-osb-11g-archetype/pipeline.png)