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
This document specifies additional ways for discovering and emitting
Manufacturer Usage Descriptions (MUD), especially in constrained environments,
utilizing the Constrained Application Protocol (CoAP).

TODO: Should be updated.

--- middle

# Introduction and Overview

Manufacturer Usage Descriptions (MUDs), which have been specified in {{!RFC8520}},
provide a means for end devices to indicate to the network what sort of access and
network functionality they require to properly function.
This information enables automatic configuration of firewall appliances to limit
device network access to the specified network resources only, which in turn reduces
the attack surface for MUD-capable devices.

While {{!RFC8520}} contemplates the use of CoAP-related {{!RFC7252}} policies,
the MUD URL discovery methods it specifies (DHCP/DHCPv6, LLDP, and X.509
certificates) are not well-suited for constrained environments (e.g., 802.15.4
networks using 6LoWPAN and SLAAC).

Therefore, this document introduces a number of additional ways for distributing
MUD URLs using CoAP -- such as well-known URIs and parameters for the CoRE Link-Format --
which are better suited for constrained devices.
Furthermore, this document specifies a method of encoding MUD URLs in (signed)
CBOR Web Tokens (CWTs), which allows MUD managers to better validate the
authenticity of both the URL itself and the associated MUD file.

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
For the purposes of this document, we extend the MUD architecture with another
component: The MUD receiver.
The MUD receiver is the device responsible for providing and/or performing CoAP
requests to resources intended for MUD URL delivery.
While this component can also be a part of the same physical device as the MUD
Manager or the router/switch, this is not required.
Both the Thing and the MUD receiver can play active roles when
using CoAP {{!RFC7252}} for exposing and discovering MUD URLs.

A general overview of the MUD architecture adjusted for using CoAP in a
constrained environment can be seen in {{arch1-fig}}.
Here, we can see that both the Thing and the MUD receiver (as the recipient of
the MUD URLs) may initiate the MUD discovery process:
The Thing can contact and register with MUD URL recipients, e.g. by sending a
CoAP POST request via Multicast, addressing a well-known registration endpoint.
Conversely, MUD recipients can initiate the discovery process, e.g. by sending a
COAP GET request to a well-known URI via multicast.

Note that the protocol used for communication between the MUD Receiver and
MUD Manager is not specified in this document and implementation-specific.

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
.         __________ Forward valid ____________   .
.        +          +  MUD-URLs   +            +  .
.        |   MUD    +------------>+    MUD     |  .
.        | Receiver |             |  Manager   |  .
.        +__________+             +____________+  .
.             ^                         .         .
.             |                         .         .
. Provide MUD |     End system network  .         .
.  URL (CoAP) |                         .         .
.           __+____                 _________     .
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
For this purpose, this document specifies how to use CBOR Web Tokens {{!RFC8392}}
to include MUD URLs as part of a signed and therefore authenticable token.
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

# General Considerations

For this document, we focus on two mechanisms for exposing MUD URLs in a
constrained network:
On the one hand, Things can expose MUD-URLs as a dedicated CoAP resource.
This allows them to further secure the payload and provide additional
authentication, e.g., by embedding the URL in a CBOR Web Token.
On the other hand, Things can include a MUD URL in a list of links, using, for
example, the CoRE Link-Format.
This Web Linking approach also allows Things to submit MUD URLs to a Directory
Service.

The first part of this section describes the mechanisms that use a dedicated
MUD URL CoAP resource.
In the later parts of this section, discovery methods for this MUD URL resource
are specified.
As part of the discovery method description, a method of providing MUD URLs directly
using the CoRE link format is also described.

## MUD-URL CoAP Submission Flows {#general_submission-flows}
In general, this specification provides two ways by which a MUD-URL transmission can be performed using an explicit MUD URL resource.

In environments where many Things need to be managed over several subnets and where multicast usage is not desirable, it can be advantageous if the MUD receiver provides a CoAP resource to perform submissions to and the Things initiate the MUD-URL submission.
This will be referred to as the "Thing-initiated" submission flow for the remainder of this specification.

Conversely, in environments where multicast is not an issue and things might be limited in their capabilities, it can be advantageous if MUD receivers retrieve the MUD-URL from a CoAP resource provided by the Things.
In this specification, this will be referred to as the "Receiver-initiated" submission flow.

### Using the MUD-URL Resource (Receiver-initiated)

In the Receiver-initiated flow, Things provide a CoAP resource discoverable by the means provided in {{general_discovery}}, which is then requested by MUD receivers to retrieve the MUD-URL.

