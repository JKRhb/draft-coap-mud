---
title: Using MUD on CoAP environments
abbrev: MUD and CoAP
docname: draft-jimenez-t2trg-mud-coap-latest
category: info

ipr: trust200902
area: IRTF
workgroup: T2TRG Research Group
keyword: CoAP, MUD, Problem Details

stand_alone: yes
pi: [toc, sortrefs, symrefs, iprnotified]

author:
 -
    ins: J. Romann
    name: Jan Romann
    organization: Universität Bremen
    email: jan.romann@uni-bremen.de
    role: editor
 -
    ins: J. Jiménez
    name: Jaime Jiménez
    organization: Ericsson
    phone: "+358-442-992-827"
    email: jaime@iki.fi

--- abstract
This document provides some guidelines on how to add Manufacturer Usage Descriptions (MUD) to constrained environments.
We propose allowing the use of MUD URLs using the "coaps://", "coaps+tcp://", and "coaps+ws://" schemes and additional mechanisms for emitting the URLs.

--- middle

# Introduction

Manufacturer Usage Descriptions (MUDs) have been specified in {{!RFC8520}}.
As the RFC states, the goal of MUD is to provide a means for end devices to
signal to the network what sort of access and network functionality they require
to properly function.

Schemes that rely on connectivity to bootstrap the network might be flaky if that connectivity is not present, potentially preventing the device from working correctly in the absence of Internet connectivity. Moreover, even in environments that do provide connectivity it is unclear how continued operation can occur when the manufacturer's server is no longer available.

While {{!RFC8520}} contemplates the use of CoAP-related {{!RFC7252}} policies, it does not provide a viable means for constrained devices to distribute their MUD URLs in a network, since the methods it specifies (DHCP/DHCPv6, LLDP, and X.509 certificates) are not well-suited for the use with IPv6 in general and protocols like 6LoWPAN in particular.

Therefore, this document introduces a number of additional ways for distributing MUD URLs such as well-known URIs, an NDP option and parameters for the CoRE Link-Format, which are better suited for constrained devices.
Furthermore, it allows the secure CoAP protocol variants ("coaps://" {{!RFC7252}} as well as "coaps+tcp://", and "coaps+ws://" {{!RFC8323}}) for the retrieval of MUD URLs.

In theory, the permission for using secure CoAP also allows for the hosting of MUD files on IoT devices themselves.
However, since MUD files must be encoded as JSON {{!RFC8259}}, this practice is discouraged for constrained devices as of writing this document and should only be considered once a more efficient encoding format, such as CBOR {{!RFC8949}}, has been specified for the use with MUD files.
Such a specification is out of this document's scope, though.

The rest of this document is structured as follows: ... TODO

## Terminology

{::boilerplate bcp14}

Building upon the terminology defined in {{!RFC8520}}, this specification introduces the following additional terms:

TODO. (Remove if there are no additional terms.)

# Additions to the MUD Architecture

MUDs are defined in {{!RFC8520}} and are composed of:

- A URL that can be used to locate a description;
- the description itself, including how it is interpreted; and
- a means for local network management systems to retrieve the description;
- which is retrieved from a MUD File Server.

In a MUD scenario, the end device is a "Thing" that exposes a "MUD URL" to the network. Routers or Switches in the path that speak MUD can forward the URL to "MUD Managers" that query a "MUD file server" and retrieve the "MUD File" from it. After processing, the "MUD Manager" applies an access policy to the IoT Thing.

~~~
.......................................                      +-------+
.                      ____________   .                      |  MUD  |
.                     +            +  .          +-----------+-+File |
.                     |    MUD     +-->get URL+->+    MUD      +-----+
.                     |  Manager   |  .(https)   | File Server |
.  End system network +____________+<+MUD file<-<+-------------+
.                             .       .
.                             .       .
. _______                 _________   .
.+       + (DHCP et al.) + router  +  .
.| Thing +---->MUD URL+->+   or    |  .
.+_______+               | switch  |  .
.                        +_________+  .
.......................................

