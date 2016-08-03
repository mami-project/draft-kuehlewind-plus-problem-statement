---
title: Problem Statement and Initial Use Cases for a Path Layer UDP Substrate (PLUS)
abbrev: PLUS Problem Statement
docname: draft-kuehlewind-plus-problem-statement-latest
date: 2016-08-03
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: M. Kuehlewind
    name: Mirja Kuehlewind
    role: editor
    org: ETH Zurich
    email: mirja.kuehlewind@tik.ee.ethz.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
  -
    ins: B. Trammell
    name: Brian Trammell
    role: editor
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland


informative:
  RFC0792:
  I-D.kuehlewind-spud-use-cases:
  I-D.trammell-spud-req:
  I-D.trammell-privsec-defeating-tcpip-meta:


--- abstract




--- middle

# Problem Statement

Given the increasing number of middleboxes in the Internet, we see an
ossification of the protocol stack that makes it difficult or at least overly complicated to deploy
new protocols or protocol extensions. Examples are new transport protocols
such as SCTP or QUIC, which can only be deployed using UDP as an encapsulation
protocol, or simply TCP extensions that might be stripped at any point during
the connection. This ossification does not only make protocol design complicated 
as such cases must be handled by fall-back or happy-eyeball mechanisms, 
it also hinders innovation in the transport layer that is needed to establish
new services with different traffic characteristic and network requirements,
given that performance enhancing extensions are in many case just not usable.

While UDP encapsulation provides the needed port information
which is necessary for rather simple network mechanisms such NAT, stateful
network elements such as most firewalls require more information about
the flow semantics to maintain state, and therefore potentially might block everything other
than TCP. In addition, some network operators assume that UDP is not often used for
high-volume traffic, and is often a source of spoofing or reflected attack
traffic, and is therefore safe to block or rate-limit, even though
the volume of (good) UDP traffic is growing, mostly due to voice and 
video (real-time) services (e.g. RTCWEB) where TCP is not suitable.

Middleboxes contribute to stack ossification through two basic mechanisms: 
The first is essential manipulation of packets. An essential manipulation 
is something the middlebox was explicitly deployed to do. The second, 
accidental manipulation, is either a side effect of an essential manipulation, 
an effect of an implementation error, or an effect of a 
configuration or deployment error. Accidental manipulations 
arise from a mismatch between the actual traffic on the network and the 
assumptions made by the designers of the middlebox about that traffic. 
These tend to persist in the network, given the long development and 
deployment cycles of networking equipment.

In general, protocol ossification happens when network devices interpret 
transport or higher layer (header or even payload) information as input 
for making decisions for their provided in-network
function, without the knowledge or approval of the endpoint. Even though
these protocols were not designed to be interpreted by in-network devices, 
this is possible because these information are today most often available 
in plain text. Given the lack of a way to explicitly request or signal 
information from and to middleboxes, this had lead to a situation 
where connectivity breaks if the 
assumption made by the network devices does not map the observed traffic,
e.g. when new protocols or protocol extensions are used.

While the essential manipulation of packets should be minimized to support
innovation in the transport and application layer, 
there are in-network functions that provide value for both the traffic and
therefore the user (experience) as well as the operator's traffic management 
decisons when resources are scarce. Further, new protocols or protocol 
extension can benefit from additional knowledge provided by the network
about the expected network conditions. 

Encryption of the transport layer headers (and payload) inside UDP as 
an encapsulation layer would solve the problem of middlebox mangling. 
However, this would also break a
large number of deployed functions in the Internet, e.g. network address
translators and firewalls that rely on TCP's SYN/SYNACK semantic to set up
state. Even if firewall vendors and administrators are willing to change firewall
rules to allow more diverse UDP services, it is hard to track session state
for UDP traffic. As UDP is unidirectional, it is unknown whether the receiver
is willing to accept the connection. Further there is no way to figure how
long state must be maintained once established. To efficiently establish state
along the path an explicit contract between endpoint and stateful devices on the path is needed, 
as we have it implicitly with TCP today. We note that these resulting deployment issues led the TCP increased
security (tcpinc) working group to decide to not encrypt the TCP header.

Further, not providing any support for existing and valuable in-network
functions, can incentive blocking of encrypted traffic. Given the ossification 
we already have today, every new protocol or protocol extension must address 
today's deployment problems by the use of fallback mechanisms given service providers can 
usually not risk to lose connectivity for some of their costumers due to the 
deployment of new transport of higher layer mechanisms (despite the performance
enhancement these 'upgrades' would provide). QUIC falls back to TCP if either
no UDP-based connectivity can be established or the experienced performance of
QUIC is lower than expected. We do see blocking of TLS already today that often leads to 
fallbacks to unencrypted communication: {{RIPE72}} reports 25% of blocking of
TLS connection over port 80 in mobile network, while blocking goes up to 70%
for users that are known to be behind a proxy. Blocking by proxies might be 
an instant of accidental blocking but is likely to continue to happen if we don't
enable a way to communicate intents to middleboxes. 

