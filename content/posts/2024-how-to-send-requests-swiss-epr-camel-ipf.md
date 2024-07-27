---
title: "Beginners guide to Apache Camel and IPF for the Swiss EPR"
date: 2024-07-14
draft: true
tags: ['Development', 'IPF', 'Camel', 'Java', 'Swiss EPR']
description: "The blog post is ."
---

## Introduction

In this article, I will focus on using these libraries with the Spring Boot integration.
While it is not mandatory to use Spring Boot, it is a good choice, as a lot of features implemented in IPF are
requiring it.

## About Apache Camel

Apache Camel is a powerful open-source integration framework based on known Enterprise Integration Patterns (EIP).
It allows you to define routing and mediation rules in a variety of domain-specific languages, including a Java-based
Fluent API, Spring or Blueprint XML Configuration files, and a Scala DSL.
If you are still unsure about what Camel is, you can refer to this
[nice StackOverflow thread](https://stackoverflow.com/questions/8845186/what-exactly-is-apache-camel).

I will quickly describe some components of Camel that we will need to use in this article.

### The registry

The Camel context contains a [bean registry](https://camel.apache.org/manual/registry.html), a component that Spring 
also provides.
It allows to set (binding) and retrieve (lookup) beans in the current context, and we will use it directly when 
setting the endpoint parameters.
When both Camel and Spring are used together, the Camel context will automatically delegate its bean registry to
Spring.
This is quite helpful, as you can simply define your beans in the Spring context and refer to them in the Camel 
endpoint URIs.

### The type converters

Camel implements a [type conversion](https://camel.apache.org/manual/type-converter.html) mechanism that allows you to
convert a message from one type to another.
For example, converting a `String` to a `byte[]`.
There are two types of type converters in Camel: the built-in type converters and the custom type converters.
The built-in type converters are automatically registered by Camel and are available for all routes, providing 
conversions between common types (like `String`, `byte[]`, streams, readers/writers, etc.).

IPF and Husky also provide some custom type converters, which are automatically registered in the Camel context, for all
models that are implemented for specific transactions.
For example, if you want to send a PIXv3 Query message, you can send a `PIXv3Query` object, and Camel will automatically
convert it to any type it natively supports for the HTTP transaction.
This means you can work with any type you will find in the IPF or Husky documentation without having to worry about 
using the converter.

## About IPF and Husky

The Open eHealth Integration Platform, commonly known as IPF, is a Java framework that provides an extension to 
Apache Camel to support common interfaces for health-care related data exchange, mainly around IHE and HL7 
specifications.
It contains a set of components, type converters, and utilities that will make easy for us to implement the Swiss EPR
in a Java application.

## Setting up your application

As said, we will focus here on a Spring Boot application, as it is the most common way to use IPF and Camel together.
You can use IPF without Spring Boot, but it will be more difficult to set up and use, requiring you to manually 
configure the Camel context and some of the IPF components.

### Adding the dependencies

To import IPF dependencies in the project, we will use the Spring Boot starter artifacts, providing all necessary 
dependencies and auto-configuration:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <properties>
    <ipf-version>4.8.0</ipf-version>
  </properties>
  
  <dependencies>
    <dependency>
      <groupId>org.openehealth.ipf.boot</groupId>
      <artifactId>ipf-xds-spring-boot-starter</artifactId>
      <version>${ipf-version}</version>
    </dependency>
    <dependency>
      <groupId>org.openehealth.ipf.boot</groupId>
      <artifactId>ipf-hl7v3-spring-boot-starter</artifactId>
      <version>${ipf-version}</version>
    </dependency>
    <dependency>
      <groupId>org.openehealth.ipf.boot</groupId>
      <artifactId>ipf-hpd-spring-boot-starter</artifactId>
      <version>${ipf-version}</version>
    </dependency>
    <dependency>
      <groupId>org.openehealth.ipf.commons</groupId>
      <artifactId>ipf-commons-ihe-xua</artifactId>
      <version>${ipf-version}</version>
    </dependency>
  </dependencies>
</project>
```

### Configuration at startup

The first step to send a message with Camel is to instantiate the client that will send the message.
The client is an instance of the `ProducerTemplate` class, which is instantiated from the Camel context.
The Camel context is automatically instantiated by `org.apache.camel:camel-spring` and you can autowire it with 
Spring Boot.

```java {title="Camel client instantiation"}
@Config
public class AppConfig {

    @Bean
    public ProducerTemplate createProducerTemplate(final CamelContext camelContext) {
        return camelContext.createProducerTemplate();
    }
}
```

A `ProducerTemplate` can be resource-intensive, because it creates a new thread pool for each instance
and keeps a cache of initialized transactions (i.e. when you send a message for the first time, Camel initializes
the component needed by the transaction type, which can take a few seconds).
[Camel recommends](https://camel.apache.org/manual/faq/why-does-camel-use-too-many-threads-with-producertemplate.html)
to create it once and reuse it; you are not meant to create a new instance for each message you want to send.

## Sending a request

Once you have the `ProducerTemplate`, you can use it to send a message to a Camel endpoint.
[Camel uses URIs](https://camel.apache.org/manual/uris.html) to defines the destination of the message, the protocol
to use, and the options to apply.
An example of endpoint sending a PIXv3 Query message to an HTTP endpoint would be:
`pixv3-iti45://www.example.com/pix-manager?secure=true&audit=true&sslContextParameters=#pixContext`
It is made of:

- `pixv3-iti45`: the protocol to use, in this case, a PIXv3 Query message. This one is defined by IPF,
  we will see later how to find implemented protocols.
- `www.example.com/pix-manager`: the host and path to send the message to.
- `secure=true&audit=true&sslContextParameters=#pixContext`: some configuration values applied to this transaction 
  only, defined as query string parameters in the URI.
  Those are a mix of Camel and IPF options, we will see later how to discover existing options, and the most
  common ones.

### The endpoint URI

You should have received from your EPR community the HTTP endpoints for all the transactions.
Now, you need to build the endpoint URI to send a message to that endpoint.
All needed schemes are defined by IPF and can be found in its documentation:

- [XDS based transactions](https://oehf.github.io/ipf-docs/docs/ihe/xds/), for document-related transactions:
  searching (ITI-18), retrieving (ITI-43) and publishing (ITI-41).
- [HL7v3 based transactions](https://oehf.github.io/ipf-docs/docs/ihe/hl7v3/), for patient-related transactions:
  querying identifier (ITI-45), querying demographics (ITI-47) and publishing identifiers (ITI-44).
- [HPD based transactions](https://oehf.github.io/ipf-docs/docs/ihe/hpd/), for practitioner-related transactions:
  searching (ITI-58) and updating (ITI-59).

| Transaction                                                    | URI scheme    |
|----------------------------------------------------------------|---------------|
| [ITI-45](https://oehf.github.io/ipf-docs/docs/ihe/iti45/)      | `pixv3-iti45` |
| [ITI-47](https://oehf.github.io/ipf-docs/docs/ihe/iti47/)      | `pdqv3-iti47` |
| [ITI-44](https://oehf.github.io/ipf-docs/docs/ihe/iti44pixv3/) | `pixv3-iti44` |
| [ITI-18](https://oehf.github.io/ipf-docs/docs/ihe/iti18/)      | `xds-iti18`   |
| [ITI-41](https://oehf.github.io/ipf-docs/docs/ihe/iti41/)      | `xds-iti41`   |
| [ITI-43](https://oehf.github.io/ipf-docs/docs/ihe/iti43/)      | `xds-iti43`   |
| [ITI-57](https://oehf.github.io/ipf-docs/docs/ihe/iti57/)      | `xds-iti57`   |
| [ITI-58](https://oehf.github.io/ipf-docs/docs/ihe/iti58/)      | `hpd-iti58`   |
| [ITI-59](https://oehf.github.io/ipf-docs/docs/ihe/iti59/)      | `hpd-iti59`   |

### mTLS configuration

The Swiss EPR environment requires the use of {{< abbr mTLS >}} for all transactions.
To configure mTLS in Camel, you need to set the two following options in the endpoint URI:

- `secure`: a boolean value to enable or disable the use of TLS for the HTTP connection.
  Since mTLS is mandatory in the Swiss EPR, this option should always be set to `true`.
- `sslContextParameters`: a reference to a bean that defines the TLS context to use for the connection.
  It is used to define the truststore and keystore to use, and optionally other parameters.

While the first setting is a simple boolean value, the second one is a reference to a bean.
You need to define a named bean in your application for your instance of `SSLContextParameters`.
That bean will contain a keystore, which contains the private key of your client certificate to authenticate to the
server, and a truststore, which contains the public key of the server certificate that you can use to authenticate
the server (ensuring you are connected to a legitimate host).

```java {hl_lines="4" title="SSLContextParameters bean definition"}
@Config
public class AppConfig {

    @Bean("eprTlsContext")
    public SSLContextParameters createSSLContextParameters() {
        // The keystore contains the private key of your client certificate
        final var keyStoreParameters = new KeyStoreParameters();
        keyStoreParameters.setResource("file:/path/to/your/keystore.jks");
        keyStoreParameters.setPassword("your-keystore-password");

        // The truststore contains the public key of the server certificate
        final var trustStoreParameters = new KeyStoreParameters();
        trustStoreParameters.setResource("file:/path/to/your/truststore.jks");
        trustStoreParameters.setPassword("your-truststore-password");

        final var sslContextParameters = new SSLContextParameters();
        sslContextParameters.setKeyManagers(new KeyManagersParameters());
        sslContextParameters.setTrustManagers(new TrustManagersParameters());
        sslContextParameters.setSecureSocketProtocol("TLSv1.2");
        sslContextParameters.setKeyStore(keyStoreParameters);
        sslContextParameters.setTrustStore(trustStoreParameters);

        return sslContextParameters;
    }
}
```

With this example, you can now send a message with the configured mTLS context by referencing it in
the endpoint URI:
`pixv3-iti45://www.example.com?secure=true&<strong>sslContextParameters=#eprTlsContext</strong>`.
{{< code_inline java >}}pixv3-iti45://www.example.com?secure=true&<strong>sslContextParameters=#eprTlsContext</strong>{{< /code_inline >}}.
For more information, you can refer to the
['Web Service Security' IPF documentation](https://oehf.github.io/ipf-docs/docs/ihe/wsSecureTransport#producer).

### ATNA audit configuration

Another requirement of the Swiss EPR is to audit all transactions, meaning to send a DICOM Audit Message to the
community, detailing the transaction.
IPF provides a component to automatically audit all transactions, with all information required by the Swiss EPR.
Two options are used to enable the audit of a transaction:
The

- `audit`: a boolean value to enable or disable the audit of that transaction.
- `auditContext`: a reference to a bean that defines the audit configuration to use for the transaction.
  You will need to define your `AuditEnterpriseSiteID`, and the mTLS configuration.

```java {hl_lines="4" title="AuditContext bean definition"}
@Config
public class AppConfig {

    @Bean("eprAuditContext")
    public AuditContext auditContext() throws Exception {
        final var auditContext = new DefaultAuditContext();

        auditContext.setTlsParameters(new TlsParameterTest(this.createSSLContextParameters()));
        auditContext.setAuditTransmissionProtocol(new TCPSyslogSender());
        context.setAuditEnterpriseSiteId("1.2.3");
        context.setSendingApplication("MyApplication");

        return auditContext;
    }
}
```

You can refer to the ['ATNA Auditing' IPF documentation](https://oehf.github.io/ipf-docs/docs/ihe/atna/) for more details.

### Other options

Other options can be set in the endpoint URI to configure the transaction. Some of the most common ones are:

- `inInterceptors`, `inFaultInterceptors`, `outInterceptors`, `outFaultInterceptors`: these options allow you to use
  [CXF interceptors](https://oehf.github.io/ipf-docs/docs/ihe/wsCustomInterceptors) to process incoming or outgoing 
  SOAP messages or faults.
- `features`: that option allows you to load 
  [CXF features](https://oehf.github.io/ipf-docs/docs/ihe/wsCustomFeatures) to the transaction.


## Writing the requests and reading the responses

Now that we know how to build the endpoint URI, assuming you know the particular hosts and paths to use in your EPR
community, and that we know how to configure the mTLS and audit context, we need to write the requests and read the 
response.

The IPF documentation provides information about the data models that can be used for each transaction.
I am listing here the models I recommend to use for the Swiss EPR transactions:

| Transaction message | Recommended data object                                                                                                                                              |
|---------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ITI-45 request      | [PixV3QueryRequest](https://oehf.github.io/ipf-docs/_pages/apidocs/org/openehealth/ipf/commons/ihe/hl7v3/core/requests/PixV3QueryRequest.html)                       |
| ITI-45 response     | [PixV3QueryResponse](https://oehf.github.io/ipf-docs/_pages/apidocs/org/openehealth/ipf/commons/ihe/hl7v3/core/responses/PixV3QueryResponse.html)                    |
| ITI-47 request      | `PRPAIN201305UV02Type`                                                                                                                                               |
| ITI-47 response     | `PRPAIN201306UV02Type`                                                                                                                                               |
| ITI-44 request      | `PRPAIN201301UV02Type`, `PRPAIN201302UV02Type`, `PRPAIN201304UV02Type`                                                                                               |
| ITI-44 response     | `MCCIIN000002UV01Type`                                                                                                                                               |
| ITI-18 request      | [QueryRegistry](https://oehf.github.io/ipf-docs/_pages/apidocs/org/openehealth/ipf/commons/ihe/xds/core/requests/QueryRegistry.html)                                 |
| ITI-18 response     | [QueryResponse](https://oehf.github.io/ipf-docs/_pages/apidocs/org/openehealth/ipf/commons/ihe/xds/core/responses/QueryResponse.html)                                |
| ITI-41 request      | [ProvideAndRegisterDocumentSet](https://oehf.github.io/ipf-docs/_pages/apidocs/org/openehealth/ipf/commons/ihe/xds/core/requests/ProvideAndRegisterDocumentSet.html) |
| ITI-41 response     | [Response](https://oehf.github.io/ipf-docs/_pages/apidocs/org/openehealth/ipf/commons/ihe/xds/core/responses/Response.html)                                          |
| ITI-43 request      | [RetrieveDocumentSet](https://oehf.github.io/ipf-docs/_pages/apidocs/org/openehealth/ipf/commons/ihe/xds/core/requests/RetrieveDocumentSet.html)                     |
| ITI-43 response     | [RetrievedDocumentSet](https://oehf.github.io/ipf-docs/_pages/apidocs/org/openehealth/ipf/commons/ihe/xds/core/responses/RetrievedDocumentSet.html)                  |
| ITI-57 request      | [RegisterDocumentSet](https://oehf.github.io/ipf-docs/_pages/apidocs/org/openehealth/ipf/commons/ihe/xds/core/requests/RegisterDocumentSet.html)                     |
| ITI-57 response     | [Response](https://oehf.github.io/ipf-docs/_pages/apidocs/org/openehealth/ipf/commons/ihe/xds/core/responses/Response.html)                                          |
| ITI-58 request      | [BatchRequest](https://oehf.github.io/ipf-docs/_pages/apidocs/org/openehealth/ipf/commons/ihe/hpd/stub/dsmlv2/BatchRequest.html)                                     |
| ITI-58 response     | [BatchResponse](https://oehf.github.io/ipf-docs/_pages/apidocs/org/openehealth/ipf/commons/ihe/hpd/stub/dsmlv2/BatchResponse.html)                                   |

When the request body is created, we can now initiate an instance of the transaction, set the request, send it and 
read the response.
In Camel, a transaction instance is an instance of the `Exchange` class, which contains the request and the response,
plus other properties.

```java {title="Creating an Exchange"}
Exchange exchange = new DefaultExchange(producerTemplate.getCamelContext(), ExchangePattern.InOut);
```

The first parameter is used to bind the exchange to the Camel context (to access its bean registry), and the second 
one is the exchange pattern: it informs Camel we are expecting a response.

We can then set the request in the _Exchange_ body.
In that exchange pattern, the request is called _In_, and the response is called _Out_, which can be confusing when 
implementing the client actor.

```java {title="Setting the request in the Exchange"}
final var pixRequest = new PixV3QueryRequest();
pixRequest.setQueryPatientId(new II("2.756.1.2.3.4", "patient-12345"));
// Set other request parameters

exchange.getIn().setBody(pixRequest);
```

The request is now ready to be sent to the endpoint.
This is done through the _ProducerTemplate_ instance:
  
```java {title="Sending the request"}
final var endpointUri = "pixv3-iti45://www.example.com?secure=true&sslContextParameters=#eprTlsContext";
exchange = producerTemplate.send(endpointUri, exchange);
```

The method is synchronous and will block until the response is received.
The response is now in the _Out_ body, and it must be read not with the `getOut()` method, but with the `getMessage()`
for some reason.
But before reading the response, you should check if the exchange has been successful, as the response can be null if
the transaction failed.
In that case, an exception can be found in the exchange properties.

```java {title="Checking the response"}
if (exchange.getException() !== null) {
  throw result.getException();
}
final PixV3QueryResponse response = exchange.getMessage().getBody(PixV3QueryResponse.class);
response.getPatientIds(); // Do something with the response
```

### Using a XUA assertion

For the XDS transactions, you will need to insert a XUA assertion in the message, to be authorized to access the 
patient record.
The XUA assertion is a SAML token that you get from your EPR community.

As we have seen earlier, when using IPF or Husky types in SOAP transactions, you only control the SOAP body, and the 
SOAP envelope is generated automatically.
This means that you cannot directly insert the XUA assertion in the SOAP envelope, but you can instruct IPF to insert
it for you in the following way:

```java {title="Inserting the XUA assertion"}
/**
 * Creates a Web Services Security (WSS) header and adds it to an outgoing message.
 *
 * @param messageOut   The outgoing message.
 * @param xuaAssertion The XUA Assertion to set.
 */
void addWssHeader(final Message messageOut,
                  final Element xuaAssertion) throws ParserConfigurationException {
    List<SoapHeader> soapHeaders = CastUtils.cast((List<?>) messageOut.getHeader(AbstractWsEndpoint.OUTGOING_SOAP_HEADERS));
    if (soapHeaders == null) {
        soapHeaders = new ArrayList<>(1);
        messageOut.setHeader(AbstractWsEndpoint.OUTGOING_SOAP_HEADERS, soapHeaders);
    }

    final var wssDocument = XmlFactories.newSafeDocumentBuilder().newDocument();
    wssDocument.appendChild(wssDocument.createElementNS(WSSecurityConstants.WSSE_NS, "Security"));
    wssDocument.getDocumentElement().appendChild(wssDocument.adoptNode(xuaAssertion));

    final SoapHeader wssHeader;
    try {
        wssHeader = new SoapHeader(new QName(WSSecurityConstants.WSSE_NS,
                                             "Security", WSSecurityConstants.WSSE_PREFIX),
                                   wssDocument.getDocumentElement());
        wssHeader.setDirection(Header.Direction.DIRECTION_OUT);
        soapHeaders.add(wssHeader);
    } catch (final Exception exception) {
        log.error("Error while creating the outgoing WSS header", exception);
    }
}
```

IPF is [looking for a specific message header](https://oehf.github.io/ipf-docs/docs/ihe/wsProtocolHeaders) whose 
name is contained in the constant `AbstractWsEndpoint.OUTGOING_SOAP_HEADERS`, and will automatically insert it in the
outgoing SOAP envelope.

**Warning**: when dealing with SAML assertions, you should be aware that the assertion is a signed XML element, and that 
you need to keep the original XML format when inserting it in the message.
Otherwise, the signature will be invalid, and the message will be rejected by the server.
