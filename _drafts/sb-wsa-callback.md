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

Maarten Smeets posted a nice [article](https://technology.amis.nl/2016/02/28/asynchronous-interaction-in-oracle-bpel-and-bpm-ws-addressing-and-correlation-sets/) describing asynchronous interaction in BPEL and BPM using WS-Addressing and correlation sets. I wanted to use Service Bus 12c to force the callback to go through a proxy instead of going directly to the consumer. This can be useful when the target service should not have the ability to call services outsite of its trusted domain or network.

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

## Testing it!