This document proposes explicit signaling to and from middleboxes about 
transport and path semantics and traffic characteristic to enable
encryption of existing transport protocol headers as a solution to the 
ossification problem described above. The next section further specifies 
derived demands on the protocol design space for a new or the extension of
an existing protocol to implement such signaling.
Most important points are endpoint control over and minimization 
of exposed information to avoid future ossification that again leads to network devices
making implicit assumptions as well as the ability to make any unauthorized 
middleboxes mangling detectable as the lack of this detectability in today's protocols 
is one of the mayor root causes for the ossification problem we have today.
Detection of mangling together with the ability to have an endpoint-authorized
communication with in-network devices on the communication path will also help to
draw a clearer line between in-network functionality that the Internet architecture 
should support and unauthorized mangling.


# Solution Design Space for the Path Layer

Given there is no ability for middlebox signaling in the current Internet architecture 
(which led to today's situation where middleboxes make implicit assumptions 
based on the information that are visible on other layers),
we define any mechansim for this kind of communication as being part of a new path layer. 
This does not imply that these mechanisms must be implemented in a separate protocol.

The following points summarize the demands on a solution for path signaling based on the 
described stack ossification and resulting protocol deployment and evolution problems above. 

The path layer needs to provide:

- a clear boundary between what the path can see and what it cannot, enforced by encryption requiring a trust relation between the endpoints,

- transport-independent, in-band signaling of transport and path semantics (based on UDP encapsulation) to enable protocol evolution within the encrypted communication,

- endpoint control over explicit exposure and detectability of middlebox mangling, and

- and all this without the requirement for a trust relationship between the endpoints and middleboxes to signal basic semantics as provided today, but potential support to utilize such a relationship if it exists.

Further requirements for protocol design that also consider 
purely technical aspects with respect to, e.g., deployability are 
detailed in {{I-D.trammell-spud-reqs}}.

The following subsections detail further aspects that can be derived form the demands listed above.

## Explicit Signaling and Encryption

Compared to the present regime where middleboxes are built on 
implicit assumptions about the protocols the endpoints are
speaking to derive information about the traffic, e.g. through Deep Packet Inspection (DPI),
we see explicit signaling of transport and path semantics and traffic characteristic
as the basis for large-scale deployment of encryption without an incentive for 
network blocking.

## Trust between Endpoints and Detectability 


Detection of middlebox mangling by integrity protection 
of any plain text data/bits requires a trust relationship between endpoints.
While endpoints often establish a trust relationship at some point during their communication, 
transport protocols today usually don't take advance of this existing ability to protect their bits.
As any information above the path layer must be encrypted to avoid ossification in these layers,
the same trust relationship can be utilized to enable integrity protection in the path layer itself.
If nor the transport protocol itself or the application layer 
provide such a trust relationship, the path layer
must impose an additional shim, e.g. utilizing DTLS, for both 
encryption of these higher layers and integrity of any 
information exposed by the endpoint in the path layer.

## Endpoint Control

Explicit signaling and detectability together have the added benefit that applications (and thereby
users) have control over the information exposed to the network, providing a
means by which we can re-balance the current tussle between user privacy and
in-network functionality.


### Trust with Middleboxes and Integrity

However, the case where there is an existing trust relationship 
between an endpoint and one or multiple middleboxes on the path 
is rather rare. This implies that while endpoints can verify the integrity of information
exposed by remote endpoints, they cannot verify the integrity of information
exposed by middleboxes and middleboxes cannot verify the integrity of any
information at all. In limited situations where a trust relationship can be
established, e.g., between a managed end-user device in an enterprise network
and a corporate firewall, this verifiability can be improved.


## Declarative Signaling and Verification

There are many existing attempts to build in-band signaling for, e.g., \ac{QoS}). 
However, these have largely failed to deploy at Internet-scale, in part because the
imperative nature of the signaling: a given signal is essentially a command to
the on-path device to handle the packet in a certain, predetermined,
predictable way. This leads to incentives for endpoints to lie about their
traffic to gain advantage over traffic from other endpoints.

As, in the general case, we don't assume a pre-existing trust relationship between the
communication endpoints and any middlebox or router on the path.  We must
therefore always assume that information that is exposed can be incorrect,
and/or that the information will be ignored. That means, if there is no 
trust relationship established, signals on the path layer can only be declarative, 
in the sense that the endpoints reveal information about intended properties of traffic and hints
about which tradeoffs the traffic is willing to accept, and the path reveals
information about intended traffic handling. However, there are no guarantees
to receive a certain treatment from the network.
Providing only information that can be verified by e.g. on-path 
measurement (but with a potential high(er) effort) or are bounded to a trade-off
reduce the incentive to lie, even in entirely untrusted environments.