<!-- TODO ascii-drawing of this resource -->

In general, the Receiver-initiated MUD-URL flow can be divided into these steps:

1.  After joining the network, the Thing starts providing a CoAP resource to retrieve the MUD-URL.
    This resource should provide the MUD-URL in one of the formats specified in {{general_payloads}}.
    It also makes this resource discoverable for MUD receivers using the methods specified in {{general_discovery}}.

2.  The MUD Receiver discovers the resource using the aforementioned methods.
    Depending on the method of discovery, this could for example happen using a periodic scan for devices, e.g., by periodically requesting a well-known URI using multicast.

3.  The MUD Receiver retrieves the discovered resource for devices where the MUD controller does not have an up-to-date MUD URL in memory.
    To do so, it performs a CoAP request for the discovered MUD-URL resource URI using the GET method, which is responded to with the appropriate payload.
    Receivers MUST specify their desired payload format using the Accept option {{Section 5.10.4 of RFC7252}}.
    During content negotiation, Receivers MUST start with requesting the Content-Format that provides the greatest degree of authenticity protection (i.e., prefer CWTs over plaintext transmission).

<!-- TODO advantages/disadvantages? -->

### Using the MUD-URL Submission Resource (Thing-initiated)

In the Thing-initiated flow, Things discovery a submission resource provided by the MUD Receiver and submit their MUD-URLs to this resource.

This flow can be divided into these general steps:

1.  The MUD Receiver provides a CoAP resource that Things can submit their MUD-URLs to.
    It also makes itself discoverable for Things using the methods specified in {{general_discovery}}.

2.  The Thing connects to the network.
    After connecting, it discovers the MUD-URL submission resource using the aforementioned methods.

3.  The Thing submits the MUD-URL to the previously discovered URI.
    To do so, it performs a CoAP request to the discovered URI with the POST method.
    The MUD-URL is contained as the message payload in this request using one of the content formats defined in {{general_payloads}}.
    Receivers MAY be configured to limit the accepted Content-Formats to ones that provide some level of authenticity for the MUD-URL based on policy.
    However, receiver implementations MUST also support unauthenticated MUD-URL transmissions if no policy forbids it.
<!-- TODO message response/response code indicating success? -->

<!-- TODO advantages/disadvantages? -->

## MUD CoAP Payloads {#general_payloads}
For the purposes of this specification, we will define two formats for transmitting MUD-URLs, which are suitable for different environments.
MUD Receivers that conform to this specification MUST support both formats.

### Plain URL
The easiest method of transmitting MUD-URLs is using a plain text payload containing only the MUD-URL.
While this method has the advantage of simplicity, it does not contain any additional information that could be used by a MUD receiver to authenticate the supplied MUD-URL.

CoAP requests and responses that use this format MUST use the Content-Format option with the value corresponding to the "application/mud-url+plain" media type.

### MUD-URLs inside of CBOR Web Tokens
Previous methods of transmitting MUD-URLs do not allow for authentication of supplied MUD URLs.
To accomodate for environments where authentication of MUD-URLs is desired, it is also possible to include the MUD-URL as a claim inside of a CBOR Web Token {{!RFC8392}}.
This allows for MUD receivers or MUD controllers to verify the authenticity of the provided MUD-URL.

CBOR Web Tokens that contain MUD-URL information have the following properties:

- The MUD-URL is contained as an ASCII-encoded string in the "mud-url" claim.
- The Token MAY contain Proof-of-Possession claims {{!RFC8747}}.
  If it does, the MUD receiver MUST verify that the device is in possession of the key specified in the cnf claim.
  Proof-of-Possession claims that are transmitted over CoAP without DTLS/TLS MUST represent public keys, Receivers MUST treat any symmetric keys sent over insecure channels as invalid/compromised.
  <!-- TODO specify PoP mechanism in flow section -->
- The Token MAY contain an expiry time.
  If an expiry time is specified, the MUD-URL should be resubmitted or requested again shortly before the original CWT expires.
  Note that using an expiry time could cause problems if the device is unable to perform a refresh, e.g., due to a power outage.
  <!-- TODO maybe be more specific regarding the time where the refresh should happen -->

CoAP requests and responses that use this format MUST use the Content-Format option with the value corresponding to the "application/mud-url+cwt" media type.

## Proof-of-Possession for CBOR Web Tokens with MUD-URLs {#general_pop}
If the MUD-URL is transmitted as a CBOR Web Token that includes Proof-of-Possession claims, the proof-of-possession claim SHOULD be verified.
To do so, the following procedures SHOULD be used.
Both of the following procedures use the establishment of a DTLS session using the PoP key in order to prove the possession of the key, similarly to the procedures defined in {{!RFC9202}}.

