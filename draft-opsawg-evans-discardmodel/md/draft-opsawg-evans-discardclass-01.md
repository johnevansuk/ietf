---
title: An Information Model for Packet Discard Reporting
abbrev: Info. Model for Pkt Discard Reporting
docname: draft-opsawg-evans-discardmodel-01
date: 2023-11-13
category: info

ipr: trust200902
workgroup: Independent Stream
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

normative:
     RFC2119:

informative:
     RFC1213:
     RFC1157:
     RFC3444:
     RFC5153:
     RFC6241:
     RFC7950:
     RED93:
          title: Random Early Detection gateways for Congestion Avoidance
          author:
               ins: S. Floyd
          author:
               ins: V. Jacobson
     RFC2475:
     RFC8289:
     
--- abstract

Router reported packet loss is the primary signal of when a network is not doing its job.  Some packet loss is normal or intended in TCP/IP networks, however.  To minimise network packet loss through automated network operations requires clear and accurate signals of all packets which are dropped and the reasons why.  This document defines an information model for packet loss reporting, which classifies these signals to enable automated network mitigation of unintended packet loss.

--- middle

Introduction        {#introduction}
============

The primary function of a network is to transport packets. Understanding both where and why packet loss occurs is essential for effective network operation.  Router-reported packet loss is the most direct signal for network operations to identify customer impact resulting from unintended packet loss. Accurate accounting of packet loss is not enough, however, as some level of packet loss is normal in TCP/IP networks.  In automating network operations, there are only a relatively small number of automated actions that can be taken to mitigate customer-impacting packet loss.  Hence, precise classification of packet loss signals is crucial both to ensure that customer impacting packet loss is detected and that the right action is taken to mitigate the impact, as taking the wrong action can make problems worse.

The existing metrics for packet loss, as defined in {{RFC1213}} - namely ifInDiscards, ifOutDiscards, ifInErrors, ifOutErrors - do not provide sufficient precision to automatically identify the cause of the loss and mitigate the impact.  From a network operator's perspective, ifInDiscards can represent both intended packet loss (i.e., packets discarded due to policy) and unintended packet loss (e.g., packets dropped in error). Furthermore, these definitions are ambiguous, as vendors can and have implemented them differently.  In some implementations, ifInErrors accounts only for errored packets that are dropped, while in others, it accounts for all errored packets, whether they are dropped or not.  Many implementations support more discard metrics than these; where they do, they have been inconsistently implemented due to the lack of a clearly defined classification scheme and semantics for packet loss reporting.

Hence, this document defines an information model for packet loss reporting, aiming to address these issues by presenting a packet loss classification scheme that can enable automated mitigation of unintended packet loss.  This information model is independent of any specific implementations or protocols used to transport the data {{RFC3444}}.  There are multiple ways that this information model could be implemented, including SNMP {{RFC1157}}, NETCONF {{RFC6241}} / YANG {{RFC7950}}, and IPFIX {{RFC5153}}, but they are outside of the scope of this document.

Section 2 describes the problem. Section 3 defines the information model and semantics with examples.  Section 4 provides examples of discard signal-to-cause-to-auto-mitigation action mapping.  Appendix B details the authors' experience from implementing this model.

The terms 'packet drop' and 'discard' are considered equivalent and are used interchangeably in this document.


Problem statement   {#problem}
=================

Working backwards from the goal of auto-mitigation of unintended packet loss, there are only a relatively small number of potential actions than can be taken to auto-mitigate 
customer impacting packet loss:

1. Take a device, link, or set of devices and/or links out of service.
2. Return a device, link, or set of devices and/or links back into service.
3. Move traffic to other links or devices.
4. Roll back a recent change to a device that might have caused the problem.
5. Escalate to a human (e.g., network operator) as a last resort.

A precise signal of impact is crucial, as taking the wrong action can be worse than taking no action. For example, taking a congested device out of service can make congestion worse by moving the traffic to other links or devices, which are already congested.

To detect whether router-reported packet loss is a problem and to determine what actions should be taken to mitigate the impact and remediate the cause, depends on four primary features of the packet loss signal:

1. The cause of the loss.
2. The rate and/or degree of the loss.
3. The duration of the loss.
4. The location of the loss.

Features 2, 3, and 4 are already addressed with passive monitoring statistics, for example, obtained with SNMP {{RFC1157}} / MIB-II {{RFC1213}} or NETCONF {{RFC6241}} / YANG {{RFC7950}}. Feature 1, however, is dependent on the classification scheme used for packet loss reporting. In the next section, we define a new classification scheme to address this problem.


Information model   {#model}
=================

The classification scheme is defined as a tree which follows the structure component/direction/type/layer/sub-type/sub-sub-type/.../metric, where:  
a. component can be interface|device|control_plane|flow  
b. direction can be ingress|egress  
c. type can be traffic|discards, where traffic accounts for packets successfully received or transmitted, and discards account for packet drops  
d. layer can be l2|l3

~~~~~~~~~~
.
|-- interface/
|   |-- ingress/
|   |   |-- traffic/
|   |   |   |-- l2/
|   |   |   |   |-- frames
|   |   |   |   `-- bytes
|   |   |   |-- l3/
|   |   |   |   |-- v4/
|   |   |   |   |   |-- packets
|   |   |   |   |   |-- bytes
|   |   |   |   |   |-- unicast/
|   |   |   |   |   |   |-- packets
|   |   |   |   |   |   `-- bytes
|   |   |   |   |   `-- multicast/
|   |   |   |   |       |-- packets
|   |   |   |   |       `-- bytes
|   |   |   |   `-- v6/
|   |   |   |       |-- packets
|   |   |   |       |-- bytes
|   |   |   |       |-- unicast/
|   |   |   |       |   |-- packets
|   |   |   |       |   `-- bytes
|   |   |   |       `-- multicast/
|   |   |   |           |-- packets
|   |   |   |           `-- bytes
|   |   |   `-- qos/
|   |   |       |-- class_0/
|   |   |       |   |-- packets
|   |   |       |   `-- bytes
|   |   |       |-- ...
|   |   |       `-- class_n/
|   |   |           |-- packets
|   |   |           `-- bytes
|   |   `-- discards/
|   |       |-- l2/
|   |       |   |-- frames
|   |       |   `-- bytes
|   |       |-- l3/
|   |       |   |-- v4/
|   |       |   |   |-- packets
|   |       |   |   |-- bytes
|   |       |   |   |-- unicast/
|   |       |   |   |   |-- packets
|   |       |   |   |   `-- bytes
|   |       |   |   `-- multicast/
|   |       |   |       |-- packets
|   |       |   |       `-- bytes
|   |       |   `-- v6/
|   |       |       |-- packets
|   |       |       |-- bytes
|   |       |       |-- unicast/
|   |       |       |   |-- packets
|   |       |       |   `-- bytes
|   |       |       `-- multicast/
|   |       |           |-- packets
|   |       |           `-- bytes
|   |       |-- errors/
|   |       |   |-- l2/
|   |       |   |   `-- rx/
|   |       |   |       |-- frames
|   |       |   |       |-- crc_error/
|   |       |   |       |   `-- frames
|   |       |   |       |-- invalid_mac/
|   |       |   |       |   `-- frames
|   |       |   |       |-- invalid_vlan/
|   |       |   |       |   `-- frames
|   |       |   |       `-- invalid_frame/
|   |       |   |           `-- frames
|   |       |   |-- l3/
|   |       |   |   |-- rx/
|   |       |   |   |   |-- packets
|   |       |   |   |   |-- checksum_error/
|   |       |   |   |   |   `-- packets
|   |       |   |   |   |-- mtu_exceeded/
|   |       |   |   |   |   `-- packets
|   |       |   |   |   |-- invalid_packet/
|   |       |   |   |   |   `-- packets
|   |       |   |   |   `-- ttl_expired/
|   |       |   |   |       `-- packets
|   |       |   |   `-- no_route/
|   |       |   |       `-- packets
|   |       |   `-- local/
|   |       |       |-- packets
|   |       |       `-- hw/
|   |       |           |-- packets
|   |       |           `-- parity_error/
|   |       |               `-- packets
|   |       |-- policy/
|   |       |   `-- l3/
|   |       |       |-- packets
|   |       |       |-- acl/
|   |       |       |   `-- packets
|   |       |       |-- policer/
|   |       |       |   |-- packets
|   |       |       |   `-- bytes
|   |       |       |-- null_route/
|   |       |       |   `-- packets
|   |       |       `-- rpf/
|   |       |           `-- packets
|   |       `-- no_buffer/
|   |           |-- class_0/
|   |           |   |-- packets
|   |           |   `-- bytes
|   |           |-- ...
|   |           `-- class_n/
|   |               |-- packets
|   |               `-- bytes
|   `-- egress/
|       |-- traffic/
|       |   |-- l2/
|       |   |   |-- frames
|       |   |   `-- bytes
|       |   |-- l3/
|       |   |   |-- v4/
|       |   |   |   |-- packets
|       |   |   |   |-- bytes
|       |   |   |   |-- unicast/
|       |   |   |   |   |-- packets
|       |   |   |   |   `-- bytes
|       |   |   |   `-- multicast/
|       |   |   |       |-- packets
|       |   |   |       `-- bytes
|       |   |   `-- v6/
|       |   |       |-- packets
|       |   |       |-- bytes
|       |   |       |-- unicast/
|       |   |       |   |-- packets
|       |   |       |   `-- bytes
|       |   |       `-- multicast/
|       |   |           |-- packets
|       |   |           `-- bytes
|       |   `-- qos/
|       |       |-- class_0/
|       |       |   |-- packets
|       |       |   `-- bytes
|       |       |-- ...
|       |       `-- class_n/
|       |           |-- packets
|       |           `-- bytes
|       `-- discards/
|           |-- l2/
|           |   |-- frames
|           |   `-- bytes
|           |-- l3/
|           |   |-- v4/
|           |   |   |-- packets
|           |   |   |-- bytes
|           |   |   |-- unicast/
|           |   |   |   |-- packets
|           |   |   |   `-- bytes
|           |   |   `-- multicast/
|           |   |       |-- packets
|           |   |       `-- bytes
|           |   `-- v6/
|           |       |-- packets
|           |       |-- bytes
|           |       |-- unicast/
|           |       |   |-- packets
|           |       |   `-- bytes
|           |       `-- multicast/
|           |           |-- packets
|           |           `-- bytes
|           |-- errors/
|           |   |-- l2/
|           |   |   `-- tx/
|           |   |       `-- frames
|           |   `-- l3/
|           |       `-- tx/
|           |           `-- packets
|           |-- policy/
|           |   `-- l3/
|           |       |-- acl/
|           |       |   `-- packets
|           |       `-- policer/
|           |           |-- packets
|           |           `-- bytes
|           `-- no_buffer/
|               |-- class_0/
|               |   |-- packets
|               |   `-- bytes
|               |-- ...
|               `-- class_n/
|                   |-- packets
|                   `-- bytes
`-- control_plane/
    `-- ingress/
        |-- traffic/
        |   |-- packets
        |   `-- bytes
        `-- discards/
            |-- packets
            |-- bytes
            `-- policy/
                `-- packets
            
~~~~~~~~~~

For additional context, Appendix A provides an example of where packets may be dropped in a device.


Discard Class Descriptions {#class_descriptions}
--------------------------

discards/policy/  
    These are intended discards, meaning packets dropped due to a configured policy. There are multiple sub-classes.

discards/error/l2/rx/  
    Frames dropped due to errors in the received L2 frame. There are multiple sub-classes, such as those resulting from failing CRC, invalid header, invalid MAC address, or invalid VLAN.

discards/error/l3/rx/  
    These drops occur due to errors in the received packet, indicating an upstream problem rather than an issue with the device dropping the errored packets. There are multiple sub-classes, including header checksum errors, MTU exceeded, and invalid packet, i.e. due to incorrect version, incorrect header length, or invalid options.
    
discards/error/l3/rx/ttl_expired  
    There can be multiple causes for TTL-exceed drops: i) trace-route; ii) TTL set too low by the end-system; iii) routing loops. 
    
discards/error/l3/no_route/  
    Discards occur due to a packet not matching any route.
    
discards/error/local/  
    A device may drop packets within its switching pipeline due to internal errors, such as parity errors. Any errored discards not explicitly assigned to the above classes are also accounted for here.

discards/no_buffer/  
    Discards occur due to no available buffer to enqueue the packet. These can be tail-drop discards or due to an active queue management algorithm, such as RED {{RED93}} or CODEL {{RFC8289}}.


Semantics {#semantics}
---------
Rules 1-10 relate to packets forwarded by the device; rule 11 relates to packets destined to/from the device:

1. All instances of frame or packet receipt, transmission, and drops MUST be reported.
2. All instances of frame or packet receipt, transmission, and drops SHOULD be attributed to the physical or logical interface of the device where they occur.
3. An individual frame MUST only be accounted for by either the L2 traffic class or the L2 discard classes within a single direction, i.e., ingress or egress.
4. An individual packet MUST only be accounted for by either the L3 traffic class or the L3 discard classes within a single direction, i.e., ingress or egress.
5. A frame accounted for at L2 MUST NOT be accounted for at L3 and vice versa
6. The aggregate L2 and L3 traffic and discard classes MUST account for all underlying packets received, transmitted, and dropped across all other classes.
7. The aggregate qos traffic and discard (no buffer) classes MUST account for all underlying packets received, transmitted, and dropped across all other classes.
8. In addition to the L2 and L3 aggregate classes, an individual dropped packet MUST only account against a single error, policy, or no_buffer discard subclass.
9. When there are multiple drop reasons for a packet, the ordering of discard class reporting MUST be defined.
10. If Diffserv {{RFC2475}} quality of service (QOS) is not used, no_buffer discards SHOULD be reported as class0.
11. Traffic to the device control plane has its own class, however, traffic from the device control plane SHOULD be accounted for in the same way as other egress traffic.  


Examples {#examples}
--------

Assuming all the requirements are met, a good unicast IPv4 packet received would increment:
- interface/ingress/traffic/l3/v4/unicast/packets  
- interface/ingress/traffic/l3/v4/unicast/bytes  
- interface/ingress/traffic/qos/class_0/packets  
- interface/ingress/traffic/qos/class_0/bytes  

A received unicast IPv6 packet dropped due to TTL expiry would increment:  
- interface/ingress/discards/l3/v6/unicast/packets  
- interface/ingress/discards/l3/v6/unicast/bytes  
- interface/ingress/discards/l3/rx/ttl_expired/packets  

An IPv4 packet dropped on egress due to no buffers would increment:
- interface/egress/discards/l3/v4/unicast/packets  
- interface/egress/discards/l3/v4/unicast/bytes  
- interface/egress/discards/no_buffer/class_0/packets  
- interface/egress/discards/no_buffer/class_0/bytes  


A Possible Signal-Cause-Mitigation Mapping {#mapping}
==========================================

Example discard signal-to-cause-to-mitigation mappings are shown in the table below:

~~~~~~~~~~
+-------------------------------------------+---------------------+------------+----------+-------------+-----------------------+
| Discard class                             | Cause               | Discard    | Discard  | Unintended? | Possible actions      |
|                                           |                     | rate       | duration |             |                       |
+-------------------------------------------+---------------------+------------+----------+-------------+-----------------------+
| ingress/discards/errors/l2/rx             | Upstream device     | >Baseline  | O(1min)  | Y           | Take upstream link or |
|                                           | or link errror      |            |          |             | device out-of-service |
| ingress/discards/errors/l3/rx/ttl_expired | Tracert             | <=Baseline |          | N           | no action             |
| ingress/discards/errors/l3/rx/ttl_expired | Convergence         | >Baseline  | O(1s)    | Y           | no action             |
| ingress/discards/errors/l3/rx/ttl_expired | Routing loop        | >Baseline  | O(1min)  | Y           | Roll-back change      |
| .*/policy/.*                              | Policy              |            |          | N           | no action             |
| ingress/discards/errors/l3/no_route       | Convergence         | >Baseline  | O(1s)    | Y           | no action             |
| ingress/discards/errors/l3/no_route       | Config error        | >Baseline  | O(1min)  | Y           | Roll-back change      |
| ingress/discards/errors/l3/no_route       | Invalid destination | >Baseline  | O(10min) | N           | Escalate to operator  |
| ingress/discards/errors/local             | Device errors       | >Baseline  | O(1min)  | Y           | Take device           |
|                                           |                     |            |          |             | out-of-service        |
| egress/discards/no_buffer                 | Congestion          | <=Baseline |          | N           | no action             |
| egress/discards/no_buffer                 | Congestion          | >Baseline  | O(1min)  | Y           | Bring capacity back   |
|                                           |                     |            |          |             | into service or move  |
|                                           |                     |            |          |             | traffic               |
+-------------------------------------------+---------------------+------------+----------+-------------+-----------------------+

~~~~~~~~~~

The 'Baseline' in the 'Discard Rate' column is network dependent.


Security Considerations {#security}
=======================
There are no new security considerations introduced by this document.


IANA Considerations {#iana}
===================
There are no new IANA considerations introduced by this document.


Terminology {#terminology}
===========

{::boilerplate bcp14-tagged}


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
The content of this draft has benefitted from feedback from JR Rivers, Ronan Waide, Chris DeBruin, and Marcoz Sanz.

--- back

Where do packets get dropped? 
=============================
The diagram below is an example of where and why packets may be dropped in a typical single ASIC, shared buffered type device, where packets ingress on the left and egress on the right.

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
                                         null_route

Unintended                 error/rx/l2   error/rx/l3   no_buffer     error/tx/l3
  Discards:                              error/local
                                         no_route
                                         ttl

~~~~~~~~~~

Implementation Experience {#experience}
=========================
This appendix captures the authors' experience gained from implementing and applying this information model across multiple vendors' platforms, as guidance for future implementers.

1. The number and granularity of classes described in Section 3 represent a compromise.  It aims to offer sufficient detail to enable appropriate automated actions while avoiding excessive detail which may hinder quick problem identification.  Additionally, it helps constrain the quantity of data produced per interface to manage data volume and device CPU impacts.  Although further granularity is possible, the scheme described has generally proven to be sufficient.
2. There are multiple possible ways to define the discard classification tree.  For example,  we could have used a multi-rooted tree, rooted in each protocol.  Instead we opted to define a tree where protocol discards and causal discards are accounted for orthogonally.  This decision reduces the number of combinations of classes and has proven sufficient for determining mitigation actions.
3. NoBuffer discards can be realized differently with different memory architectures. Hence, whether a NoBuffer discard is attributed to ingress or egress can differ accordingly.  For successful auto-mitigation, discards due to egress interface congestion should be reported on egress, while discards due to device-level congestion (exceeding the device forwarding rate) should be reported on ingress.
4. Platforms often account for the number of packets dropped where the TTL has expired, and the CPU has returned an ICMP Time Exceeded message.  There is typically a policer applied to limit the number of packets sent to the CPU, however, which implicitly limits the rate of TTL discards that are processed.  One method to account for all packet discards due to TTL exceeded, even those that are dropped by a policer when being forwarded to the CPU, is to use accounting of all ingress packets received with TTL=1.
5. Where no route discards are implemented with a default null route, separate discard accounting is required for any explicit null routes configured, in order to differentiate between interface/ingress/discards/policy/null_route/packets and interface/ingress/discards/errors/no_route/packets.
6. It is useful to account separately for transit packets dropped by transit ACLs or policers, and packets dropped by ACLs or policers which limit the number of packets to the device control plane.
7. It is not possible to identify a configuration error - e.g., when intended discards are unintended - with device packet loss metrics alone.  For example, to determine if ACL drops are intended or due to a misconfigured ACL some other method is needed, i.e., with configuration validation before deployment or by detecting a significant change in ACL drops after a change compared to before.
8. Where traffic byte counters need to be 64-bit, packet and discard counters that increase at a lower rate may be encoded in fewer bits, e.g., 48-bit.
9. In cases where the reporting device is the source or destination of a tunnel, the ingress protocol for a packet may differ from the egress protocol; if IPv4 is tunneled over IPv6 for example.  Some implementations may attribute egress discards to the ingress protocol.
10. While the classification tree is seven layers deep, a minimal implementation may only implement the top six layers.

