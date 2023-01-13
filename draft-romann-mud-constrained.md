---
v: 3
coding: utf-8

title: Using MUD in Constrained Environments
abbrev: MUD and CoAP
docname: draft-romann-mud-constrained-latest

category: info # std is not allowed for independent submissions, as it seems
submissiontype: independent
# consensus: true # Not allowed for independent submissions

# area: $AREA
# workgroup: $WorkGroup
keyword: Internet-Draft, CoAP, MUD

venue:
  github: namib-project/draft-coap-mud

author:
 -
    ins: J. Romann
    name: Jan Romann
    organization: Universität Bremen
    email: jan.romann@uni-bremen.de
    role: editor
    country: Germany
 -
    ins: H. Damer
    name: Hugo Hakim Damer
    organization: Universität Bremen
    email: hdamer@uni-bremen.de
    role: editor
    country: Germany

entity:
        SELF: "[RFC-XXXX]"

--- abstract
This document specifies additional ways for discovering Manufacturer Usage Descriptions (MUD), especially in constrained environments.
We propose allowing the use of MUD URLs using the "coaps://", "coaps+tcp://", and "coaps+ws://" schemes and additional mechanisms for emitting the URLs via CoAP and NDP.

TODO: Should be updated.

--- middle

# Introduction and Overview

Manufacturer Usage Descriptions (MUDs) have been specified in {{!RFC8520}}.
As the RFC states, the goal of MUD is to provide a means for end devices to
signal to the network what sort of access and network functionality they require
to properly function.

Schemes that rely on connectivity to bootstrap the network might be flaky if that connectivity is not present, potentially preventing the device from working correctly in the absence of Internet connectivity. Moreover, even in environments that do provide connectivity it is unclear how continued operation can occur when the manufacturer's server is no longer available.

While {{!RFC8520}} contemplates the use of CoAP-related {{!RFC7252}} policies, it does not provide a viable means for constrained devices to distribute their MUD URLs in a network, since the methods it specifies (DHCP/DHCPv6, LLDP, and X.509 certificates) are not well-suited for the use with IPv6 in general and protocols like 6LoWPAN in particular.

Therefore, this document introduces a number of additional ways for distributing MUD URLs -- such as well-known URIs, an NDP option and parameters for the CoRE Link-Format -- which are better suited for constrained devices.
Furthermore, using COSE objects, Things can distribute a signed MUD URL which allows MUD managers to better validate the authenticity of both the URL itself and the associated MUD file.
<!--
Furthermore, it allows the secure CoAP protocol variants ("coaps://" {{!RFC7252}} as well as "coaps+tcp://", and "coaps+ws://" {{!RFC8323}}) for the retrieval of MUD URLs.

TODO: Make final decision if still relevant
-->

<!-- In theory, the permission for using secure CoAP also allows for the hosting of MUD files on IoT devices themselves.
However, since MUD files must be encoded as JSON {{!RFC8259}}, this practice is discouraged for constrained devices as of writing this document and should only be considered once a more efficient encoding format, such as CBOR {{!RFC8949}}, has been specified for the use with MUD files.
Such a specification is out of this document's scope, though. -->

The rest of this document is structured as follows: ... TODO

## Terminology

{::boilerplate bcp14}

Building upon the terminology defined in {{!RFC8520}}, this specification introduces the following additional terms:

TODO. (Remove if there are no additional terms.)

# Architecture

Building upon the MUD architecture specified in {{!RFC8520}}, there are two
main network components relevant for this document:
The *Thing* that wants to obtain network access via a Router or Switch, and the
*MUD Manager* that processes MUD URLs, retrieves MUD files from the
MUD file server, and configures other network components accordingly.

While exposing a MUD URL via the NDP option defined in {{ndp}} works similar to
using DHCP(v6), with the router forwarding an emitted MUD URL to the MUD
manager, both the Thing and the MUD manager can play more active roles when
using CoAP {{!RFC7252}} for exposing and discovering MUD URLs.

A general overview of the MUD architecture adjusted for using CoAP in a
constrained environment can be seen in {{arch1-fig}}.
Here, we can see that both the Thing and the MUD manager (as the recipient of
the MUD URLs) may initiate the MUD discovery process:
The Thing can contact and register with MUD URL recipients, e.g. by sending a
CoAP POST request via Multicast, addressing a well-known registration endpoint.
Conversely, MUD recipients can initiate the discovery process, e.g. by sending a
COAP GET request to a well-known URI via multicast.

TODO: Refine diagram

<!--
TODO: Fix once plantuml is included in the GitHub Actions Docker image.
~ plantuml
package "MUD Architecture" {
  [Router or switch]
  [MUD manager]
  [Thing]
}