<!-- TODO required ciphers? -->

### For the Thing-initiated Flow

1.  After submitting the MUD-URL, the MUD Receiver parses the token.
    If it detects Proof-of-Possession claims, the receiver MUST reply with a 4.01 (Unauthorized) CoAP response code and reject the token for now.
    If the original submission request was not performed using CoAPS, the Receiver MUST also return a set of key information that can be used to establish a CoAP+DTLS session with it, as well as a CoAPS URI indicating the location of the secured submission resource.
    <!-- TODO key format? -->
2.  The Thing then repeats the submission request using its own key information, the Receiver's key information, and the provided URI.
3.  During the DTLS handshake, the Receiver MUST require the Thing to use the Proof-of-Possession key included in the originally submitted token in order to successfully establish a DTLS session.
4.  After the DTLS handshake, the Thing has proven possession of the key included in the CWT to the MUD Receiver, which can then treat the CWT as valid.
<!-- TODO do we still want to re-send the CWT? -->

### For the Receiver-initiated Flow
1. After receiving the MUD-URL from a request (e.g., a multicast request), the MUD Receiver parses the token.
   If it detects Proof-of-Possession claims, the receiver MUST continue with the following procedure before treating the CWT as valid.
2. The receiver repeats the MUD-URL request, this time using DTLS.
   During the DTLS handshake, the Receiver must require the Thing to use the Proof-of-Possession key included in the originally received token in order to successfully establish the DTLS session.
3. After the DTLS handshake, the Thing has proven possession of the key included in the CWT to the MUD Receiver, which can then treat the CWT as valid.
   <!-- TODO Unsure if DTLS even works without mutual auth, but i dont see why it shouldn't   -->

## Resource Discovery {#general_discovery}

In this section, additional methods for resource discovery in constrained environments are defined.

### Well-known URI and Multicast Addresses

This document introduces a new well-known URI for discovering MUD URLs directly: `/.well-known/mud-url`.

`/.well-known/mud-url` MAY be used to expose a URL pointing to a MUD file hosted by an external MUD file server.
This MUD file MUST describe the device the URL was retrieved from or is referring to within a list of CoRE links.

{{!RFC7252}} registers one IPv4 and one IPv6 address each for the purpose of CoAP multicast.
In addition to these already existing "All CoAP Nodes" multicast addresses, this document defines an additional "All MUD CoAP Nodes" IPv6 multicast addresses that can be used to address only the subset of CoAP Nodes that support MUD.
If a device exposes a MUD URL via CoAP, it SHOULD join the respective multicast groups for the IP versions it supports.
Lastly, this document also defines an additional "All MUD Receivers" IPv6 multicast address that can be used by Things to interact with MUD receivers in their vicinity.

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

### Exposing MUD URLs

Things MAY expose their MUD URL using a dedicated resource hosted under
`/.well-known/mud-url`.
If a MUD URL is exposed this way, the resource MUST offer at least one of the
specified serialization methods and MUST allow clients to choose between them
using the Accept option, if provided.
If the Thing supports resource discovery using a `/.well-known/core` resource,
MUD URL resources SHOULD be advertised there using the CoRE Link-Format
{{!RFC6690}}, indicating the available Content-Formats using the `ct` parameter.

### Registering MUD URLs with MUD Receivers

Things MAY actively register their MUD URLs with MUD receivers offering a
registration interface.
To perform the registration process, Things can perform a discovery
process using the CoRE Link Format or directly include their MUD URL as a
payload of a CoAP POST request.
If Things are actively registering their MUD URLs with MUD managers, they
SHOULD regularly repeat the process to keep interested parties informed about
their presence and their associated MUD URL.

Using the CoRE Link Format, Things MAY send a GET request to the All CoAP Nodes
(IPv4) or the All MUD Receivers (IPv6) multicast address, requesting the `/.well-known/core`
resources and including an (optional) query parameter `rt=mud.url-register`.
After filtering the obtained links for the Resource Type `mud.url-register` to
identify available submission interfaces, the Thing MAY then send a unicast
POST request to the discovered MUD manager, containing the MUD URL as a payload
and a corresponding Content-Format option.
Alternatively, a Thing MAY also use a CoRE Resource Directory {{!RFC9176}} to
perform the discovery process.

