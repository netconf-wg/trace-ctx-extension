---
title:  NETCONF Extension to support Trace Context propagation
abbrev: nc_trace
category: std
date: 2024-01-11

docname: draft-netconf-trace-ctx-extension-latest
ipr: trust200902
submissiontype: IETF
consensus: true
v: 3
area: Operations and Management
workgroup: NETCONF
keyword:
 - telemetry
 - distributed systems
 - opentelemetry
venue:
  group: NETCONF
  type: Working Group
  mail: netconf@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/netmod/
  github: TBD
  latest: TBD

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    fullname: Roque Gagliano
    organization: Cisco Systems
    street: Avenue des Uttins 5
    code: 1180
    city: Rolle
    country: Switzerland
    email: rogaglia@cisco.com

 -
    fullname: Kristian Larsson
    organization: Deutsche Telekom AG
    email: kll@dev.terastrm.net

 -
    fullname: Jan Lindblad
    organization: Cisco Systems
    email: jlindbla@cisco.com

normative:
  RFC2119:
  RFC8174:
  RFC3688:
  RFC8525:

  W3C-Trace-Context:
    title: W3C Recommendation on Trace Context
    target: https://www.w3.org/TR/2021/REC-trace-context-1-20211123/
    date: 2021-11-23

informative:
  OpenTelemetry:
    title: OpenTelemetry Cloud Native Computing Foundation project
    target: https://opentelemetry.io
    date: 2022-08-29

  I-D.ietf-netconf-transaction-id:

  RFC8641:

  W3C-Baggage:
    title: W3C Propagation format for distributed context Baggage
    target: https://www.w3.org/TR/baggage/#examples-of-http-headers
    date: 2021-11-23
    
--- abstract

This document defines how to propagate trace context information across the Network Configuration Protocol (NETCONF), that enables distributed tracing scenarios.  It is an adaption of the HTTP-based W3C specification.

--- middle

# Introduction

Network automation and management systems commonly consist of multiple
sub-systems and together with the network devices they manage, they effectively form a distributed system.  Distributed tracing is a methodology implemented by tracing tools to follow, analyze and debug operations, such as configuration transactions, across multiple distributed systems.  An operation is uniquely identified by a trace-id and through a trace context, carries some metadata about the operation.  Propagating this "trace context" between systems enables forming a coherent view of the entire operation as carried out by all involved systems.

The W3C has defined two HTTP headers for context propagation that are useful in use case scenarios of distributed systems like the ones defined in {{?RFC8309}}.  This document defines an extension to the NETCONF protocol to add the same concepts and enable trace context propagation over NETCONF.

It is worth noting that the trace context is not meant to have any relationship with the data that is carried with a given operation (including configurations, service identifiers or state information).

A trace context also differs from {{I-D.ietf-netconf-transaction-id}} in several ways as the trace operation may involve any operation (including for example validate, lock, unlock, etc.) Additionally, a trace context scope may include the full application stack (orchestrator, controller, devices, etc) rather than a single NETCONF server, which is the scope for the transaction-id. The trace context is also complemetary to {{I-D.ietf-netconf-transaction-id}} as a given trace-id can be associated with the different transaction-ids as part of the information exported to the collector.

The following enhancement of the reference SDN Architecture from RFC 8309 shows the impact of distributed traces for a network operator.

~~~ art
                 ------------------                    -------------
                |   Orchestrator   |                   |           |
                |                  |     ------------> |           |
                .------------------.                   |           |
               .          :         .                  |           |
              .           :          .                 | Collector |
   ------------     ------------     ------------      | (Metrics, |
  |            |   |            |   |            |     |  Events,  |
  | Controller |   | Controller |   | Controller | --> |  Logs,    |
  |            |   |            |   |            |     |  Traces)  |
   ------------     ------------     ------------      |           |
      :              .       .               :         |           |
      :             .         .              :         |           |
      :            .           .             :         |           |
 ---------     ---------   ---------     ---------     |           |
| Network |   | Network | | Network |   | Network |    |           |
| Element |   | Element | | Element |   | Element | -> |           |
 ---------     ---------   ---------     ---------     -------------

    Figure 1: A Sample SDN Architecture from RFC8309 augmented
      to include the export of metrics, events, logs and traces
      from the different components to a common collector.
~~~

