---
layout: post
title:  "More chaos as an OWSM policy"
tags:
- oracle
- soa suite
- osb
- owsm
disqus: true
---
## Errors & failures

So it is very easy to implement a happy flow, however handling errors, rolling back transactions and recovering from errors and failures can be quite challenging. It is not possible to find all the possibilities during development or design. Some errors or failures will only occur when very rare circumstances come into play together. Then all the parts of the application either will handle the error and will ensure that no data is lost and the application survices or not. In the last case people lose data, applications crash and managers get upset.

A very good read on this topic is the book [Release It!](https://pragprog.com/book/mnee/release-it).

## A policy

Most of the time I write services in either Oracle Service Bus or Oracle SOA Suite. I can mock expected error behaviour, however sometimes having errors when you don't expect them can give you new inside into the stability and resilience of the application.

To create (unexpected) errors a Managed Server can be stopped, data sources can be removed or entire virtual machine's can be deleted. However these Managed Servers are quite heavy and when I ask somebody if I can break something during a test I am usually asked to get a cup of coffee ;)

So I wanted a different method (unfortunately not implemeted at a customer) so I created an Oracle Web Service Manager (OWSM) policy. I was inspired by the [Chaos Monkey](http://techblog.netflix.com/2012/07/chaos-monkey-released-into-wild.html) application made by Netflix. The Chaos Monkey application creates havoc. The OWSM policy should also create problems, but in a very modest way. It generates an error on a random basis.

The OWSM framework is not really meant for this kind of policies, but it is a start! The sources can be found on my [GitHub](https://github.com/ninckblokje/ChaosPolicy) repository.

### Building & installing the policy

The policy is build using JDeveloper. I build it using JDeveloper 12.2.1.1, but I think it can be back ported to 12.1.3. There are two deployment profiles:

![Deployment profiles]({{ site.github.url }}/assets/more-chaos-as-an-owsm-policy/deployment_profiles.png)

The deployment profile **ChaosPolicyJar** creates a JAR file called **ChaosPolicyJar.jar** which should be copied to the **lib** directory of the domain (and restart it afterwards). The deployment profile **ChaosPolicyConfig** creates a ZIP file called **ChaosPolicyConfig.zip**. This file is used by Enterprise Manager to register the policy.

![Registration, part 1]({{ site.github.url }}/assets/more-chaos-as-an-owsm-policy/register_1.png)

![Registration, part 2]({{ site.github.url }}/assets/more-chaos-as-an-owsm-policy/register_2.png)

### Attaching the policy

The policy can be attached to both exposed services and external references.

![Attach]({{ site.github.url }}/assets/more-chaos-as-an-owsm-policy/attach_policy.png)

The default change for chaos is 1 in 8. This configuration can be overruled for each attachment.

![Configure]({{ site.github.url }}/assets/more-chaos-as-an-owsm-policy/configure_policy.png)

The setting can also be overruled using the environment variable **CHAOS_POLICY_CHANGE**. However the environment setting will precedence take over any other configuration.

### Action!

The error can come from both the request, response or fault handling in the OWSM policy. Several example error messages:

~~~~~~~~xml
<env:Envelope xmlns:env="http://schemas.xmlsoap.org/soap/envelope/">
   <env:Header/>
   <env:Body>
      <env:Fault xmlns:ns0="http://schemas.oracle.com/owsm/policy-enforcement-2007-06">
         <faultcode>ns0:GenericFault</faultcode>
         <faultstring>GenericFault : generic error</faultstring>
         <faultactor/>
      </env:Fault>
   </env:Body>
</env:Envelope>
~~~~~~~~

~~~~~~~~xml
<env:Envelope xmlns:env="http://schemas.xmlsoap.org/soap/envelope/">
   <env:Header>
      <tracking:faultId xmlns:tracking="http://oracle.soa.tracking.core.TrackingProperty">20004</tracking:faultId>
   </env:Header>
   <env:Body>
      <env:Fault xmlns:ns0="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd">
         <faultcode>ns0:InvalidSecurity</faultcode>
         <faultstring>InvalidSecurity : error in processing the WS-Security security header</faultstring>
         <faultactor/>
      </env:Fault>
   </env:Body>
</env:Envelope>
~~~~~~~~

~~~~~~~~xml
<env:Envelope xmlns:env="http://schemas.xmlsoap.org/soap/envelope/">
   <env:Header>
      <tracking:faultId xmlns:tracking="http://oracle.soa.tracking.core.TrackingProperty">20005</tracking:faultId>
   </env:Header>
   <env:Body>
      <env:Fault>
         <faultcode>env:Server</faultcode>
         <faultstring>oracle.fabric.common.FabricInvocationException: Unable to invoke endpoint URI "http://localhost:7101/soa-infra/services/default/EchoService!1.0*soa_997fd0ad-3705-4398-b013-12a7bbb2f092/EchoService" successfully due to: oracle.fabric.common.PolicyEnforcementException: 1719cae2-f632-49ea-934a-e874ee6c8c36</faultstring>
         <faultactor/>
         <detail>
            <exception>Unable to invoke endpoint URI "http://localhost:7101/soa-infra/services/default/EchoService!1.0*soa_997fd0ad-3705-4398-b013-12a7bbb2f092/EchoService" successfully due to: oracle.fabric.common.PolicyEnforcementException: 1719cae2-f632-49ea-934a-e874ee6c8c36</exception>
         </detail>
      </env:Fault>
   </env:Body>
</env:Envelope>
~~~~~~~~

In my test I configured the change for the OWSM policy attached to the exposed service at 1 in 2. The change for the external reference OWSM policy is configured at 1 in 8. Just to be certain, sometimes an actual response is coming!

~~~~~~~~xml
<env:Envelope xmlns:env="http://schemas.xmlsoap.org/soap/envelope/" xmlns:wsa="http://www.w3.org/2005/08/addressing">
   <env:Header>
      <wsa:MessageID>urn:1458a3fd-46e4-11e6-a109-606720b7b1f0</wsa:MessageID>
      <wsa:ReplyTo>
         <wsa:Address>http://www.w3.org/2005/08/addressing/anonymous</wsa:Address>
         <wsa:ReferenceParameters>
            <instra:tracking.ecid xmlns:instra="http://xmlns.oracle.com/sca/tracking/1.0">6ebb4c8b-2228-4c1c-a509-56f87aa45bcd-0000027f</instra:tracking.ecid>
            <instra:tracking.FlowEventId xmlns:instra="http://xmlns.oracle.com/sca/tracking/1.0">30391</instra:tracking.FlowEventId>
            <instra:tracking.FlowId xmlns:instra="http://xmlns.oracle.com/sca/tracking/1.0">30033</instra:tracking.FlowId>
            <instra:tracking.CorrelationFlowId xmlns:instra="http://xmlns.oracle.com/sca/tracking/1.0">0000LNMfc0pDg^WzLwyGOA1NWec400000Z</instra:tracking.CorrelationFlowId>
            <instra:tracking.quiescing.SCAEntityId xmlns:instra="http://xmlns.oracle.com/sca/tracking/1.0">10004</instra:tracking.quiescing.SCAEntityId>
         </wsa:ReferenceParameters>
      </wsa:ReplyTo>
      <wsa:FaultTo>
         <wsa:Address>http://www.w3.org/2005/08/addressing/anonymous</wsa:Address>
      </wsa:FaultTo>
   </env:Header>
   <env:Body>
      <singleString xmlns:sin="http://xmlns.oracle.com/singleString" xmlns="http://xmlns.oracle.com/singleString">test</singleString>
   </env:Body>
</env:Envelope>
~~~~~~~~

### Finding logging

The OWSM policy generates some logging through the ADF Logging framework. It can be enabled from Enterprise Manager, though I find it quite hard to find the right application on which to configure the logging.

![Logging, part 1]({{ site.github.url }}/assets/more-chaos-as-an-owsm-policy/finding_odl_1.png)

![Logging, part 2]({{ site.github.url }}/assets/more-chaos-as-an-owsm-policy/finding_odl_2.png)