Things MAY also directly send a POST request containing the MUD URL as
a payload and a corresponding Content-Format option via unicast or to the All
CoAP Nodes (IPv4) or the All MUD Receivers (IPv6) multicast address, using the well-known URI
`/.well-known/mud-submission`.

### Finding MUD URLs of Other Things

If a Thing should be interested in the MUD URLs of one or more of its peers, it
MAY use the same mechanisms specified for MUD Receivers described in the
following section.

## Receiver Behavior

MUD receivers are assumed to be mostly non-constrained devices.
Accordingly, this specification puts most of the implementation burden regarding support for flows and formats on the receivers, while keeping the requirements for Things as small as possible.
In general, it is recommended that MUD receivers support as much of the specification as possible in order to support as many different Things as possible.

### Discovery

For the discovery process described in {{general_discovery}}, the following considerations apply to MUD receivers:

- MUD receivers that support IPv6 SHOULD regularly perform a CoAP request to the "All MUD CoAP Nodes" multicast address for the `/.well-known/mud-url` URI.

- MUD receivers that support IPv4 SHOULD regularly perform a CoAP request to the "All CoAP Nodes" multicast address for the `/.well-known/mud-url` URI.

- MUD receivers that support IPv6 devices MUST join the "All MUD Receivers" IPv6 multicast address.

- MUD receivers that support IPv4 devices MUST join the "All CoAP Nodes" IPv4 multicast address.

- MUD receivers SHOULD regularly query any CoRE Resource Directories relevant for the subnet they are responsible for.

- MUD receivers SHOULD register their submission resource to any CoRE Resource Directories relevant for the subnet they are responsible for.

### MUD-URL Submission Resource

This section describes the behavior for MUD receivers regarding the Thing-initiated submission flow:

- MUD receivers MUST provide a submission resource under the `/.well-known/mud-submission` well-known URI.

- MUD receivers SHOULD include their submission resource in any CoRE Link Format descriptions of their resources (both in `/.well-known/core` and in CoRE RD registrations, if applicable).

- MUD receivers MAY indicate failure of MUD-URL submission using a CoAP Error Code.
  If the submitted MUD URL is encoded in a CBOR Web Token including proof-of-possession claims, MUD receivers MUST return the appropriate error code to initiate the proof of possession flow as described in {{general_pop}}.

### MUD-URL Resource

This section describes considerations for MUD receivers regarding the Receiver-initiated submission flow:

- MUD receivers MUST request MUD URLs from any resources known to them.

- MUD receivers MUST re-request MUD-URLs submitted as a CWT claim if the CWT has an expiry time that passed.

### MUD-URL Payload

Regarding the actual MUD URL payload transmitted using CoAP, the following considerations apply:

- MUD receivers SHOULD treat devices for which MUD-URL retrieval failed as devices the same way as devices that do not provide a MUD-URL at all.

- MUD receiver implementations MUST support the plain MUD-URL payload, although support for these MAY be disabled by policy.

- MUD receivers MUST support the CWT MUD-URL claim.

- MUD receivers SHOULD be configured with a policy as to which signers are authorized to sign tokens.

- MUD receivers SHOULD support proof-of-possession semantics in CBOR Web Tokens and validate them before treating submitted MUD URLs using this format as valid.

<!--
- Discovery
    - (same as  General Architecture)
- Providing the MUD-URL Submission Resource
- Parsing of MUD-URL Resources
    - behavior for invalid objects/URLs
        - Feedback for Thing?
            - CoAP Response Code?
    - Signature Verification
-->

# Security Considerations
(Very WIP for now)

For Thing manufacturers that intend to implement this specification:
- It is recommended to use CBOR Web Token-encoded MUD URLs whereever possible to allow for better integrity checking
- Using expiry times for CWTs containing MUD URLs can be advantageous, but also has its shortcomings.
  Most notably, expiry times are not recommended if it is possible for Things to reach a state where they have neither the means to update their CWT nor a non-expired token in their persistent storage
  As devices may be out of service for longer periods of time, preventing them from refreshing CWTs, this state is unavoidable for the vast majority of devices.
- In order to use expiry times anyways, manufacturers could use two different CWTs: A backup CWT without an expiry time with a MUD URL pointing to a very restricted MUD file that only allows updating the main CWT, and a main CWT that has an expiry time but has a more broad MUD file referenced in it.

For network operators:
Network operators SHOULD specify a policy that describes:
- Whether to accept unauthenticated MUD URLs (those that were not submitted as part of CWTs).
- The keys whose signatures should be accepted by the MUD receiver if a submission using CWTs is performed.

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
