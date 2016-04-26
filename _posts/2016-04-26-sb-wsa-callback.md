---
layout: post
title:  "WS-Addressing callback interception using Service Bus"
tags:
- oracle
- osb
- service bus
- ws-addressing
- callback
disqus: true
---

Maarten Smeets posted a nice [article](https://technology.amis.nl/2016/02/28/asynchronous-interaction-in-oracle-bpel-and-bpm-ws-addressing-and-correlation-sets/) describing asynchronous interaction in BPEL and BPM using WS-Addressing and correlation sets. I wanted to use Service Bus 12c to force the callback to go through a Service Bus proxy instead of going directly to the consumer. This can be useful when the target service should not have the ability to call services outsite of its trusted domain or network.

## WS-Addressing
WS-Addressing provides a way for message routing and correlation using SOAP headers. It is an official specification. Service Bus, SOA Suite, JAX-WS and quite a lot of other frameworks and tools support it one way or the other. There are two versions of WS-Addressing: 200508 and 200408.

In an synchronous interaction the following headers are important:

- Request:
  - ReplyTo
  - MessageId
- Callback:
  - RelatesTo

In the request the **ReplyTo** specifies the address where the reply must be send to (or **http://www.w3.org/2005/08/addressing/anonymous** if no reply address is specified). The **MessageId** headers contains a unique id which can be used to to correlate the request to a future callback. In the callback the **RelatesTo** header contains the original **MessageId**. There are others headers for example **Action** which are important, but for creating an asynchronous interaction **ReplyTo**, **MessageId** and **RelatesTo** are all that are required.

SOA Suite 11g and 12c support WS-Addressing (don't mention SOA Suite 10g...) and provide out of the box correlation support. Using BPEL no code is required to use WS-Addressing for correlation (both as consumer and as producer).

## Test scenario
A SOA composite will implement an asynchronous webservice (a fire & forget request followed by a future fire & forget callback). It will be exposed using a Service Bus project which will 'hide' the implementation for the outside world. For example the SOA composite might be running in a local domain, while the Service Bus project migt be running in a SOA CS cloud instance. This will allow to SOA composite to only receive traffic from 'trusted' networks and the Service Bus will handle all security.

This will be the test setup, where SoapUI will play the external consumer:

![Setup]({{ site.github.url }}/assets/sb-wsa-callback/setup.png)

## Abstract WSDL
The [abstract WSDL](https://github.com/ninckblokje/callback-wrapper-sb12c/blob/master/CallbackWrapperSOA/CallbackWrapperService/SOA/WSDLs/CallbackWrapperProcess.wsdl) is really simple. It contains a message for the request and a message for the callback, two port types (one for the request and one for the callback) and a partnerlink specifying the roles. No WS-Addressing information is included here!

~~~~~~~~
<wsdl:definitions name="CallbackWrapperProcess"
             targetNamespace="http://xmlns.oracle.com/CallbackWrapperSOA/CallbackWrapperService/CallbackWrapperProcess"
             xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/"
             xmlns:client="http://xmlns.oracle.com/CallbackWrapperSOA/CallbackWrapperService/CallbackWrapperProcess"
             xmlns:plnk="http://docs.oasis-open.org/wsbpel/2.0/plnktype">

	...

	<wsdl:portType name="CallbackWrapperProcess">
		<wsdl:operation name="process">
			<wsdl:input message="client:CallbackWrapperProcessRequestMessage"/>
		</wsdl:operation>
	</wsdl:portType>

	<wsdl:portType name="CallbackWrapperProcessCallback">
		<wsdl:operation name="processResponse">
			<wsdl:input message="client:CallbackWrapperProcessResponseMessage"/>
		</wsdl:operation>
	</wsdl:portType>

	<plnk:partnerLinkType name="CallbackWrapperProcess">
		<plnk:role name="CallbackWrapperProcessProvider" portType="client:CallbackWrapperProcess"/>
		<plnk:role name="CallbackWrapperProcessRequester" portType="client:CallbackWrapperProcessCallback"/>
	</plnk:partnerLinkType>
</wsdl:definitions>
~~~~~~~~

## SOA Suite project
The SOA composite contains a BPEL 2.0 component implementing the abstract WSDL. For this scenario I create a really simple BPEL 2.0 component, but I did include a wait activity to simulate some time consuming background work. The BPEL 2.0 component starts with a receive and ends with an invoke of the original partnerlink. The roles specified in the abstract WSDL define which message is send through wich port type.

![BPEL 2.0]({{ site.github.url }}/assets/sb-wsa-callback/bpel.png)

## Concrete WSDL
After deploying the SOA composite SOA Suite generates the concrete WSDL. This WSDL contains a lot more information, including WS-Addressing information!

~~~~~~~~
<wsdl:definitions name="CallbackWrapperProcess" targetNamespace="http://xmlns.oracle.com/CallbackWrapperSOA/CallbackWrapperService/CallbackWrapperProcess" xmlns:client="http://xmlns.oracle.com/CallbackWrapperSOA/CallbackWrapperService/CallbackWrapperProcess" xmlns:plnk="http://docs.oasis-open.org/wsbpel/2.0/plnktype" xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">

	...

	<wsp:Policy wsu:Id="wsaddr_policy" xmlns:wsp="http://schemas.xmlsoap.org/ws/2004/09/policy" xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd">
		<wsaw:UsingAddressing xmlns:wsaw="http://www.w3.org/2006/05/addressing/wsdl"/>
	</wsp:Policy>
	
	...

	<wsdl:binding name="CallbackWrapperProcessBinding" type="client:CallbackWrapperProcess">
		<soap:binding transport="http://schemas.xmlsoap.org/soap/http"/>
		<wsp:PolicyReference URI="#wsaddr_policy" wsdl:required="false" xmlns:wsp="http://schemas.xmlsoap.org/ws/2004/09/policy"/>
		<wsdl:operation name="process">
			<soap:operation soapAction="process" style="document"/>
			<wsdl:input>
				<soap:body use="literal"/>
			</wsdl:input>
		</wsdl:operation>
	</wsdl:binding>
	<wsdl:binding name="CallbackWrapperProcessCallbackBinding" type="client:CallbackWrapperProcessCallback">
		<soap:binding transport="http://schemas.xmlsoap.org/soap/http"/>
		<wsdl:operation name="processResponse">
			<soap:operation soapAction="processResponse" style="document"/>
			<wsdl:input>
				<soap:body namespace="http://xmlns.oracle.com/CallbackWrapperSOA/CallbackWrapperService/CallbackWrapperProcess" use="literal"/>
			</wsdl:input>
		</wsdl:operation>
	</wsdl:binding>
	<wsdl:service name="callbackwrapperprocess_client_ep">
		<wsdl:port binding="client:CallbackWrapperProcessBinding" name="CallbackWrapperProcess_pt">
			<soap:address location="http://localhost:7101/soa-infra/services/default/CallbackWrapperService/callbackwrapperprocess_client_ep"/>
		</wsdl:port>
	</wsdl:service>
</wsdl:definitions>
~~~~~~~~

Two bindings are generated, one for the request and one for the callback. The request binding contains a **PolicyReference** wich specifies WS-Addressing. Only one service element is generated, since the callback will be send back to the consumer. The consumer must expose an endpoint for that.

## Service Bus project

![Service Bus]({{ site.github.url }}/assets/sb-wsa-callback/sb.png)

The Service Bus project contains two proxy services, two business services and two pipelines. The first lane is for receiving the request from an external consumer, processing it and passing it on to the SOA composite. The second lane is for receiving the callback from the SOA composite, processing it and sending it to the correct external consumer.

### First lane for request

Now we want the Service Bus to receive the callback from the SOA composite so we need to change the WS-Addressing header **ReplyTo**. This header should contain a URL to the second proxy in this project. In the pipeline a stage contains the following logic:

- Retrieve protocol, hostname and port and store it in variable **sbHostUrl**

~~~~~~~~
fn:substring-before($inbound/ctx:transport/ctx:request/http:absolute-URI/text(),$inbound/ctx:transport/ctx:uri/text())
~~~~~~~~

- Add the path of the second proxy to **sbHostUrl** and store in variable **sbCallbackUrl**

~~~~~~~~
fn:concat($sbHostUrl,'/CallbackWrapperService/Proxy/CallbackWrapperCallbackService')
~~~~~~~~

- Replace the WS-Addressing header **ReplyTo** (actually the child element **Address**) with variable **sbCallbackUrl**

So now the SOA composite will receive the correct **ReplyTo** address for the callback. However the second lane handling the callback in Service Bus does not yet know where to send the callback to. There are several options to solve this problem, for example putting the original callback URL in Coherence and adding the Coherence key to the new callback URL. However I decided to encode the original callback URL and add it as a query parameter. The second lane can then retrieve the query parameter, decode it and send the callback to the decoded callback URL.

Before replacing the WS-Addressing header **ReplyTo** the following code should be added:

- Assign the original WS-Addressing header **ReplyTo** (actually the child element **Address**) to variable **clientReplyToUrl**
- Add a JavaScript message processing with the following code for encoding the URL
  - This is actually a Java based JavaScript engine in which we can call Java classes
  - Variables prefixed with **process.** are variables retrieved / stored in the pipeline
  - Notice the nice quirk of JDeveloper offering us browser based events!

~~~~~~~~ javascript
var clientReplyToUrl=new java.lang.String(process.clientReplyToUrl)
var encoder=java.util.Base64.getEncoder()
process.encodedReplyToUrl=encoder.encodeToString(clientReplyToUrl.getBytes('UTF-8'))
~~~~~~~~

![Service Bus]({{ site.github.url }}/assets/sb-wsa-callback/jdev_quirk.png)

- The encoded URL is stored in variable **encodedReplyToUrl**
- Instead of the previous replacement of **ReplyTo** the variables **sbCallbackUrl** will be concatenated with **encodedReplyToUrl** (as query parameter **clientCallbackUrl**)

~~~~~~~~
concat($sbCallbackUrl,'?clientCallbackUrl=',$encodedReplyToUrl)
~~~~~~~~

### Second lane for callback

The SOA composite is going to use the complete URL specified in the **ReplyTo** header for its callback. This means that we get the query parameter **clientCallbackUrl** back from the SOA composite. The pipeline should contain the following logic:

- Retrieve the value from the **clientCallbackUrl** query parameter and store it in the variable **encodedReplyToUrl**

Make sure to reply HTTP status code 202 to the external consumer.

~~~~~~~~
$inbound/ctx:transport/ctx:request/http:query-parameters/http:parameter[@name='clientCallbackUrl']/@value
~~~~~~~~

- Decode the variable **encodedReployToUrl** and store it using a JavaScript message processing

~~~~~~~~ javascript
var encodedReplyToUrl=new java.lang.String(process.encodedReplyToUrl)
var decoder=java.util.Base64.getDecoder()
process.clientReplyToUrl=new java.lang.String(decoder.decode(encodedReplyToUrl),'UTF-8')
~~~~~~~~

- Replace the **To** header with the decoded callback URL stored in variable **clientReplyTo**
- In the routing add a Routing Options and set **URI** to the variable **clientReplyTo**

Make sure to reply the HTTP status 200 back to the SOA composite, since HTTP status 202 is not really accepted by SOA Suite.

### Example URL's

- SoapUI
  - Request URL: ``http://localhost:7101/CallbackWrapperService/Proxy/CallbackWrapperService``
  - Callback URL: ``http://localhost:8080/CallbackTestSuite/Complete``
- Service Bus first lane
  - Request URL: ``http://localhost:7101/soa-infra/services/default/CallbackWrapperService/callbackwrapperprocess_client_ep``
  - Callback URL: ``http://localhost:7101/CallbackWrapperService/Proxy/CallbackWrapperCallbackService?clientCallbackUrl=aHR0cDovL2xvY2FsaG9zdDo4MDgwL0NhbGxiYWNrVGVzdFN1aXRlL0NvbXBsZXRl``
- Service Bus Second Lane:
  - Request URL: ``http://localhost:7101/CallbackWrapperService/Proxy/CallbackWrapperCallbackService?clientCallbackUrl=aHR0cDovL2xvY2FsaG9zdDo4MDgwL0NhbGxiYWNrVGVzdFN1aXRlL0NvbXBsZXRl``
  - Callback URL: ``http://localhost:8080/CallbackTestSuite/Complete``

## Testing it!

I have added several pipeline alerts to both proxy services for easy debugging. The body is also changed in order to easily identify which components handled the request and the response. Using SoapUI I have created a testcase which sends a request to the Service Bus and then waits for the callback.

The first pipeline, the SOA composite and the second pipeline each concatenate something to the element **input**:

- First pipeline adds **OSB-**
- SOA composite adds **SOA-**
- Second pipeline adds **OSBCALLBACK-**

Request SOAP Envelope:

~~~~~~~~ xml
<soapenv:Envelope xmlns:cal="http://xmlns.oracle.com/CallbackWrapperSOA/CallbackWrapperService/CallbackWrapperProcess" xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
	<soapenv:Header xmlns:wsa="http://www.w3.org/2005/08/addressing">
		<wsa:ReplyTo>
			<wsa:Address>http://localhost:8080/CallbackTestSuite/Complete</wsa:Address>
		</wsa:ReplyTo>
		<wsa:MessageID>CallbackTestSuite-Complete</wsa:MessageID>
	</soapenv:Header>
	<soapenv:Body>
		<cal:process>
			<cal:input>TestCase</cal:input>
		</cal:process>
	</soapenv:Body>
</soapenv:Envelope>
~~~~~~~~

Callback SOAP Envelope:

~~~~~~~~ xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
	<env:Header xmlns:env="http://schemas.xmlsoap.org/soap/envelope/" xmlns:wsa="http://www.w3.org/2005/08/addressing">
		<wsa:To>http://localhost:8080/CallbackTestSuite/Complete</wsa:To>
		<wsa:Action>processResponse</wsa:Action>
		<wsa:MessageID>urn:6372b7ec-0bb4-11e6-8755-606720b7b1f0</wsa:MessageID>
		<wsa:RelatesTo>CallbackTestSuite-Complete</wsa:RelatesTo>
		<wsa:ReplyTo>
			<wsa:Address>http://www.w3.org/2005/08/addressing/anonymous</wsa:Address>
			<wsa:ReferenceParameters>
				<instra:tracking.ecid xmlns:instra="http://xmlns.oracle.com/sca/tracking/1.0">81406fc3-1efd-4616-9e06-bbcfc5656d53-00000144</instra:tracking.ecid>
				<instra:tracking.conversationId xmlns:instra="http://xmlns.oracle.com/sca/tracking/1.0">CallbackTestSuite-Complete</instra:tracking.conversationId>
				<instra:tracking.FlowEventId xmlns:instra="http://xmlns.oracle.com/sca/tracking/1.0">100017</instra:tracking.FlowEventId>
				<instra:tracking.FlowId xmlns:instra="http://xmlns.oracle.com/sca/tracking/1.0">100003</instra:tracking.FlowId>
				<instra:tracking.CorrelationFlowId xmlns:instra="http://xmlns.oracle.com/sca/tracking/1.0">0000LHImctN1zWIqyoyWMG1N7qm1000003</instra:tracking.CorrelationFlowId>
				<instra:tracking.quiescing.SCAEntityId xmlns:instra="http://xmlns.oracle.com/sca/tracking/1.0">40003</instra:tracking.quiescing.SCAEntityId>
			</wsa:ReferenceParameters>
		</wsa:ReplyTo>
		<wsa:FaultTo>
			<wsa:Address>http://www.w3.org/2005/08/addressing/anonymous</wsa:Address>
		</wsa:FaultTo>
	</env:Header>
	<env:Body xmlns:env="http://schemas.xmlsoap.org/soap/envelope/" xmlns:wsa="http://www.w3.org/2005/08/addressing">
		<processResponse xmlns="http://xmlns.oracle.com/CallbackWrapperSOA/CallbackWrapperService/CallbackWrapperProcess">
			<result>OSBCALLBACK-SOA-OSB-TestCase</result>
		</processResponse>
	</env:Body>
</soapenv:Envelope>
~~~~~~~~
