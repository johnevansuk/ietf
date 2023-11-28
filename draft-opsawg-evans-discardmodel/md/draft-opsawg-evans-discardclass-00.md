---
title: An Information Model for Packet Discard Reporting
abbrev: Info. Model for Pkt Discard Reporting
docname: draft-opsawg-evans-discardmodel-00
date: 2023-10-05
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

Router reported packet loss is the primary signal of when a network is not doing its job.  Some packet loss is normal or intended in TCP/IP networks, however.  To minimise network packet loss through automated network operations we need clear and accurate signals of all packets which are dropped and why.  This document defines an information model for packet loss reporting, which classifies these signals to enable automated network mitigation of unintended packet loss.

--- middle

Introduction        {#introduction}
============

The job of a network is to transport packets.  Understanding both where and why packet loss occurs is essential for effective network operation.   Router-reported packet loss is the most direct signal for network operations to identify customer impact from unintended packet loss. Accurate accounting of packet loss is not enough, however, as some level of packet loss is normal in TCP/IP networks.  In automating network operations, there are only a relatively small number of automated actions that can be taken to mitigate customer impacting packet loss. Precise classification of packet loss signals is important to ensure the right action is taken as taking the wrong action can make problems worse. 

The existing metrics for packet loss as defined in {{RFC1213}} - namely ifInDiscards, ifOutDiscards, ifInErrors, ifOutErrors - do not provide sufficient precision to be able to automatically identify the cause of the loss and mitigate the impact.  From a network operators' perspective, ifindiscards can represent both intended packet loss (i.e., packets discarded due to policy) and unintended packet loss (e.g., packets dropped in error). Further, these definitions are ambiguous, in that vendors can and have implemented them differently.  In some implementations, ifinerrors accounts only for errored packets which are dropped, whilst in others it accounts for all errored packets whether they are dropped or not.  Many implementations support more discard metrics than these; where they do, they are inconsistently implemented due to an absence of a clearly defined classification scheme and semantics for packet loss reporting.

Hence, this document defines an information model for packet loss reporting, which aims to address these issues by presenting a packet loss classification scheme that can enable automated mitigation of unintended packet loss.  This information model is independent of any specific implementations or protocols used to transport the data {{RFC3444}}.  There are multiple ways that this information model could be implemented, including SNMP {{RFC1157}}, IPFIX {{RFC5153}}, and NETCONF {{RFC6241}} / YANG {{RFC7950}}, but this document does not define specific data models.

Section 2 describes the problem.  Section 3 defines the information model and semantics with examples.  Section 4 gives examples of discard signal-to-cause-to-auto-mitigation action mapping.  Appendix B details our experience from implementing this model.

The terms 'packet drop' and 'discard' are considered equivalent and are used interchangeably.

Problem statement   {#problem}
=================

Working backwards from the goal of auto-mitigation of unintended packet loss, there are only a relatively small number of potential auto-mitigation actions, e.g.:

1. Take a device, link or set of devices and/or links out of service
2. Return a device, link or set of devices and/or links back into service
3. Move traffic to another device
4. Roll-back a recent change to a device that might have caused the problem
5. Escalate to a human (e.g., network operators) as a last resort

Precise signal of impact is important as taking the wrong action can be worse than taking no action.  For example, taking a congested device out of service can make congestion worse by moving the traffic to other already congested links and/or devices.

To be able to detect whether router reported packet loss is a problem and determine what actions should be taken to mitigate the impact and remediate the cause, depends on four primary features of the packet loss signal:

1. the cause of the loss
2. the rate and/or degree of the loss
3. the duration of the loss
4. the location of the loss

Features 2, 3 and 4 are already addressed with passive monitoring statistics, e.g., obtained with SNMP {{RFC1157}} / MIB-II {{RFC1213}} or NETCONF {{RFC6241}} / YANG {{RFC7950}}.  Feature 1, however, is dependent on the classification scheme used for packet loss reporting.  In the next section we define a new classification scheme to address this problem.


Information model   {#model}
=================

The classification scheme is defined as a tree which follows the structure &lt;component&gt;&lt;direction&gt;&lt;type&gt;&lt;layer&gt;&lt;sub-type&gt;&lt;sub-sub-type&gt;&lt;metric&gt;, where:  
a. component can be interface|device  
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
|   |       |       `-- urpf/
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
    |-- traffic/
    |   |-- packets
    |   `-- bytes
    `-- discards/
        |-- packets
        |-- bytes
        `-- policy/
            |-- acl/
            |   `-- packets
            `-- policer/
                `-- packets
            
~~~~~~~~~~

For additional context, Appendix A provides an example of where packets may be dropped in a device.


Discard Class Descriptions {#class_descriptions}
--------------------------

discards/policy/  
     These are intended discards, i.e., packets dropped due to a configured policy. There are multiple sub-classes.

discards/error/l2/rx/  
     Frames dropped due to errors in the received L2 frame.  There are multiple sub-classes e.g., due to failing CRC, invalid header, invalid MAC address, invalid VLAN.

discards/error/l3/rx/  
    These are drops due to errors in the received packet, i.e., which indicate an upstream problem, rather than a problem with the device that is dropping the errored packets. There are multiple sub-classes, e.g., header checksum errors, MTU exceeded, incorrect version, incorrect header length, invalid options.
    
discards/error/l3/rx/ttl_expired  
     There can also be multiple causes for TTL-exceed drops: i) trace-route; ii) TTL set too low by the end-system; iii) routing loops
    
discards/error/l3/no_route/  
     Discards due to a packet not matching any route.
    
discards/error/local/  
    A device may drop packets within its switching pipeline due to internal errors, e.g., parity errors. Any errored discards not explicitly assigned to the above classes are accounted here.

discards/no_buffer/  
     Discards due to no available buffer to enqueue the packet. These can be tail-drop discards or due to an active queue management algorithm, e.g., RED {{RED93}}, CODEL {{RFC8289}}.


Semantics {#semantics}
---------
1-10 below apply to the packets forwarded by the device, i.e., rather than packets destined to/from the device:

1. All packet receipt, transmission and drops MUST be reported
2. All packet receipt, transmission and drops SHOULD be attributed to the physical or logical interface where they occur.
3. If a frame is discarded at L2, it MUST NOT be accounted for at L3
4. An individual packet MUST NOT account against both the L2 traffic and L2 discard classes on a single direction, i.e., ingress or egress
5. An individual packet MUST NOT account against both the L3 traffic and L3 discard classes on a single direction, i.e., ingress or egress
6. The aggregate L2 and L3 traffic and discard classes MUST account for all underlying packets received, transmitted and dropped across all other classes
7. The aggregate QOS traffic and discard (no buffer) classes MUST account for all underlying packets received, transmitted and dropped across all other classes
8. In addition to the L2 and L3 aggregate classes, an individual dropped packet MUST only account against a single error, policy or no buffer discard sub class
9. Where there may be multiple drop reasons for a packet, the ordering of discard class reporting MUST be defined
10. If Diffserv {{RFC2475}} quality of service (QOS) is not used, no_buffer discards SHOULD be reported as class0
11. Traffic from the device control plane SHOULD be accounted for the same as other egress traffic


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
The authors would like to thank Nadav Chachmon for his valuable contribution to this work.

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
This appendix captures experience gained from implementing this information model, as guidance for future implementers.

1. The number and granularity of classes described in section 3 is a compromise between: providing sufficient detail to be able to take the appropriate automated actions whilst: a) not providing too much detail which may require deeper understanding rather than helping to surface the problem quickly; b) constraining the quantity of data produced where these metrics are produced per interface to limit data volume and device CPU impacts.  While further granularity is possible, we found the scheme described to be generally sufficient.
2. There are multiple ways that we could have defined the discard classification tree, e.g., we could have used a multi-rooted tree, rooted in each protocol.  We opted instead to define a tree where protocol discards and causal discards are accounted orthogonally, as this reduces the number of classes and we found it sufficient to determine mitigation actions.
3. NoBuffer discards can be realised differently with different memory architectures.  Whether a NoBuffer discard is attributed to ingress or egress can differ accordingly.  For successful auto-mitigation: where the discards are due to egress interface congestion, they should be reported on egress; where the discards are due to device-level congestion (exceeding the device forwarding rate), they should be reported on ingress. 
4. Most platforms account for the number of packets where the TTL has expired, and the CPU has returned an ICMP Time Exceeded message. In practise, however, there is often a policer applied to limit the number of packets to the CPU.  Implicitly, this limits the rate of TTL discards processed by the CPU and hence it limits the number of discards reported.  One method to account for all packets discards due to TTL exceeded, even those that are dropped by a policer when being forwarded to the CPU, is to use accounting of all ingress packets received with TTL=1.
5. Where a no route discard is implemented with a default null route, separate accounting is needed for any explicit null routes configured, in order to differentiate between interface/ingress/discards/policy/null_route/packets and interface/ingress/discards/errors/no_route/packets.
6. It is useful to account separately for transit packets dropped by transit ACLs or policers, and packets dropped by ACLs or policers which limit the number of packets to the device control packets.
7. It is not possible to identify a configuration error - i.e., when intended discards are unintended - with device packet loss metrics alone.  For example, to determine if ACL drops are intended or due to a misconfigured ACL some other method is needed, e.g., with configuration validation before deployment or in detecting a significant change in ACL drops after a change compared to before.
8. Where traffic byte counters need to be 64-bit, packet and discard counters which increase at a lower rate may be encoded in fewer bits, e.g., 48-bit.
9. Where the reporting device is the source or destination of a tunnel, the ingress protocol for a packet may be different to the egress protocol, e.g., if IPv4 is tunnelled over IPv6.  In this case, some implementations may attribute egress discards to the ingress protocol.