## Least Exposure

Path signaling should follow the principle of least exposure: for all potential
use case, we attempt to define the minimum amount of information exposed by
endpoints and middleboxes required by the proposed mechanism to solve the
identified problem. In addition to being good engineering practice, this
approach reduces the risk to privacy through inadvertent irrelevant metadata
exposure, reduces the amount of information available for application
fingerprinting, and reduces the risk that exposed information could 
otherwise be used for unintended purposes leading to unwanted ossification.

Any information exposure under endpoint control must avoid to enable a 
linkage between that information and encrypted information in the higher-level data.

Given we assume no direct relationship between an endpoint
and a certain network devices, potential use cases include in-network functions
related to traffic management as well as security function provided by the network.

## Transport-Independency and Extensibility

Any information that will be exposed in clear (not matter if integrity protected or not)
will ossify. The task of the path layer is to enable this kind of ossification
as a defined interface between endpoints and the path under endpoint-control.
To still enable innovation in the (encrypted) higher layers, any exposed information
must be transport independent to not enforce new unwanted restriction on the higher layers.
As we do not know the requirements of future transport protocol or services, it is not clear
if we can now make a decision about what should be ossified. An extendable design of 
any mechanism in the path layer would allow changes to these decision (to a certain extend).
While extensibility enables protocol evolution in itself, or even transition to comply new
mechanisms, it also adds complexity, overhead, and risk of misuse and unauthorized exploitation
of the provided facilities. Therefore extensibility must be designed carefully and restricted to a 
minimum. 

## Security and Privacy

Extensibility in general makes it possible to enlarge the maximum number of available bits,
therefore as long as any extensibility is implemented using the number of available bits as a restriction
on the information exposed is not possible. Further any available bits that have no dedicated use 
at a certain part of a communication (e.g. reserved bits as an simple example) can be exploit as a
side channel to directly signal information or connect available information with certain packets.
See {{I-D.trammell-privsec-defeating-tcpip-meta}} for further examples.

Any realization
of a signaling protocol which does not preserve the security properties of applications
with respect to their operation without this extension, or
which presents a threat to end-user privacy with respect to operation without
the extension, is not likely to be deployed.


## Diagnosability and Failure Transparency

Troubleshooting of network-related problems with transport protocols is key to
the smooth operation of a network; this is even more the case when transport
protocol headers are encrypted to prevent middlebox meddling not explicitly
authorized by the endpoints. Today troubleshooting is realized by utilizing 
exposed information in the transport header (e.g. sequence numbers and TCP timestamps)
as well as additional mechanisms that may or may not work, 
e.g., ICMP {{RFC0792}} that might be blocked.
To maintain the status quo or even improve network monitoring and 
failure detection mechanisms it is essential to integrate facilities for supporting 
troubleshooting into the pat layer as an integral part that cannot be separated 
from the basic functionality provided in the path layer. Further given that these
information will be transport-independent and under endpoint control, this simplify 
trouble-shooting and allows for optimization.


# Initial Use Cases

## Exposure of Transport Semantics for Firewall Traversal for UDP-Encapsulated Traffic

Supporting NAT and firewall traversal are fundamental requirements that any new protocol
has to support. More specifically, one needs to provide a binding of
limited related semantics (start, acknowledgment, and stop) to packets of a flow 
that are semantically related in terms of the application or higher layer protocol.  
By explicitly signaling of such transport semantics, a flow allows middleboxes 
to use those signals for setting up and tearing down their relevant state 
(NAT bindings, firewall pinholes), rather than requiring the middlebox 
to infer this state from continued traffic (such as STUN keep alive packets {{RFC6223}}).

initiator/responder scheme?


## Exposure of Transport Semantics for In-Band Measurement and Diagnosability
seq# and timestamps

## Exposure of On-Path State for Lifetime Discovery and Management (???)
time-outs? MTU? Max capacity?

## Exposure of Per-Packet Information (???)
low-latency trade-off, relative in-flow priority, re-order-sensibility...?


# IANA Considerations

This document has no actions for IANA.

# Security Considerations

Security and privacy considerations are so far given in the
corresponding subsections on security, privacy, and trust...

# Acknowledgments

This work is supported by the European Commission under Horizon 2020 grant
agreement no. 688421 Measurement and Architecture for a Middleboxed Internet
(MAMI), and by the Swiss State Secretariat for Education, Research, and
Innovation under contract no. 15.0268. This support does not imply
endorsement.