The network automation, management and control architectures are distributed in nature.  In order to "manage the managers", operators would like to use the same techniques as any other distributed systems in their IT environment.  Solutions for analysing Metrics, Events, Logs and Traces (M.E.L.T) are key for the successful monitoring and troubleshooting of such applications.  Initiatives such as the OpenTelemetry {{OpenTelemetry}} enable rich ecosystems of tools that NETCONF-based applications would want to participate in.

With the implementation of this trace context propagation extension to NETCONF, backend systems behind the M.E.L.T collector will be able to correlate information from different systems but related to a common context.

## Implementation example 1: OpenTelemetry

We will describe an example to show the value of trace context propagation in the NETCONF protocol.  In Figure 2, we show a deployment based on Figure 1 with a single controller and two network elements.  In this example, the NETCONF protocol is running between the Orchestrator and the Controller.  NETCONF is also used between the Controller and the Network Elements.

Let's assume an edit-config operation between the orchestrator and the controller that results (either synchronously or asynchronously) in corresponding edit-config operations from the Controller towards the two network elements.  All trace operations are related and will create M.E.L.T data.

~~~ art
             ------------------                         -------------
            |   Orchestrator   |    OTLP protocol       |           |
            |                  |  ------------------->  |           |
            .------------------.                        |           |
           .  NETCONF                                   |           |
          .   edit-config (trace-id "1", parent-id "A") | Collector |
 ------------                                           | (Metrics, |
|            |                                          |  Events,  |
| Controller |   ------------------------------------>  |  Logs,    |
|            |                 OTLP protocol            |  Traces)  |
 ------------                                           |           |
   :      .  NETCONF                                    |           |
   :        . edit-config (trace-id "1", parent-id "B") |           |
   :          .                                         |           |
 ---------     ---------                                |           |
| Network |   | Network |       OTLP protocol           |           |
| Element |   | Element |  -------------------------->  |           |
 ---------     ---------                                -------------

        Figure 2: An implementation example where the NETCONF 
        protocol is used between the Orchestrator and the Controller 
        and also between the Controller and the Network Elements. 
        Every component exports M.E.L.T information to the collector 
        using the OTLP protocol.
~~~

Each of the components in this example (Orchestrator, Controller and Network Elements) is exporting M.E.L.T information to the collector using the OpenTelemetry Protocol (OTLP).

For every edit-config operation, the trace context is included.  In particular, the same trace-id "1" (simplified encoding for documentation) is included in all related NETCONF messages, which enables the collector and any backend application to correlate all M.E.L.T messages related to this transaction in this distributed stack.

Another interesting attribute is the parent-id.  We can see in this example that the parent-id between the orchestrator and the controller ("A") is different from the one between the controller and the network elements ("B").  This attribute will help the collector and the backend applications to build a connectivity graph to understand how M.E.L.T information exported from one component relates to the information exported from a different component.

With this additional metadata exchanged between the components and exposed to the M.E.L.T collector, there are important improvements to the monitoring and troubleshooting operations for the full application stack.

## Implementation example 2: YANG DataStore

OpenTelemetry implements the "push" model for data streaming where information is sent to the back-end as soon as produced and is not required to be stored in the system. In certain cases, a "pull" model may be envisioned, for example for performing forensic analysis while not all OTLP traces are available in the back-end systems.

An implementation of a "pull" mechanism for M.E.L.T. information in general and for traces in particular, could consist of storing traces in a yang datastore (particularly the operational datastore.) Implementations should consider the use of circular buffers to avoid resources exhaustion. External systems could access traces (and particularly past traces) via NETCONF, RESTCONF, gNMI or other polling mechanisms. Finally, storing traces in a YANG datastore enables the use of YANG-Push {{?RFC8641}} or gNMI Telemetry as an additional "push" mechanisms.

This document does not specify the YANG module in which traces could be stored inside the different components. That said, storing the context information described in this document as part of the recorded traces would allow back-end systems to correlate the information from different components as in the OpenTelemetry implementation.

Note to be removed in the future: Some initial ideas are under discussion in the IETF for defining a standard YANG data model for traces. For example see: I-D.quilbeuf-opsawg-configuration-tracing which focusses only on configuration change root cause analysis use case (see the use case desciption below). These ideas are complementary to this draft.