~~~
{: #arch-fig title='Current MUD Architecture' artwork-align="center"}

The general operation consists on a Thing emitting the MUD as a URL using DHCP, LLDP or through 802.1X, then a Switch or Router will send the URI to some IoT Controlling Entity. That Entity will fetch the MUD file from a Server on the Internet over HTTP.

MUDs can be used to mitigate DDOS attacks to and from devices, for example by prohibiting unauthorized traffic to and from IoT devices. MUD also prevents devices from becoming the attacker, as that would require the device to send traffic to an unauthorized destination. Overall MUDs can be used to automatically permit the device to send and receive only the traffic it requires to perform its intended function.

This is indeed an important subject as trustworthy IoT operations is a recurring topic in the IETF {{?RFC8576}}.

## Problems

The biggest issue with this architecture is that if the MUD File server is not available at a given time, no Thing can actually join the network. Relying on a single server is generally not a good idea. The DNS name may point to a load balancer group, but then that would require added infrastructure and complexity.

Another potential issue is that MUD files seem to be oriented to classes of devices and not specific device instances. It could be that during bootstrapping or provisioning different devices of the same class have different properties and thus different MUD files, more granularity would be preferable. This could be achieved by creating per-device MUD files on the server, but that mechanism does not seem to have been currently specified.

This brings us to the third problem, which is that the MUD file is somewhat static on a web server and out of the usual interaction patterns towards a device. In CoAP properties that are intrinsic to a device (e.g. sensing information) or configuration information (e.g. lwm2m objects used for management) are hosted by the device too, even if they could be replicated by a cloud server.

# MUD architecture using CoAP

{{!RFC8520}} allows a Thing to use the CoAP protocol scheme on the MUD URL. In this document we modify slightly the architecture. The components are:

- A URL using the "coaps://" scheme that can be used to locate a description;
- the description itself, including how it is interpreted, which is now hosted and linked from "/.well-known/core" and
- a means for local network management systems to retrieve the MUD description
- which is hosted by the Thing itself acting as CoAP MUD File Server.

~~~
...................................................
.                                  ____________   .
.                                 +            +  .
.             +------------------ |    MUD     |  .
.   get URL   |                   |  Manager   |  .
.   (coaps)   |                   +____________+  .
.  MUD file   |                         .         .
.             |                         .         .
.             |     End system network  .         .
.             |                         .         .
.           __v____                 _________     .
.          +       + (DHCP et al.) + router  +    .
.     +--- | Thing +---->MUD URL+->+   or    |    .
.     |MUD +_______+               | switch  |    .
.     |File  |                     +_________+    .
.     +------+                                    .
...................................................
~~~
{: #arch2-fig title='Self-hosted MUD Architecture' artwork-align="center"}

## Problems

This design has some problems of its own. The main being that, if DHCP is restricted in the network, the device may not be able to advertise a functional MUD URL ({{!RFC8520}} Section 1.9).

# Finding a policy: MUD URL advertisement

A CoAP endpoint will emit a URL that uses the CoAP scheme {{!RFC7252}}. This URL serves both to classify the device type and to provide a means to locate a policy file, as specified on {{!RFC8520}}.

## Dynamic Host Configuration Protocol (DHCP)

{{!RFC8520}} specifies a new MUD URL DHCP Option that carries the MUD URL to the DHCP Server. The functioning is the same as specified on {{!RFC8520}}.

If the DHCP Server cannot provide a valid IPv4 or IPv6 address, the device may use a temporary IPv6 link-local address as base URI (e.g. `coaps://[FE80::AB8]/mud/light-class`). As link local is not a globally addressable address, the MUD Manager will only be able to reach the device if it is in the same network as the device.

TBD : IPv4, coap://, ...

## Neighbor Discovery Protocol (NDP)

IPv6 hosts do not require DHCP to get access to the default gateway.
Using NDP {{!RFC4861}} and Stateless Address Autoconfiguration (SLAAC) {{!RFC4862}}, nodes can configure global addresses on their own based on prefixes contained in NDP Router Advertisements (RAs).
Therefore, DHCPv6 is typically not necessary for IPv6 hosts, which is a problem for the distribution of MUD URLs for these kinds of devices.

To solve this issue, this document introduces a new option for the NDP which makes it possible to include a MUD URL in Router Solicitations, which can be used for prompting Routers to generate RAs.
This option has the following format:

~~~~
0                   1
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Type     |    Length   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             |
+          MUDstring          |
|                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Fields:

Type:                   TBD38

Length:                 8-bit unsigned integer. Contains the
                        length of the MUDstring in octets.

MUDstring:              String containing a MUD URL as defined
                        in section 10 of {{!RFC8520}}.
                        MUST NOT exceed 254 Bytes.
~~~~
{: #fig-ndp-mud title='MUD URL Option' align="left"}

### NDP on 6LoWPANs

6LoWPANs are characterized as lossy, low-power, low-bit-rate, short-range; with many nodes saving energy with long sleep periods. For that reason vanilla NDP {{!RFC4861}} might not be the most desirable solution in such networks and optimizations like the ones provided by {{!RFC6775}} are required.

TBD : Figure out how these work

# CoAP Operations

Things can expose MUDs as well as MUD-URLs as any other resource.
Furthermore, they can expose hypermedia links pointing to MUD files using the
CoRE Link-Format {{!RFC6690}}.
Using additional Link-Format parameters and well-known URIs, this document
introduces new possibilities for discovering MUD URLs in constrained
environments.

MUD Managers can send a GET requests to a CoAP server for `/.well-known/core` and get in return a list of hypermedia links to other resources hosted in that server, encoded using the CoRE Link-Format {{!RFC6690}}.
Among those, it will get the path to the MUD file, for example `/mud` and Resource Types like `rt=mud`.

## Discovery

### Additional Well-known URIs

This document introduces two new well-known URIs for discovering both MUD files and MUD URLs directly: `/.well-known/mud-file` and `/.well-known/mud-url`.

`/.well-known/mud-file` MAY be used to expose a MUD file hosted by a device itself.
This MUD file MUST describe the device that hosts it and SHOULD be signed in accordance with section 13 of {{!RFC8520}}. <!-- TODO: Revisit this requirement -->
As stated in the introduction, this strategy is currently NOT RECOMMENDED for constrained devices, as only MUD files encoded as JSON are defined at the time of writing.
This recommendation will most likely be updated once a canonical encoding format for MUD in CBOR becomes available.

On the other hand, `/.well-known/mud-url` MAY be used to expose a URL pointing to a MUD file hosted by an external MUD file server.
This MUD file also MUST describe the device the URL was retrieved from.

### CoRE Link Format

Resources which either host MUD URLs or MUD files MAY also be indicated using the CoRE Link Format !{{RFC6690}}.
For this purpose, additional link parameters are defined:
With the link relation-types `mud-file` and `mud-url`, a link MAY be annotated as pointing to a MUD file or a MUD URL, respectively.
Note that the use of these relation-types is not limited to constrained environments and can also be used to annotate links in other contexts as well.

<!-- TODO: Mention resource-type and /.well-known/core -->

<!-- TODO: Add example -->

### Resource Directory

By using CoRE Resource Directories {{?RFC9176}}, devices can register a MUD file or MUD URL and use the directory as a MUD repository, making it discoverable with the usual RD Lookup steps.
A MUD manager itself MAY also act as a Resource Directory, directly applying registered MUD URLs or files to the network.

Lookup will use the link-relation type `rel=mud-file`, the example in Link-Format {{?RFC6690}} is:

~~~
REQ: GET coap://rd.company.com/rd-lookup/res?rel=mud-url

RES: 2.05 Content

     <coap://[2001:db8:3::101]/mud/box>;rel=mud-file;
       anchor="coap://[2001:db8:3::101]"
     <coap://[2001:db8:3::102]/mud/switch>;rel=mud-file;
       anchor="coap://[2001:db8:3::102]",
     <coap://[2001:db8:3::102]/mud/lock>;rel=mud-file;
       anchor="coap://[2001:db8:3::102]",
     <coap://[2001:db8:3::104]/mud/light>;rel=mud-file;
       anchor="coap://[2001:db8:3::104]"
~~~

TBD: Should this example be included?

The same example in CoRAL ({{?I-D.ietf-core-coral}}, {{?I-D.hartke-t2trg-coral-reef}}) is:

~~~
REQ: GET coap://rd.company.com/rd-lookup/res?rt=mud
     Accept: TBD123456 (application/coral+cbor@identity)

RES: 2.05 Content
     Content-Format: TBD123456 (application/coral+cbor@identity)

     rd-item <coap://[2001:db8:3::101]/mud/box> { rel "mud-file" }
     rd-item <coap://[2001:db8:3::102]/mud/switch> { rel "mud-file" }
     rd-item <coap://[2001:db8:3::102]/mud/lock> { rel "mud-file" }
     rd-item <coap://[2001:db8:3::103]/mud/light> { rel "mud-file" }
~~~

### Multicast

{{!RFC7252}} registers one IPv4 and one IPv6 address each for the purpose of CoAP multicast. All CoAP Nodes can be addressed at `224.0.1.187` and at `FF0X::FD`. Multicast could also be used to discover all MUDs in a subnet.

The example in Link-Format {{?RFC6690}} is:

~~~
GET coap://[FF0X::FE]/.well-known/core?rel=mud-file
~~~

The same example in CoRAL ({{?I-D.ietf-core-coral}}, {{?I-D.hartke-t2trg-coral-reef}}) is:

~~~
REQ: GET coap://[FF0X::FE]/.well-known/core?rt=mud
     Accept: TBD123456 (application/coral+cbor@identity)
~~~

### Direct MUD discovery

Using {{?RFC6690}} using CoRE Link Format, a CoAP endpoint could attempt to configure itself based on another Thing's MUD. For that reason it might fetch directly the MUD file from the device. It would start by finding if the endpoint has a MUD. The example in Link-Format {{?RFC6690}} is:

~~~
REQ: GET coap://[2001:db8:3::123]:5683/.well-known/core?rel=mud-file

RES: 2.05 Content

     <coaps://example.com/mudfile>;rel="mud-file";ct=9001;anchor="/"
~~~

<!--
TBD: Should this example be included?
In CoRAL ({{?I-D.ietf-core-coral}}, {{?I-D.hartke-t2trg-coral-reef}}):

~~~
REQ: GET coap://[2001:db8:3::123]:5683/.well-known/core?rt=mud
     Accept: TBD123456 (application/coral+cbor@identity)

RES: 2.05 Content
     Content-Format: TBD123456 (application/coral+cbor@identity)

     rd-item </mud/light> { rt "mud" }
~~~ -->

Once the client knows that there is a MUD file under `coaps://example.com/mudfile`, it can decide to follow the presented link and query it.

<!-- ~~~
REQ: GET coap://[2001:db8:3::123]:5683/mud/light
RES: 2.05 Content

     [{MUD Payload in SENML}]
~~~

In CoRAL ({{?I-D.ietf-core-coral}}, {{?I-D.hartke-t2trg-coral-reef}}):

~~~
REQ: GET coap://[2001:db8:3::123]:5683/mud/light

RES: 2.05 Content

     [{MUD Payload in SENML}]
~~~

The device may also observe the MUD resource using {{?RFC7641}}, directly subscribing to future network configuration changes. -->

# MUD File

TBD. Behaviors that are specific of CoAP should be here.

## Serialization

TBD. Write about SenML/CBOR MUDs.

# Security Considerations

TBD.

Things will expose a MUD file that has to be be signed both by the MUD author and by the device operator. Security Considerations present on Section 4.1 of {{?RFC8576}}.

We might want to use BRSKI or another similar mechanism.

Optionally the device could advertise localhost on the URL with the path to the MUD. When the network has the IP it'd append the path to it in order to fetch the MUD.

If the host part is always dynamically computed how are bootstrap / attachment schemes that depend on certs (EAP) work?

# IANA Considerations

##  Well-Known 'mud-url' URI

<!-- Retrieval of MUD URL from device, Example: "/.well-known/mud-url" -->

##  Well-Known 'mud-file' URI

<!-- Direct retrieval of MUD file, Example: "/.well-known/mud-file" -->

## New 'mud' Relation Type

## Media Types Registry

- application/mud+cbor?

## CoAP Content-Format Registry

- application/mud+json

--- back

# Acknowledgments
{: numbered="no"}

Thanks to Thomas Fossati, Klaus Hartke, Carsten Bormann and Michael Richardson for discussions on the problem space and reviews.