[Thing] - -> [Router or switch] : Emit MUD URL via NDP
[Thing] - -> [MUD manager] : Register MUD URL
[MUD manager] - -> [Thing] : Retrieve MUD URL
[Router or switch] - -> [MUD manager] : Pass MUD URL
~
-->
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
{: #arch1-fig title="Exposing and discovering MUD URLs via CoAP" artwork-align="center"}

Optionally, Things may also provide additional means for proving the
authenticity of the MUD URL associated with them.
For that purpose, this document specifies how to use CBOR Object Signing and
Encryption (COSE) {{!RFC8152}} objects to attach a manufacturer's signature to
a MUD URL.
Using this additional feature, the MUD architecture is augmented once more, as
visualized in {{arch2-fig}}.

TODO: Replace diagram

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
{: #arch2-fig title="MUD Discovery using COSE objects." artwork-align="center"}

TODO: Add more architecture stuff here.

# Exposing a MUD URL using NDP {#ndp}

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

TODO: Is there anything to take into account when using NDP on 6LoWPANs?

# MUD URL using CoAP - General Considerations

Things can expose MUD-URLs as any other resource.
Furthermore, they can expose hypermedia links pointing to MUD files using the
CoRE Link-Format {{!RFC6690}}.
Using additional Link-Format parameters and well-known URIs, this document
introduces new possibilities for discovering MUD URLs in constrained
environments.

## MUD CoAP Payloads

- Text/Plain
    - text/mud-url+plain
- CBOR (unsigned)
    - application/mud-url+cbor
- Signed MUD-URL Objects
    - application/mud-url+cose...?

## MUD-URL CoAP Submission Flows

### Using the MUD-URL Resource (Thing-side)

### Using the MUD-URL Submission Resource (Receiver-side)

## Resource Discovery

In this section, additional methods for resource discovery in constrained environments are defined.

### Well-known URI and Multicast Addresses

This document introduces a new well-known URI for discovering MUD URLs directly: `/.well-known/mud-url`.

`/.well-known/mud-url` MAY be used to expose a URL pointing to a MUD file hosted by an external MUD file server.
This MUD file MUST describe the device the URL was retrieved from or is referring to within a list of CoRE links.

{{!RFC7252}} registers one IPv4 and one IPv6 address each for the purpose of CoAP multicast.
In addition to these already existing "All CoAP Nodes" multicast addresses, this document defines additional "All MUD CoAP Nodes" multicast addresses that can be used to address only the subset of CoAP Nodes that support MUD.
If a device exposes a MUD URL via CoAP, it SHOULD join the respective multicast groups for the IP versions it supports.

### CoRE Link Format and CoRE Resource Directories

Resources which either host MUD URLs or MUD files MAY be indicated using the CoRE Link Format {{!RFC6690}}.
For this purpose, additional link parameters are defined:
<!-- TBD: Could also use the link-relation described-by with resource types mud-file or mud-file -->
With the link relation-types `mud-file` and `mud-url`, a link MAY be annotated as pointing to a MUD file or a MUD URL, respectively.
Note that the use of these relation-types is not limited to constrained environments and can also be used to annotate links in other contexts, such as a Web of Things Thing Description {{?W3C.wot-thing-description11}}.

MUD Managers or other devices can send a GET requests to a CoAP server for `/.well-known/core` and get in return a list of hypermedia links to other resources hosted in that server, encoded using the CoRE Link-Format {{!RFC6690}}.
Among those, it will get the path to the resource exposing the MUD URL, for example `/.well-known/mud-url` and link relation-types like `rel=mud-url`.

By using CoRE Resource Directories {{?RFC9176}}, devices can register a MUD file or MUD URL and use the directory as a MUD repository, making it discoverable with the usual RD Lookup steps.
A MUD manager itself MAY also act as a Resource Directory, directly applying the policies from registered MUD files to the network.
In addition to the registration endpoint defined in {{?RFC9176}}, MUD managers MAY define a separate registration interface when acting as a CoRE RD.
<!-- TODO: Decide if this makes sense -->

# Obtaining a MUD URL in Constrained Environments

<!-- TODO: This can probably be improved -->

With the additional mechanisms for finding MUD URLs introduced in this document, MUD managers can be configured to play a more active role in discovering MUD-enabled devices.
Furthermore, IoT devices could identify their peers based on a MUD URL associated with these devices or perform a configuration process based on the linked MUD file's contents.
However, the IoT devices themselves also have more options for exposing their MUD URLs more actively, using, for instance, a MUD manager's registration interface.

In the remainder of this section, we will outline potential use-cases and procedures for obtaining a MUD URL with the additional mechanisms defined above.

## Thing Behavior

- Exposition/Announcing
    - (same as General Architecture/Discovery)
- Providing the MUD-URL Resource
    - (guidance for Content Format, static pre-generation(?), etc.)

## Receiver Behavior

- Discovery
    - (same as  General Architecture)
- Providing the MUD-URL Submission Resource
- Parsing of MUD-URL Resources
    - behavior for invalid objects/URLs
        - Feedback for Thing?
            - CoAP Response Code?
    - Signature Verification

# Security Considerations

TBD.

TODO: Mention something about signing MUD files and MUD URLs using JOSE and -- in the long run -- COSE.
(TBD: Is JOSE even relevant for this document?)

- MUD-URL COSE Object Lifetime, Key Type (PKI/RPK), Contained Information (MAC-Address?)
- Multicast Considerations (DoS prevention)

# IANA Considerations

- CoAP Resource Media Types Registry
- CoAP Content Format Registry
- well-known URI Registry
- IPv6 Multicast Address Registry


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

We would like to thank Jaime Jiménez for allowing us to build upon his draft
`draft-jimenez-t2trg-mud-coap-00` for creating an initial version of this
document.