~~~ art
             ------------------                         -------------
            | Orchestrator     |                        |           |
            |                  |    NC/RC/gNMI or YP    |           |
            |   YANG DataStore | <------------------->  |           |
            .------------------.     pull or push       |           |
           .  NETCONF                                   |           |
          .   edit-config (trace-id "1", parent-id "A") | Collector |
 ----------------                                       | (Metrics, |
|                |           NC/RC/gNMI or YP           |  Events,  |
| Controller     |   -------------------------------->  |  Logs,    |
|  YANG DataStore|             pull or push             |  Traces)  |
 ----------------                                       |           |
   :      .  NETCONF                                    |           |
   :        . edit-config (trace-id "1", parent-id "B") |           |
   :          .                                         |           |
 ---------     ---------                                |           |
| Network |   | Network |        NC/RC/gNMI or YP       |           |
| Element |   | Element |  -------------------------->  |           |
|   YG DS |   |   YG DS |         pull or push          |           |
 ---------     ---------                                -------------

        Figure 3: An implementation example where the NETCONF 
        protocol is used between the Orchestrator and the Controller 
        and also between the Controller and the Network Elements.  
        M.E.L.T. information is stored in local Yang Datastores and 
        accessed by the collector using "pull" mechanisms using the 
        NETCONF (NC), RESTCONF (RC) or gNMI protocols. A "push" 
        strategy is also possible via YANG-Push or gNMI.
~~~

## Use Cases

### Provisioning root cause analysis

When a provisioning activity fails, errors are typically propagated northbound, however this information may be difficult to troubleshoot and typically, operators are required to navigate logs across all the different components.

With the support for trace context propagation as described in this document for NETCONF, the collector will be able to search every trace, event, metric, or log in connection to that trace-id and faciliate the performance of a root cause analysis due to a network changes. The trace information could also be included as an optional resource with the different NETCONF transaction ids described in {{I-D.ietf-netconf-transaction-id}}.

### System performance profiling

When operating a distributed system such as the one shown in Figure 2, operators are expected to benchmark Key Performance Indicators (KPIs) for the most common tasks.  For example, what is the typical delay when provisioning a VPN service across different controllers and devices.

Thanks to Application Performance Management (APM) systems, from these KPIs, an operator can detect a normal and abnormal behaviour of the distributed system. Also, an operator can better plan any upgrades or enhancements in the platform.

With the support for context propagation as described in this document for NETCONF, much richer system-wide KPIs can be defined and used for troubleshooting as the metrics and traces propagated by the different components share a common context.  Troubleshooting for abnormal behaviours can also be troubleshot from the system view down to the individual element.

### Billing and auditing

In certain circumstances, we could envision tracing infomration used as additional inputs to billing systems. In particular, trace context information could be used to validate that a certain northbound order was carried out in southbound systems.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT","SHOULD","SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

The XML prefixes used in this document are mapped as follows:

- xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0",
- xmlns:w3ctc="urn:ietf:params:xml:ns:netconf:w3ctc:1.0" and
- xmlns:ietf-netconf-otlp-context=
"urn:ietf:params:xml:ns:yang:ietf-netconf-otlp-context"


# NETCONF Extension

When performing NETCONF operations by sending NETCONF RPCs, a NETCONF client MAY include trace context information in the form of XML attributes.  The {{W3C-Trace-Context}} defines two HTTP headers; traceparent and tracestate for this purpose.  NETCONF clients that are taking advantage of this feature MUST add one w3ctc:traceparent attribute and MAY add one w3ctc:tracestate attribute to the nc:rpc tag.

A NETCONF server that receives a trace context attribute in the form of a w3ctc:traceparent attribute SHOULD apply the mutation rules described in {{W3C-Trace-Context}}.  A NETCONF server MAY add one w3ctc:traceparent attribute in the nc:rpc-reply response to the nc:rpc tag above.  NETCONF servers MAY also add one w3ctc:traceparent attribute in notification and update message envelopes: notif:notification, yp:push-update and yp:push-change-update.

For example, a NETCONF client might send:

~~~ xml
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1"
     xmlns:w3ctc="urn:ietf:params:xml:ns:netconf:w3ctc:1.0"
     w3ctc:traceparent=
       "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01">
  <get-config/>
</rpc>
~~~

In all cases above where a client or server adds a w3ctc:traceparent attribute to a tag, that client or server MAY also add one w3ctc:tracestate attribute to the same tag.

