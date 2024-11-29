---
title: Information and Data Models for Packet Discard Reporting
abbrev: IM and DM for Packet Discard Reporting
docname: draft-ietf-opsawg-discardmodel-05
date: 2024-11-25
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
     RED93:
          title: Random Early Detection gateways for Congestion Avoidance
          author:
               ins: S. Floyd
          author:
               ins: V. Jacobson
     gMNI:
          title: gRPC Network Management Interface, IETF 98, March 2017, <https://datatracker.ietf.org/meeting/98/materials/slides-98-rtgwg-gnmi-intro-draft-openconfig-rtgwg-gnmi-spec-00>
          author:
               ins: Shakir, R.
          author:
               ins: Shaikh, A.
          author:
               ins: Borman, P.
          author:
               ins: Hines, M.
          author:
               ins: Lebsack, C.
          author:
               ins: C. Marrow
     RFC2475:
     RFC8289:
     RFC6241:
     RFC8040:
     RFC6242:
     RFC8446:
     RFC8341:
     
--- abstract

This document defines an information model and corresponding data model for packet discard reporting. The information model provides an implementation indepedent framework for classifying packet loss, to enable automated network mitigation of unintended packet loss.  The data model specifies a implementation of this framework in YANG for network elements.

--- middle

Introduction        {#introduction}
============

The primary function of a network is to transport and deliver packets according to service level objectives. Understanding both where and why packet loss occurs within a network is essential for effective network operation, with device-reported packet loss providing the most direct signal for identifying customer impact.  To effectively automate network operations, operators must be able to detect anomalous packet loss, determine its root cause, and apply appropriate mitigation actions. Some packet loss is normal or intended in IP/MPLS networks, however.  Therefore, precise classification of packet loss signals is crucial both to ensure that anomalous packet loss is easily detected and that the right action or sequence of actions is taken to mitigate the impact, as taking the wrong action can make problems worse. For example, taking a congested device out of service can make congestion worse by moving the traffic to other links or devices, which are already congested. 

Existing metrics for reporting packet loss, such as ifInDiscards, ifOutDiscards, ifInErrors, and ifOutErrors defined in {{?RFC1213}} and {{?RFC8343}}, are insufficient for several reasons. First, they lack precision; for instance, ifInDiscards aggregates all discarded inbound packets without specifying the cause, making it challenging to distinguish between intended and unintended discards. Second, these definitions are ambiguous, leading to inconsistent vendor implementations. For example, in some implementations ifInErrors accounts only for errored packets that are dropped, while in others, it includes all errored packets, whether they are dropped or not. Many implementations support more discard metrics than these, however, they have been inconsistently implemented due to the lack of a standardised classification scheme and clear semantics for packet loss reporting. For example, {{?RFC7270}} provides support for reporting discards per flow in IPFIX using forwardingStatus, however, the defined drop reason codes also lack sufficient clarity to support automated root cause analysis and impact mitigation, e.g., the "For us" reason code.

This document defines an information model for packet loss reporting which addresses these issues, providing precise classification of packet loss causes to enable accurate automated mitigation and supporting different data model implementations while maintaining consistency through clear semantics.

The scope of this document is limited to reporting packet loss at Layer 3 and frames discarded at Layer 2. This document considers only the signals that may trigger automated mitigation actions and not how the actions are defined or executed.

{{problem}} describes the problem to be solved. {{infomodel}} describes the information model. {{datamodel}} describes the corresponding network element data model and implementation requirements together with a set of examples.  {{module-datamodel}} defines the corresponding YANG module.  {{module-infomodel}} defines the information model as an abstract data structure in YANG, in accordance with {{!RFC8791}}.  {{wheredropped}} provides an example of where packets may be discarded in a device. {{mapping}} provides examples of discard signal-to-cause-to-auto-mitigation action mapping. {{experience}} details the authors' experience from implementing this model.


Terminology {#terminology}
===========

{::boilerplate bcp14-tagged}

A packet discard is any packet dropped by a device, whether intentionally or unintentionally.

Intended packet loss refers to packet discards that occur due to deliberate network policies or configurations designed to enforce security or quality of service. For example, packets dropped because they match an Access Control List (ACL) denying certain traffic types.

Unintended packet loss is the discarding of packets that the network operator otherwise intends to deliver, i.e. which indicates an error state.  There are many possible reasons for unintended packet loss, including: erroring links may corrupt packets in transit; incorrect routing tables may result in packets being dropped because they do not match a valid route; configuration errors may result in a valid packet incorrectly matching an Access Control List (ACL) and being dropped.

Tree diagrams used in this document follow the notation defined in {{?RFC8340}}.

Problem Statement   {#problem}
=================
The fundamental problem for network operators is how to automatically detect when unintended packet loss is occurring and determine the appropriate action to mitigate it. For any network there are a small set of potential actions that can be taken to minimise customer impact when unintended packet loss is detected:

1. Take a device, link, or set of devices and/or links out of service.
2. Return a device, link, or set of devices and/or links back into service.
3. Move traffic to other links or devices.
4. Roll back a recent change to a device that might have caused the problem.
5. Escalate to a network operator as a last resort.

The ability to select the appropriate mitigation action depends on four key features of packet loss:

FEATURE-DISCARD-LOCATION:
: Determines which devices, interfaces and/or flows are impacted.

FEATURE-DISCARD-RATE:
: The rate and/or magnitude of the discards which helps determine the scale of the impact and urgency of the required action.

FEATURE-DISCARD-DURATION:
: The duration of the discards which helps distinguish temporary from persistent issues.

FEATURE-DISCARD-CLASS:
: The type or class of discards, which is crucial for selecting the correct type of mitigation - for example:
  * Error discards may require taking faulty components out of service
  * No-buffer discards may require traffic redistribution
  * Policy discards typically require no automated action

While FEATURE-LOSS-LOCATION, FEATURE-LOSS-RATE, and FEATURE-LOSS-DURATION are provided by {{?RFC1213}} or {{?RFC8343}}, FEATURE-LOSS-CLASS requires a more detailed classification scheme than they define. The following information model defines such a classification scheme to enable automated mapping from loss signals to appropriate mitigation actions.

Information Model   {#infomodel}
=================
The information model is defined using YANG {{?RFC6020}} using Data Structure Extensions {{!RFC8791}}, allowing the model to remain abstract and decoupled from specific implementations in accordance with {{?RFC3444}}. This abstraction supports different data model implementations - for example, in YANG, IPFIX {{?RFC7011}}, gMNI {{gMNI}} or SNMP {{?RFC1157}} - while ensuring consistency across implementations. Using YANG for the information model enables this abstraction, leverages the community's familiarity with its syntax, and ensures lossless translation to the corresponding YANG data model for network elements, which is also defined in this document.

Structure {#infomodel-structure}
---------
The information model defines a hierarchical classification schema for packet discards. It is structured as a tree with seven layers: component, direction, type, layer, sub-type, sub-sub-type, and metric. This layered approach allows precise categorization of packet loss while maintaining flexibility for different implementations. The model separates traffic accounting from discard accounting and distinguishes between Layer 2 and Layer 3 statistics.

The elements of the tree are defined as follows:

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
  - l2: Layer 2 traffic and discards, i.e. frame and byte counts.
  - l3: Layer 3 traffic and discards, i.e. packet and byte counts.

- Sub-Type:
  - For discards:
    - errors: discards due to errors in processing packets or frames, e.g., checksum errors.
    - policy: discards due to policy enforcement, e.g., ACL drops.
    - no-buffer: discards due to lack of buffer space, e.g., congestion-related drops.

Each sub-type may contain further specific reasons for discards, providing more detailed insight into the cause of packet loss.

~~~~~~~~~~
{::include ../yang/draft-ietf-opsawg-discardmodel-04.yang.tree.txt}
~~~~~~~~~~

The corresponding YANG module is defined in {{module-infomodel}}.

For additional context, {{wheredropped}} provides an example of where packets may be discarded in a device.

An example of possible signal-to-mitigation action mapping is provided in {#mapping}.


Data Model   {#datamodel}
==========
This data model implements the preceding information model for the interface and device components.  This is classed as a Network Element model as defined by {{?RFC1157}}.

Structure {#datamodel-structure}
---------
Each component follows the hierarchical structure of direction/type/layer/sub-type defined in the information model. The following YANG tree diagram shows the complete structure:

~~~~~~~~~~
{::include ../yang/draft-ietf-opsawg-discardmodel-04.yang.tree.txt}
~~~~~~~~~~


Implementation Requirements {#requirements}
---------------------------
The following requirements apply to the implementation of the data model.  Requirements 1-10 relate to packets forwarded or discarded by the device, while requirement 11 relates to packets destined for or originating from the device:

1. All instances of Layer 2 frame or Layer 3 packet receipt, transmission, and discards MUST be accounted for.
2. All instances of Layer 2 frame or Layer 3 packet receipt, transmission, and discards SHOULD be attributed to the physical or logical interface of the device where they occur.  Where they cannot be attributed to the interface, they MUST be attributed to the device.
3. An individual frame MUST only be accounted for by either the Layer 2 traffic class or the Layer 2 discard classes within a single direction or context, i.e., ingress or egress or device.
4. An individual packet MUST only be accounted for by either the Layer 3 traffic class or the Layer 3 discard classes within a single direction or context, i.e., ingress or egress or device.
5. A frame accounted for at Layer 2 SHOULD NOT be accounted for at Layer 3 and vice versa.  An implementation MUST indicate which layers traffic and discards are counted against.
6. The aggregate Layer 2 and Layer 3 traffic and discard classes SHOULD account for all underlying frames or packets received, transmitted, and discarded across all other classes.
7. The aggregate Quality of Service (QoS) traffic and no buffer discard classes MUST account for all underlying packets received, transmitted, and discarded across all other classes.
8. In addition to the Layer 2 and Layer 3 aggregate classes, an individual discarded packet MUST only account against a single error, policy, or no-buffer discard subclass.
9. When there are multiple reasons for discarding a packet, the ordering of discard class reporting MUST be defined.
10. If Diffserv {{RFC2475}} is not used, no-buffer discards SHOULD be reported as class0.
11. Traffic to the device control plane has its own class, however, traffic from the device control plane SHOULD be accounted for in the same way as other egress traffic.  


Examples {#examples}
--------

If all of the requirements are met, a "good" unicast IPv4 packet received would increment:

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

YANG Module - Data Model {#module-datamodel}
========================

The "ietf-packet-discard-reporting" uses the "sx" structure defined in {{!RFC8791}}.

~~~~~~~~~~
<CODE BEGINS> file "ietf-packet-discard-reporting@2024-06-04.yang"
{::include ../yang/draft-ietf-opsawg-discardmodel-04.yang.txt}
<CODE ENDS>
~~~~~~~~~~

Security Considerations {#security}
=======================
This section discusses security considerations for both the information model and its implementation as a data model.

Information Model {#security-infomodel}
-----------------
The information model defined in {{module-infomodel}} specifies a YANG module using {{!RFC8791}} data extensions.  It defines a set of identities, types, and groupings. These nodes are intended to be reused by other YANG modules. The module by itself does not expose any data nodes that are writable, data nodes that contain read-only state, or RPCs. As such, there are no additional security issues related to the YANG module that need to be considered.


Data Model {#security-datamodel}
----------
The YANG module specified in {{module-datamodel}} defines a schema for data with data nodes that contain read-only state.  It is designed to be accessed via network management protocols such as NETCONF {{?RFC6241}} or RESTCONF {{?RFC8040}}. The lowest NETCONF layer is the secure transport layer, and the mandatory-to-implement secure transport is Secure Shell (SSH) {{?RFC6242}}. The lowest RESTCONF layer is HTTPS, and the mandatory-to-implement secure transport is TLS {{?RFC8446}}.

The Network Configuration Access Control Model (NACM) {{?RFC8341}} provides the means to restrict access for particular NETCONF or RESTCONF users to a preconfigured subset of all available NETCONF or RESTCONF protocol operations and content.

The module does not expose any data nodes that are writable, or RPCs. As such, there are no additional security issues related to the YANG module that need to be considered.


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

YANG Module - Information Model {#module-infomodel}
===============================

The "ietf-packet-discard-reporting" uses the "sx" structure defined in {{!RFC8791}}.


~~~~~~~~~~
<CODE BEGINS> file "ietf-packet-discard-reporting@2024-06-04.yang"
{::include ../yang/draft-ietf-opsawg-discardmodel-04.yang.txt}
<CODE ENDS>
~~~~~~~~~~


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
 Packet rx ->  Phy     +-->  Mac     +--> Ingress  +--> Buffers  +--> Egresss  +-->  Mac     +-->  Phy     +-> Packet tx
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


Example signal-to-mitigation action mapping {#mapping}
===========================================
{{ex-table}} gives an example discard signal-to-mitigation action mapping.  Mappings for a specific network will be dependent on the definition of unintended packet loss for that network.

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


Implementation Experience {#experience}
=========================
This appendix captures the authors' experience gained from implementing and applying this information model across multiple vendors' platforms, as guidance for future implementers.

1. The number and granularity of discard classes defined in the information model represent a compromise.  It aims to offer sufficient detail to enable appropriate automated actions while avoiding excessive detail, which may hinder quick problem identification.  Additionally, it helps to limit the quantity of data produced per interface, constraining the data volume and device CPU impacts.  While further granularity is possible, the defined schema has generally proven to be sufficient for the task of mitigating unintended packet loss.
2. There are many possible ways to define the discard classification tree.  For example, we could have used a multi-rooted tree, rooted in each protocol.  Instead, we opted to define a tree where protocol discards and causal discard classes are accounted for orthogonally.  This decision reduces the number of combinations of classes and has proven sufficient for determining mitigation actions.
3. NoBuffer discards can be realized differently with different memory architectures. Whether a NoBuffer discard is attributed to ingress or egress can differ accordingly.  For successful auto-mitigation, discards due to egress interface congestion should be reported on egress, while discards due to device-level congestion (e.g. due to exceeding the device forwarding rate) should be reported on ingress.
4. Platforms often account for the number of packets discarded where the TTL has expired (or Hop Limit exceeded), and the device CPU has returned an ICMP Time Exceeded message.  There is typically a policer applied to limit the number of packets sent to the device CPU, however, which implicitly limits the rate of TTL discards that are processed.  One method to account for all packet discards due to TTL expired, even those that are dropped by a policer when being forwarded to the CPU, is to use accounting of all ingress packets received with TTL=1 as a proxy measure.
5. Where no route discards are implemented with a default null route, separate discard accounting is required for any explicit null routes configured, in order to differentiate between interface/ingress/discards/policy/null-route/packets and interface/ingress/discards/errors/no-route/packets.
6. It is useful to account separately for transit packets discarded by ACLs or policers, and packets discarded by ACLs or policers which limit the number of packets to the device control plane.
7. It is not possible to identify a configuration error - e.g., when intended discards are unintended - with device packet loss metrics alone.  For example, additional context is needed to determine if ACL discards are intended or due to a misconfigured ACL, i.e., with configuration validation before deployment or by detecting a significant change in ACL discards after a configuration change compared to before.
8. Where traffic byte counters need to be 64-bit, packet and discard counters that increase at a lower rate may be encoded in 32-bit.
9. Aggregate counters need to be able to deal with the possibility of discontinuities in the underlying counters.
10. In cases where the reporting device is the source or destination of a tunnel, the ingress protocol for a packet may differ from the egress protocol; if IPv4 is tunnelled over IPv6 for example.  Some implementations may attribute egress discards to the ingress protocol.
11. While the classification tree is seven layers deep, a minimal implementation may only implement the top six layers.

