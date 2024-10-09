---
title: An Information Model for Packet Discard Reporting
abbrev: IM for Packet Discard Reporting
docname: draft-ietf-opsawg-discardmodel-04
date: 2024-09-19
category: std

ipr: trust200902
area: Operations and Management Area
workgroup: OPSAWG 
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: J. Evans
    name: John Evans
    org: Amazon
    street: 1 Principal Place, Worship Street
    city: London
    code: EC2A 2FA
    country: UK
    email: jevanamz@amazon.co.uk      
    
 -
    ins: O. Pylypenko
    name: Oleksandr Pylypenko
    org: Amazon
    street: 410 Terry Ave N
    city: Seattle
    region: WA
    code: 98109
    country: US
    email: opyl@amazon.com
    
 -
    ins: J. Haas
    name: Jeffrey Haas
    org: Juniper Networks
    street: 1133 Innovation Way
    city: Sunnyvale
    region: CA
    code: 94089
    country: US
    email: jhaas@juniper.net
 -
    ins: A. Kadosh
    name: Aviran Kadosh
    org: Cisco Systems, Inc.
    street: 170 West Tasman Dr.
    city: San Jose
    region: CA
    code: 95134
    country: US
    email: akadosh@cisco.com
 -
    ins: M. Boucadair
    name: Mohamed Boucadair
    org: Orange
    country: France
    email: mohamed.boucadair@orange.com

normative:

informative:
     RFC6241:
     RED93:
          title: Random Early Detection gateways for Congestion Avoidance
          author:
               ins: S. Floyd
          author:
               ins: V. Jacobson
     RFC2475:
     RFC8289:
     
--- abstract

The primary function of a network is to transport and deliver packets according to service level objectives.  Understanding both where and why packet loss occurs within a network is essential for effective network operation.  Device-reported packet loss provides the most direct signal for network operations to identify the customer impact resulting from unintended packet loss.  This document defines an information model for packet loss reporting, which classifies these signals to enable automated network mitigation of unintended packet loss.

--- middle