The proper encoding and interpretation of the contents of the w3ctc:traceparent attribute is described in {{W3C-Trace-Context}} section 3.2 except 3.2.1.  The proper encoding and interpretation of the contents in the w3ctc:tracestate attribute is described in {{W3C-Trace-Context}} section 3.3 except 3.3.1 and 3.3.1.1.  A NETCONF XML tag can only have zero or one w3ctc:tracestate attributes, so its content MUST always be encoded as a single string.  The tracestate field value is a list of list-members separated by commas (,).  A list-member is a key/value pair separated by an equals sign (=).  Spaces and horizontal tabs surrounding list-members are ignored.  There is no limit to the number of list-members in a list.

For example, a NETCONF client might send:

~~~ xml
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1"
     xmlns:w3ctc="urn:ietf:params:xml:ns:netconf:w3ctc:1.0"
     w3ctc:tracestate="rojo=00f067aa0ba902b7,congo=t61rcWkgMzE"
     w3ctc:traceparent=
       "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01">
  <get-config/>
</rpc>
~~~

As in all XML documents, the order between the attributes in an XML tag has no significance.  Clients and servers MUST be prepared to handle the attributes no matter in which order they appear.  The tracestate value MAY contain double quotes in its payload.  If so, they MUST be encoded according to XML rules, for example:

~~~ xml
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1"
     xmlns:w3ctc="urn:ietf:params:xml:ns:netconf:w3ctc:1.0"
     w3ctc:traceparent=
       "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
     w3ctc:tracestate=
       "value-with-quotes=&quot;Quoted string&quot;,other-value=123">
  <get-config/>
</rpc>
~~~

## Error handling

The NETCONF server SHOULD follow the "Processing Model for Working with Trace Context" as specified in {{W3C-Trace-Context}}.

If the server rejects the RPC because of the trace context extension value, the server MUST return an rpc-error with the following values:

      error-tag:      operation-failed
      error-type:     protocol
      error-severity: error

Additionally, the error-info tag SHOULD contain a 
otlp-trace-context-error-info structure with relevant details about
the error.  This structure is defined in the module
ietf-netconf-otlp-context.yang.  Example of a badly formated trace
context extension:

~~~ xml
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1"
     xmlns:w3ctc="urn:ietf:params:xml:ns:netconf:w3ctc:1.0"
     w3ctc:traceparent=
       "Bad Format"
     w3ctc:tracestate=
       "value-with-quotes=&quot;Quoted string&quot;,other-value=123">
  <get-config/>
</rpc>
~~~

This might give the following error response:

~~~ xml
<rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
            xmlns:w3ctc="urn:ietf:params:xml:ns:netconf:w3ctc:1.0"
            xmlns:ietf-netconf-otlp-context=
            "urn:ietf:params:xml:ns:yang:otlp-context"
            message-id="1">
  <rpc-error>
    <error-type>protocol</error-type>
    <error-tag>operation-failed</error-tag>
    <error-severity>error</error-severity>
    <error-message>
      OTLP traceparent attribute incorrectly formatted
    </error-message>
    <error-info>
      <ietf-netconf-otlp-context:meta-name>
        w3ctc:traceparent
      </ietf-netconf-otlp-context:meta-name>
      <ietf-netconf-otlp-context:meta-value>
        Bad Format
      </ietf-netconf-otlp-context:meta-value>
      <ietf-netconf-otlp-context:error-type>
        ietf-netconf-otlp-context:bad-format
      </ietf-netconf-otlp-context:error-type>
    </error-info>
  </rpc-error>
</rpc-reply>
~~~

## Trace Context extension versionning

This extension refers to the {{W3C-Trace-Context}} trace context capability. The W3C traceparent and trace-state headers include the notion of versions. It would be desirable for a NETCONF client to be able to discover the one or multiple versions of these headers supported by a server. We would like to achieve this goal avoiding the deffinition of new NETCONF capabilities for each headers' version.

We define a pair YANG modules (ietf-netconf-otlp-context-traceparent-version-1.0.yang and ietf-netconf-otlp-context-tracestate-version-1.0.yang) that SHOULD be included in the YANG library per {{RFC8525}} of the NETCONF server supporting the NETCONF Trace Context extension. These capabilities that will refer to the headers' supported versions. Future updates of this document could include additional YANG modules for new headers' versions.

# YANG Modules

