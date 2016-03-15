---
layout: post
title:  "Oracle Cloud Adapter SDK - Part 1: Installation"
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

This is the first part of a small series of blogs regarding my effort and will cover the installation proces. Niall Commiskey already has a great [blog post](http://niallcblogs.blogspot.nl/2015/06/408-first-steps-with-cloud-adapter-sdk.html) about installing the Cloud Adapters, however I ran into several problems so I decided to write a post of my own describing my installation steps. I will use Windows 10 and PowerShell (my favourite Windows shell!) for these blogs.

Parts:

- Part 1: Installation
- Part 2
- Who knows!

Just a small warning: Always keep track of Oracle license information and the Oracle certification matrix!

## Installation

I already have JDeveloper SOA Suite 12.1.3 QuickStart installed without a developer domain. Now the following files need to be downloaden from [Oracle](http://www.oracle.com/technetwork/middleware/adapters/downloads/index.html):

- ofm_adapters_application_generic_12.1.3.0.0_disk1_1of2.zip
- ofm_adapters_application_generic_12.1.3.0.0_disk1_2of2.zip
- fmw_12.1.3.0.1_cloud_adapters_Disk1_1of1.zip

### ofm_adapters_application_generic_12.1.3.0.0

The ZIP files contain the installers for several adapters for example Siebel. Unzip both files in a temporary directory. This will unzip 5 files for Windows, Linux, Solaris, HP-UX and AIX.

I run Windows so I will start the Windows installer **iwora12c_application-adapters_win.exe**. Unfortunately because I run Windows 10 combined with Java 8 the installer **iwora12c_application-adapters_win.exe** refuses to start. It showed the following error:

Invocation of this Java Application has caused an InvocationTargetException. This application will now exit. (LAX)

~~~~~~~~
Stack Trace:
ZeroGu2: Windows DLL failed to load
        at ZeroGa2.b(DashoA10*..)
        at ZeroGa2.b(DashoA10*..)
        at com.zerog.ia.installer.LifeCycleManager.b(DashoA10*..)
        at com.zerog.ia.installer.LifeCycleManager.a(DashoA10*..)
        at com.zerog.ia.installer.Main.main(DashoA10*..)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(Unknown Source)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(Unknown Source)
        at java.lang.reflect.Method.invoke(Unknown Source)
        at com.zerog.lax.LAX.launch(DashoA10*..)
        at com.zerog.lax.LAX.main(DashoA10*..)
~~~~~~~~

I needed the following tricks to make the installer work (don't forget to create a backup!):

- Windows 7 compatiblity mode
- Administrator rights
- Replace **C:\Windows\System32\java.exe** with a Java 7 version
- Set the Registry value of **HKEY_LOCAL_MACHINE\SOFTWARE\JavaSoft\Java Runtime Environment** **CurrentVersion** to **1.7**

Now the installer will start and I need to select the directory where I have installed JDeveloper 12.1.3, in my case this is **c:\oracle\jdeveloper\jdevstudio1213**. After this the adapters will be installed.

### fmw_12.1.3.0.1_cloud_adapters

This ZIP contains the Cloud Adapters themself and the SDK. They are supplied as a patch which can be applied using OPatch. However SOA Suite 12.1.3.0.1 is required before installing the Cloud Adapters so download patch 19707784 from [Oracle Support](https://support.oracle.com).

Before applying the pathes, make sure the environment is setup correctly (and every command is executed as Administrator):

- ORACLE_HOME must be set to the JDeveloper installation directory
- OPatch should be placed on the path

~~~~~~~~
$ $env:ORACLE_HOME="C:\oracle\jdeveloper\jdevstudio1213\"
$ $env:PATH="$env:ORACLE_HOME\OPatch;$env:PATH"
~~~~~~~~

Unzip **p19707784_121300_Generic.zip** to a directory and then goto the subdirectory **19707784**. Execute `opatch apply -jre [JDK7_DIR]`

Unzip the other two files (**p20680367_121301_Generic.zip** and **p20780464_121300_Generic.zip**) and in the subdirectories **20680367** and **20780464** also execute `opatch apply -jre [JDK7_DIR]`

Since the Cloud Adapter patches are not compatible with SOA Suite 12.1.3.0.2 and 12.1.3.0.3 I won't be upgrading to these versions.

## Starting JDeveloper 12.1.3

After installing the adapters and patches, the new Cloud Adapters were not visible in JDeveloper. For them to become visible, you must execute JDeveloper as Adminstrator! After starting JDeveloper correctly the Cloud Adapters will become visible on the right side under **Components**:

![Cloud adapters installed]({{ site.github.url }}/assets/cloud-adapter-sdk-part1/cloud_adapters_installed.png)

## Domain setup

Several steps need to be done before the Cloud Adapters have been installed successfully:

- Deploy **cloudsdk.ear**
- Grant permissions to credential store
- Create CSF map
- Create CSF keys

The  [Oracle documentation](http://docs.oracle.com/middleware/1213/cloudadapter-rightnow/OCAIG/GUID-57BCEAD7-F1A7-4EC8-A93B-CD05A157934C.htm) also describes these steps.

### Deploy **cloudsdk.ear**

- Login to the WebLogic console and goto **Deployments**
  - If required start an edit session.
- Click on **Install**, **upload your file(s)** and then on **Choose File** under **Deployment Archive**
  - Select the file **cloudsdk.ear** in the directory **$ORACLE_HOME/soa/soa/modules/oracle.cloud.adapter_12.1.3**
- Accept all default values and finish the wizard
  - If required commit the edit session

![cloudsdk.ear]({{ site.github.url }}/assets/cloud-adapter-sdk-part1/cloudsdk.ear.png)

### Grant permissions to credential store

- Login to Enterprise Manager
- Open **System Policies** by right clicking on **WebLogic Domain**, [DOMAIN_NAME]
![System Policies]({{ site.github.url }}/assets/cloud-adapter-sdk-part1/system_policies.png)
- Search for type **Codebase** with a name which includes **jca**
- Select the first hit: **file:${soa.oracle.home}/soa/modules/oracle.soa.adapter_11.1.1/jca-binding-api.jar**
![Search for jca]({{ site.github.url }}/assets/cloud-adapter-sdk-part1/grant_search.png)
- Click on **Edit** and **Add** and use the following values:
  - Select here to enter details for a new permission
  - Permission Class: oracle.security.jps.service.credstore.CredentialAccessPermission
  - Resource Name: context=SYSTEM,mapName=oracle.wsm.security,keyName=*
  - Permission Actions: *
![Add permission]({{ site.github.url }}/assets/cloud-adapter-sdk-part1/add_permission.png)
- Press **OK** twice

### Create CSF map

- Login to Enterprise Manager
- Open **Credentials** by right clicking on **WebLogic Domain**, [DOMAIN_NAME]
![Credentials]({{ site.github.url }}/assets/cloud-adapter-sdk-part1/credentials.png)
- Click on **Create Map** and use **oracle.wsm.security** as name
- Press **OK**

### Create CSF keys

For now I won't by adding any CSF keys.

## JDeveloper library

I will create a new JDeveloper library which contains all the required dependencies. I can use this library in multiple JDeveloper projects so I can easily include all the required dependencies for a Cloud Adapter. **$ORACLE_HOME** points to the JDeveloper installation directory.

- Goto **Tools**, **Manage Libraries**
- Select **User** and press **New...**
- Use as name: **Oracle Cloud Adapter SDK**
- Add a new entry for each of the following JAR's:
  - $ORACLE_HOME/soa/soa/modules/oracle.cloud.adapter_12.1.3/cloud-designtime-api.jar
  - $ORACLE_HOME/soa/soa/modules/oracle.cloud.adapter_12.1.3/cloud-designtime-impl.jar
  - $ORACLE_HOME/soa/soa/modules/oracle.cloud.adapter_12.1.3/cloud-runtime-api.jar
  - $ORACLE_HOME/soa/soa/modules/oracle.cloud.adapter_12.1.3/cloud-runtime-impl.jar
  - $ORACLE_HOME/soa/soa/modules/oracle.cloud.adapter_12.1.3/oracle.tools.cloud.adapter.sdk.jar
  - $ORACLE_HOME/soa/soa/modules/oracle.cloud.adapter_12.1.3/oracle.tools.uiobjects.sdk.jar
  - $ORACLE_HOME/soa/plugins/jdeveloper/extensions/oracle.sca.modeler.jar
  - $ORACLE_HOME/soa/plugins/jdeveloper/extensions/oracle.sca.ui.adapters.jar
  - $ORACLE_HOME/soa/plugins/jdeveloper/extensions/oracle.tools.cloud.adapter.ide.jar
  - $ORACLE_HOME/jdeveloper/jlib/jewt4.jar
  - $ORACLE_HOME/wlserver/plugins/maven/com/oracle/weblogic/javax.resource_1.7.0.jar
  - $ORACLE_HOME/jdeveloper/ide/extensions/oracle.ide.jar
  - $ORACLE_HOME/oracle_common/modules/com.oracle.webservices.orawsdlapi_12.1.3.jar
  - $ORACLE_HOME/oracle_common/modules/oracle.xdk_12.1.3/xmlparserv2.jar
- Press **OK** twice

![Cloud Adapter SDK library]({{ site.github.url }}/assets/cloud-adapter-sdk-part1/cloud_adapter_sdk_library.png)