Introduction        {#introduction}
============
To effectively automate network operations, a network operator must be able to detect anomalous packet loss, determine its root cause, and then apply appropriate actions to mitigate any customer-impacting issues.  Some packet loss is normal or intended in IP/MPLS networks, however.  Therefore, precise classification of packet loss signals is crucial both to ensure that anomalous packet loss is easily detected and that the right action or sequence of actions is taken to mitigate the impact, as taking the wrong action can make problems worse.

Existing metrics for reporting packet loss, such as ifInDiscards, ifOutDiscards, ifInErrors, and ifOutErrors defined in {{?RFC1213}}, are insufficient for several reasons. First, they lack precision; for instance, ifInDiscards aggregates all discarded inbound packets without specifying the cause, making it challenging to distinguish between intended and unintended discards. Second, these definitions are ambiguous, leading to inconsistent vendor implementations. For example, in some implementations ifInErrors accounts only for errored packets that are dropped, while in others, it includes all errored packets, whether they are dropped or not. Many implementations support more discard metrics than these, however, they have been inconsistently implemented due to the lack of a standardised classification scheme and clear semantics for packet loss reporting. For example, {{?RFC7270}} provides support for reporting discards per flow in IPFIX using forwardingStatus, however, the defined drop reason codes also lack sufficient clarity (e.g., the "For us" reason code) to support automated root cause analysis and impact mitigation.

Hence, this document presents an information model for packet loss reporting, introducing a classification scheme to facilitate automated mitigation of unintended packet loss. The model addresses the aforementioned issues while remaining protocol-agnostic and implementation-independent, in accordance with {{?RFC3444}}.

The specific implementations of this information model (i.e., protocols and associated data models) are outside the scope of this document.  The scope of this document is limited to reporting packet loss at Layer 3 and frames discarded at Layer 2, although the information model might be extended in future to cover segments dropped at Layer 4. This document considers only the signals that may trigger automated mitigation plans and not how they are defined or executed.

{{problem}} describes the problem to be solved. {{model}} describes the information model and requirements with a set of examples.  {{mapping}} provides examples of discard signal-to-cause-to-auto-mitigation action mapping.  {{module}} presents the information model as an abstract data structure in YANG, in accordance with {{!RFC8791}}.  Appendix A provides an example of where packets may be discarded in a device.  Appendix B details the authors' experience from implementing this model.

Terminology {#terminology}
===========

{::boilerplate bcp14-tagged}

A packet discard is any packet dropped by a device, whether intentionally or unintentionally.

Intended packet loss refers to packet discards that occur due to deliberate network policies or configurations - such as Access Control Lists (ACLs) or policing mechanisms - designed to enforce security or quality of service.

Unintended packet loss refers to packet discards resulting from network errors, misconfigurations, hardware failures, or other anomalies not aligned with the network operator's intended behaviour. These losses negatively impact network performance and service delivery.

For example, intended packet loss occurs when packets are dropped because they match a security policy denying certain traffic types. Unintended packet loss might happen due to a faulty interface causing corrupted packets, leading to their discard.

The meanings of the symbols in the YANG tree diagrams are defined in {{?RFC8340}}.

Problem Statement   {#problem}
=================
At the highest-level, unintended packet loss is the discarding of packets that the network operator otherwise intends to deliver, i.e. which indicates an error state.  There are many possible reasons for unintended packet loss, including: erroring links may corrupt packets in transit; incorrect routing tables may result in packets being dropped because they do not match a valid route; configuration errors may result in a valid packet incorrectly matching an Access Control List (ACL) and being dropped.  While the specific definition of unintended packet loss is network-dependent, for any network there are a small set of potential actions that can be taken to minimise customer impact by automatically mitigating unintended packet loss:

1. Take a device, link, or set of devices and/or links out of service.
2. Return a device, link, or set of devices and/or links back into service.
3. Move traffic to other links or devices.
4. Roll back a recent change to a device that might have caused the problem.
5. Escalate to a network operator as a last resort.

A precise signal of impact is crucial, as taking the wrong action can be worse than taking no action. For example, taking a congested device out of service can make congestion worse by moving the traffic to other links or devices, which are already congested.

To detect whether device-reported discards indicate a problem and to determine what actions should be taken to mitigate the impact and remediate the cause, depends on four primary features of the packet loss signal:

FEATURE-LOSS-CAUSE:
: The cause of the loss.

FEATURE-LOSS-RATE:
: The rate and/or degree of the loss.

FEATURE-LOSS-DURATION:
: The duration of the loss.

FEATURE-LOSS-LOCATION:
: The location of the loss.

FEATURE-LOSS-RATE, FEATURE-LOSS-DURATION, and FEATURE-LOSS-LOCATION are already addressed with passive monitoring statistics, for example, obtained with SNMP {{?RFC1157}} / MIB-II {{?RFC1213}} or NETCONF {{?RFC6241}}. FEATURE-LOSS-CAUSE, however, is dependent on the classification scheme used for packet loss reporting. The next section defines a new classification scheme to address this problem.


Information Model   {#model}
=================

Design Rationale {#rationale}
----------------

This document uses YANG {{?RFC6020}} to represent the information model for three main reasons. First, YANG, along with its data structure extensions {{!RFC8791}}, allows designers to define the model in an abstract way, decoupled from specific implementations. This abstraction ensures consistency and provides flexibility for diverse potential implementations, with the structure and groupings easily adaptable to data models such as those specific to SNMP {{?RFC1157}}, NETCONF {{?RFC6241}}, RESTCONF {{?RFC8040}}, or IPFIX {{?RFC7011}}.  Second, this approach ensures a lossless translation from the information model to a YANG data model, preserving both semantics and structure. Lastly, YANG capitalises on the community's broad familiarity with its syntax and use, facilitating easier adoption and evolution.

Structure {#structure}
---------
The classification scheme is structured as a hierarchical tree that follows the structure: component/direction/type/layer/sub-type/sub-sub-type/.../metric.  The elements of the tree are defined as follows:

- Component: Specifies where in the device the discards are accounted. It can be:
  - interface: discards of traffic to or from a specific network interface.
  - device: discards of traffic transiting the device.
  - control-plane: discards of traffic to or from the device's control plane.
  - flow: discards of traffic associated with a specific traffic flow.

- Direction:
  - ingress: counters for incoming packets or frames.
  - egress: counters for outgoing packets or frames.

- Type:
  - traffic: counters for successfully received or transmitted packets or frames.
  - discards: counters for packets or frames that were dropped.

- Layer:
  - l2: Layer 2 discards, such as frames with CRC errors.
  - l3: Layer 3 discards, such as IP packets with invalid headers.

- Sub-Type:
  - For discards:
    - errors: discards due to errors in processing packets or frames (e.g., checksum errors).
    - policy: discards due to policy enforcement (e.g., ACL drops).
    - no-buffer: discards due to lack of buffer space (e.g., congestion-related drops).

Each sub-type may further contain specific reasons for discards, providing more detailed insight into the cause of packet loss.

~~~~~~~~~~
{::include ../yang/draft-ietf-opsawg-discardmodel-04.yang.tree.txt}
~~~~~~~~~~

For additional context, Appendix A provides an example of where packets may be discarded in a device.


Requirements {#requirements}
------------
Requirements 1-10 relate to packets forwarded by the device, while requirement 11 relates to packets destined for or originating from the device:

1. All instances of frame or packet receipt, transmission, and discards MUST be reported.
2. All instances of frame or packet receipt, transmission, and discards SHOULD be attributed to the physical or logical interface of the device where they occur.
3. An individual frame MUST only be accounted for by either the Layer 2 traffic class or the Layer 2 discard classes within a single direction or context, i.e., ingress or egress or device.
4. An individual packet MUST only be accounted for by either the Layer 3 traffic class or the Layer 3 discard classes within a single direction or context, i.e., ingress or egress or device.
5. A frame accounted for at Layer 2 SHOULD NOT be accounted for at Layer 3 and vice versa.  An implementation MUST indicate which layers a discard is counted against.
6. The aggregate Layer 2 and Layer 3 traffic and discard classes SHOULD account for all underlying frames or packets received, transmitted, and discarded across all other classes.
7. The aggregate Quality of Service (QoS) traffic and no buffer discard classes MUST account for all underlying packets received, transmitted, and discarded across all other classes.
8. In addition to the Layer 2 and Layer 3 aggregate classes, an individual discarded packet MUST only account against a single error, policy, or no-buffer discard subclass.
9. When there are multiple reasons for discarding a packet, the ordering of discard class reporting MUST be defined.
10. If Diffserv {{RFC2475}} is not used, no-buffer discards SHOULD be reported as class0.
11. Traffic to the device control plane has its own class, however, traffic from the device control plane SHOULD be accounted for in the same way as other egress traffic.  


Examples {#examples}
--------

Assuming all the requirements are met, a "good" unicast IPv4 packet received would increment:

- interface/ingress/traffic/l3/v4/unicast/packets  
- interface/ingress/traffic/l3/v4/unicast/bytes  
- interface/ingress/traffic/qos/class_0/packets  
- interface/ingress/traffic/qos/class_0/bytes  

A received unicast IPv6 packet discarded due to Hop Limit expiry would increment:

- interface/ingress/discards/l3/v6/unicast/packets  
- interface/ingress/discards/l3/v6/unicast/bytes  
- interface/ingress/discards/l3/rx/ttl-expired/packets  

An IPv4 packet discarded on egress due to no buffers would increment:

- interface/egress/discards/l3/v4/unicast/packets  
- interface/egress/discards/l3/v4/unicast/bytes  
- interface/egress/discards/no-buffer/class_0/packets  
- interface/egress/discards/no-buffer/class_0/bytes

Example Signal-Cause-Mitigation Mapping {#mapping}
=======================================
{{ex-table}} gives an example discard signal-to-cause-to-mitigation action mapping.  Mappings for a specific network will be dependent on the definition of unintended packet loss for that network.

| Discard class | Cause | Discard rate | Discard duration | Unintended? | Possible actions |
|:--------------|:------|:------------:|:----------------:|:-----------:|:-----------------|
| ingress/discards/errors/l2/rx | Upstream device or link error | >Baseline| O(1min) | Y | Take upstream link or device out-of-service |
| ingress/discards/errors/l3/rx/ttl-expired | Tracert | <=Baseline | | N | no action |
| ingress/discards/errors/l3/rx/ttl-expired | Convergence | >Baseline | O(1s) | Y | no action |
| ingress/discards/errors/l3/rx/ttl-expired | Routing loop | >Baseline | O(1min) | Y | Roll-back change |
| .\*/policy/.\* | Policy | | | N | no action |
| ingress/discards/errors/l3/no-route | Convergence | >Baseline | O(1s) | Y | no action |
| ingress/discards/errors/l3/no-route | Config error | >Baseline | O(1min) | Y | Roll-back change |
| ingress/discards/errors/l3/no-route | Invalid destination | >Baseline | O(10min) | N | Escalate to operator |
| ingress/discards/errors/local | Device errors | >Baseline | O(1min) | Y | Take device out-of-service |
| egress/discards/no-buffer | Congestion | <=Baseline | | N | no action |
| egress/discards/no-buffer | Congestion | >Baseline | O(1min) | Y | Bring capacity back into service or move traffic |
{: #ex-table title="Example Signal-Cause-Mitigation Mapping"}

The 'Baseline' in the 'Discard Rate' column is both discard class and network dependent.

YANG Module {#module}
===========

The "ietf-packet-discard-reporting" uses the "sx" structure defined in {{!RFC8791}}.


~~~~~~~~~~
<CODE BEGINS> file "ietf-packet-discard-reporting@2024-06-04.yang"
{::include ../yang/draft-ietf-opsawg-discardmodel-04.yang.txt}
<CODE ENDS>
~~~~~~~~~~

Security Considerations {#security}
=======================

The document defines a YANG module using {{!RFC8791}}. As such, this document does
not define data nodes. Following  the guidance in {{Section 3.7 of ?I-D.ietf-netmod-rfc8407bis}},
the YANG security template is not used.

IANA Considerations {#iana}
===================

   IANA is requested to register the following URI in the "ns" subregistry within
   the "IETF XML Registry" {{!RFC3688}}:

~~~~
   URI:  urn:ietf:params:xml:ns:ietf-packet-discard-reporting
   Registrant Contact:  The IESG.
   XML:  N/A; the requested URI is an XML namespace.
~~~~

   IANA is requested to register the following YANG module in the "YANG Module
   Names" subregistry {{!RFC6020}} within the "YANG Parameters" registry:

~~~~
   Name:  ietf-packet-discard-reporting
   Namespace:  urn:ietf:params:xml:ns:ietf-packet-discard-reporting
   Prefix:  plr
   Maintained by IANA?  N
   Reference:  RFC XXXX
~~~~


Contributors {#contributors}
============

    Nadav Chachmon
    Cisco Systems, Inc.
    170 West Tasman Dr.
    San Jose, CA 95134
    United States of America
    Email: nchachmo@cisco.com

Acknowledgments {#acknowledgements}
===============
The content of this document has benefitted from feedback from JR Rivers, Ronan Waide, Chris DeBruin, and Marcoz Sanz.

--- back

Where do packets get dropped? {#wheredropped}
=============================
{{ex-drop}} depicts an example of where and why packets may be discarded in a typical single-ASIC, shared-buffered type device. Packets ingress on the left and egress on the right.

~~~~~~~~~~
                                                      +----------+
                                                      |          |
                                                      |  CPU     |
                                                      |          |
                                                      +--+---^---+
                                                from_cpu |   | to_cpu
                                                         |   |
                          +------------------------------v---+-------------------------------+
                          |                                                                  |

            +----------+  +----------+  +----------+  +----------+  +----------+  +----------+  +----------+
            |          |  |          |  |          |  |          |  |          |  |          |  |          |
 Packet rx ->  Phy     +-->  Mac     +--> Ingress  +--> Buffers  +--> Egresss  +-->  Mac     +-->  Phy     |>  Packet tx
            |          |  |          |  |  Pipeline|  |          |  |  Pipeline|  |          |  |          |
            +----------+  +----------+  +----------+  +----------+  +----------+  +----------+  +----------+

  Intended                               policy/acl                  policy/acl
  Discards:                              policy/policer              policy/policer
                                         policy/urpf
                                         policy/null-route

Unintended                 error/rx/l2   error/l3/rx   no-buffer     error/l3/tx
  Discards:                              error/local
                                         error/l3/no-route
                                         error/l3/rx/ttl-expired

~~~~~~~~~~
{: #ex-drop title="Example of where packets get dropped"}

Discard Class Descriptions
--------------------------

discards/policy/:  
: These are intended discards, meaning packets dropped by a device due to a configured policy. There are multiple sub-classes.

discards/error/l2/rx/:  
: Frames discarded due to errors in the received Layer 2 frame. There are multiple sub-classes, such as those resulting from failing CRC, invalid header, invalid MAC address, or invalid VLAN.

discards/error/l3/rx/:  
: These are discards which occur due to errors in the received packet, indicating an upstream problem rather than an issue with the device dropping the errored packets. There are multiple sub-classes, including header checksum errors, MTU exceeded, and invalid packet, i.e. due to incorrect version, incorrect header length, or invalid options.

discards/error/l3/rx/ttl-expired:  
: There can be multiple causes for TTL-expired (or Hop limit exceeded) discards: i) trace-route; ii) TTL (Hop limit) set too low by the end-system; iii) routing loops. 

discards/error/l3/no-route/:  
: Discards occur due to a packet not matching any route.

discards/error/local/:  
: A device may discard packets within its switching pipeline due to internal errors, such as parity errors. Any errored discards not explicitly assigned to the above classes are also accounted for here.

discards/no-buffer/:  
: Discards occur due to no available buffer to enqueue the packet. These can be tail-drop discards or due to an active queue management algorithm, such as RED {{RED93}} or CODEL {{RFC8289}}.


Implementation Experience
=========================
This appendix captures the authors' experience gained from implementing and applying this information model across multiple vendors' platforms, as guidance for future implementers.

1. The number and granularity of classes described in Section 3 represent a compromise.  It aims to offer sufficient detail to enable appropriate automated actions while avoiding excessive detail, which may hinder quick problem identification.  Additionally, it helps limit the quantity of data produced per interface, thus constraining the data volume and device CPU impacts.  Although further granularity is possible, the scheme described has generally proven to be sufficient for the task of auto-mitigating unintended packet loss.
2. There are many possible ways to define the discard classification tree.  For example, we could have used a multi-rooted tree, rooted in each protocol.  Instead, we opted to define a tree where protocol discards and causal discards are accounted for orthogonally.  This decision reduces the number of combinations of classes and has proven sufficient for determining mitigation actions.
3. NoBuffer discards can be realized differently with different memory architectures. Whether a NoBuffer discard is attributed to ingress or egress can differ accordingly.  For successful auto-mitigation, discards due to egress interface congestion should be reported on egress, while discards due to device-level congestion (e.g. due to exceeding the device forwarding rate) should be reported on ingress.
4. Platforms often account for the number of packets discarded where the TTL has expired (or Hop Limit exceeded), and the device CPU has returned an ICMP Time Exceeded message.  There is typically a policer applied to limit the number of packets sent to the device CPU, however, which implicitly limits the rate of TTL discards that are processed.  One method to account for all packet discards due to TTL expired, even those that are dropped by a policer when being forwarded to the CPU, is to use accounting of all ingress packets received with TTL=1.
5. Where no route discards are implemented with a default null route, separate discard accounting is required for any explicit null routes configured, in order to differentiate between interface/ingress/discards/policy/null-route/packets and interface/ingress/discards/errors/no-route/packets.
6. It is useful to account separately for transit packets discarded by ACLs or policers, and packets discarded by ACLs or policers which limit the number of packets to the device control plane.
7. It is not possible to identify a configuration error - e.g., when intended discards are unintended - with device packet loss metrics alone.  For example, additional context is needed to determine if ACL discards are intended or due to a misconfigured ACL, i.e., with configuration validation before deployment or by detecting a significant change in ACL discards after a configuration change compared to before.
8. Where traffic byte counters need to be 64-bit, packet and discard counters that increase at a lower rate may be encoded in fewer bits, e.g., 32-bit.
9. Aggregate counters need to be able to deal with the possibility of discontinuities in the underlying counters.
10. In cases where the reporting device is the source or destination of a tunnel, the ingress protocol for a packet may differ from the egress protocol; if IPv4 is tunnelled over IPv6 for example.  Some implementations may attribute egress discards to the ingress protocol.
11. While the classification tree is seven layers deep, a minimal implementation may only implement the top six layers.