## YANG module for otlp-trace-context-error-info structure

~~~~ yang
{::include src/yang/ietf-netconf-otlp-context.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-netconf-otlp-context@2023-07-01.yang”}

## YANG module for traceparent header version 1.0
~~~~ yang
{::include src/yang/ietf-netconf-otlp-context-traceparent-version-1.0.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-netconf-otlp-context-traceparent-version-1.0@2023-03-13.yang”}

## YANG module for tracestate header version 1.0
~~~~ yang
{::include src/yang/ietf-netconf-otlp-context-tracestate-version-1.0.yang}
~~~~
{: sourcecode-markers="true"
sourcecode-name="ietf-netconf-otlp-context-tracestate-version-1.0@2023-03-13.yang”}

# Security Considerations

TODO Security

# IANA Considerations

This document registers the following capability identifier URN in the 'Network Configuration Protocol (NETCONF) Capability URNs' registry:

~~~
  urn:ietf:params:netconf:capability:w3ctc:1.0
~~~

This document registers one XML namespace URN in the 'IETF XML registry', following the format defined in {{RFC3688}} (https://tools.ietf.org/html/rfc3688).

~~~
  URI: urn:ietf:params:xml:ns:netconf:w3ctc:1.0

  Registrant Contact: The IETF IESG.

  XML: N/A, the requested URI is an XML namespace.
~~~

This document registers three module names in the 'YANG Module Names' registry, defined in RFC 6020:

~~~
  name: ietf-netconf-otlp-context-traceparent-version-1.0

  prefix: ietf-netconf-otlpparent-1.0

  namespace: urn:ietf:params:xml:ns:yang:traceparent:1.0

  RFC: XXXX
~~~

and

~~~
  name: ietf-netconf-otlp-context-tracestate-version-1.0

  prefix: ietf-netconf-otlpstate-1.0

  namespace: urn:ietf:params:xml:ns:yang:tracestate:1.0

  RFC: XXXX
~~~

and

~~~
  name: ietf-netconf-otlp-context

  prefix: ietf-netconf-otlp-context

  namespace: urn:ietf:params:xml:ns:yang:otlp-context

  RFC: XXXX
~~~

# Acknowledgments

TBD 

--- back
# Changes (to be deleted by RFC Editor)

## From version 02 to 03
- Changed IANA section to IESG per IANA email
- Created sx:structure and improved error example
- Added ietf-netconf-otlp-context.yang for the sx:structure
- Created a dedicated section for the YANG modules

## From version 01 to 02
- Added Error Handling intial section
- Added how to manage versioning by defining YANG modules for each traceparent and trastate versions as defined by W3C.
- Added 'YANG Module Names'  to IANA Considerations

## From version 00 to 01
- Added new section: Implementation example 2: YANG DataStore
- Added new use case: Billing and auditing
- Added in introduction and in "Provisioning root cause analysis" the idea that the different transaction-ids defined in {{I-D.ietf-netconf-transaction-id}} could be added as part of the tracing information to be exported to the collectors, showing how the two documents are complementary.

# TO DO List (to be deleted by RFC Editor)

- Security Considerations
- The W3C is working on a draft document to introduce the concept of "baggage" {{W3C-Baggage}} that we expect part of a future draft for NETCONF and RESTCONF

# XML Attributes vs RPCs input augmentations discussion (to be deleted by RFC Editor)

There are arguments that can be raised regarding using XML Attribute or to augment NETCONF RPCs.

We studied Pros/Cons of each option and decided to propose XML attributes:

XML Attributes Pro:

- Literal alignment with W3C specification 

- Same encoding for RESTCONF and NETCONF enabling code reuse

- One specification for all current and future rpcs

XML Attributes Cons:

- No YANG modeling, multiple values represented as a single string

- Dependency on W3C for any extension or changes in the future as encoding will be dictated by string encoding

RPCs Input Augmentations Pro:

- YANG model of every leaf

- Re-use of YANG toolkits

- Simple updates by augmentations on existing YANG module

- Possibility to express deviations in case of partial support

RPCs Input Augmentations Cons:

- Need to augment every rpc, including future rpcs would need to consider these augmentations, which is harder to maintain

- There is no literal alignment with W3C standard. However, as mentioned before most of the time there will be modifications to the content

- Would need updated RFP for each change at W3C, which will make adoption of new features slower
