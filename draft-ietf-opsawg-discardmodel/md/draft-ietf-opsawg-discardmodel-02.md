---
title: An Information Model for Packet Discard Reporting
abbrev: Info. Model for Pkt Discard Reporting
docname: draft-ietf-opsawg-discardmodel-02
date: 2024-07-04
category: info

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
     RFC2119:
     RFC8791:

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
     RFC7270:
     
--- abstract

The primary function of a network is to transport packets and deliver them according to a service level objective.  Understanding both where and why packet loss occurs within a network is essential for effective network operation.  Device-reported packet loss is the most direct signal for network operations to identify customer impact resulting from unintended packet loss.  This document defines an information model for packet loss reporting, which classifies these signals to enable automated network mitigation of unintended packet loss.

--- middle

Introduction        {#introduction}
============

In automating network operations, a network operator needs to be able to detect anomalous packet loss, diagnose or root cause the loss, and then apply one of a set of possible actions to mitigate customer-impacting packet loss.  Some packet loss is normal or intended in IP networks, however.  Hence, precise classification of packet loss signals is crucial both to ensure that anomalous packet loss is easily detected and that the right action or sequence of actions are taken to mitigate the impact, as taking the wrong action can make problems worse.

The existing metrics for reporting packet loss, as defined in {{RFC1213}} - namely ifInDiscards, ifOutDiscards, ifInErrors, ifOutErrors - do not provide sufficient precision to automatically identify the cause of the loss and mitigate the impact.  From a network operator's perspective, ifInDiscards can represent both intended packet loss (e.g., packets discarded due to policy) and unintended packet loss (e.g., packets dropped in error). Furthermore, these definitions are ambiguous, as vendors can and have implemented them differently.  In some implementations, ifInErrors accounts only for errored packets that are dropped, while in others, it accounts for all errored packets, whether they are dropped or not.  Many implementations support more discard metrics than these; where they do, they have been inconsistently implemented due to the lack of a standardised classification scheme and clear semantics for packet loss reporting.  {{RFC7270}} provides support for reporting discards per flow in IPFIX using forwardingStatus, however, the defined drop reason codes also lack sufficient clarity to support automated root cause analysis and mitigation of impact.

Hence, this document defines an information model for packet loss reporting, aiming to address these issues by presenting a packet loss classification scheme that can enable automated mitigation of unintended packet loss.  Consistent with {{RFC3444}}, this information model is independent of any specific implementations or protocols used to transport the data.  There are multiple ways that this information model could be implemented (i.e., data models), including SNMP {{RFC1157}}, NETCONF {{RFC6241}} / YANG {{RFC7950}}, and IPFIX {{RFC5153}}, but they are outside of the scope of this document.  We further limit the scope of this document to reporting packet loss at layer 3 and frames discarded at layer 2, although the information model could be extended in future to cover segments dropped at layer 4. 

Section 3 describes the problem. Section 4 describes the information model and requirements with examples.  Section 5 provides examples of discard signal-to-cause-to-auto-mitigation action mapping.  Section 6 presents the information model as an abstract data structure in YANG, in coordance with {{RFC8791}}.  Appendix A provides an example of where packets may be discarded in a device.  Appendix B details the authors' experience from implementing this model.

This document considers only the signals that may trigger automated mitigation plans and not how they are defined or executed.

Terminology {#terminology}
===========

{::boilerplate bcp14-tagged}

A packet discard is considered to be any packet dropped by a device, which may be intentional (i.e. due to a configured policy, e.g. such as an Access Control List (ACL)) or unintentional (i.e. packets dropped in error).


Problem Statement   {#problem}
=================
At the highest-level, unintended packet loss is the discarding of packets that the network operator otherwise intends to deliver, i.e. which indicates an error state.  There are many possible reasons for unintended packet loss, including: erroring links may corrupt packets in transit; incorrect routing tables may result in packets being dropped because they do not match a valid route; configuration errors may result in a valid packet incorrectly matching an access control list (ACL) and being dropped.  Whilst the specific definition of unintended packet loss is network dependent, for any network there are a small set of potential actions that can be taken to minimise customer impact by auto-mitigating unintended packet loss:

1. Take a device, link, or set of devices and/or links out of service.
2. Return a device, link, or set of devices and/or links back into service.
3. Move traffic to other links or devices.
4. Roll back a recent change to a device that might have caused the problem.
5. Escalate to a human (e.g., network operator) as a last resort.

A precise signal of impact is crucial, as taking the wrong action can be worse than taking no action. For example, taking a congested device out of service can make congestion worse by moving the traffic to other links or devices, which are already congested.

To detect whether device-reported discards indicate a problem and to determine what actions should be taken to mitigate the impact and remediate the cause, depends on four primary features of the packet loss signal:

1. The cause of the loss.
2. The rate and/or degree of the loss.
3. The duration of the loss.
4. The location of the loss.

Features 2, 3, and 4 are already addressed with passive monitoring statistics, for example, obtained with SNMP {{RFC1157}} / MIB-II {{RFC1213}} or NETCONF {{RFC6241}} / YANG {{RFC7950}}. Feature 1, however, is dependent on the classification scheme used for packet loss reporting. In the next section, we define a new classification scheme to address this problem.


Information Model   {#model}
=================

The classification scheme is defined as a tree, shown in {{discard-tree}}, which follows the structure component/direction/type/layer/sub-type/sub-sub-type/.../metric, where:  
a. component can be interface|device|control_plane|flow  
b. direction can be ingress|egress  
c. type can be traffic|discards, where traffic accounts for packets successfully received or transmitted, and discards accounts for packet drops  
d. layer can be l2|l3

~~~~~~~~~~
  structure name1-here:
    +-- ingress
    |  +-- traffic
    |  |  +-- l2
    |  |  |  +-- frames?   uint64
    |  |  |  +-- bytes?    uint64
    |  |  +-- l3
    |  |  |  +-- v4
    |  |  |  |  +-- packets?     uint64
    |  |  |  |  +-- bytes?       uint64
    |  |  |  |  +-- unicast
    |  |  |  |  |  +-- packets?   uint64
    |  |  |  |  |  +-- bytes?     uint64
    |  |  |  |  +-- multicast
    |  |  |  |     +-- packets?   uint64
    |  |  |  |     +-- bytes?     uint64
    |  |  |  +-- v6
    |  |  |     +-- packets?     uint64
    |  |  |     +-- bytes?       uint64
    |  |  |     +-- unicast
    |  |  |     |  +-- packets?   uint64
    |  |  |     |  +-- bytes?     uint64
    |  |  |     +-- multicast
    |  |  |        +-- packets?   uint64
    |  |  |        +-- bytes?     uint64
    |  |  +-- qos
    |  |     +-- class* []
    |  |        +-- id?        string
    |  |        +-- packets?   uint64
    |  |        +-- bytes?     uint64
    |  +-- discards
    |     +-- l2
    |     |  +-- frames?   uint64
    |     |  +-- bytes?    uint64
    |     +-- l3
    |     |  +-- v4
    |     |  |  +-- packets?     uint64
    |     |  |  +-- bytes?       uint64
    |     |  |  +-- unicast
    |     |  |  |  +-- packets?   uint64
    |     |  |  |  +-- bytes?     uint64
    |     |  |  +-- multicast
    |     |  |     +-- packets?   uint64
    |     |  |     +-- bytes?     uint64
    |     |  +-- v6
    |     |     +-- packets?     uint64
    |     |     +-- bytes?       uint64
    |     |     +-- unicast
    |     |     |  +-- packets?   uint64
    |     |     |  +-- bytes?     uint64
    |     |     +-- multicast
    |     |        +-- packets?   uint64
    |     |        +-- bytes?     uint64
    |     +-- errors
    |     |  +-- l2
    |     |  |  +-- rx
    |     |  |     +-- frames?          uint48
    |     |  |     +-- crc-error?       uint48
    |     |  |     +-- invalid-mac?     uint48
    |     |  |     +-- invalid-vlan?    uint48
    |     |  |     +-- invalid-frame?   uint48
    |     |  +-- l3
    |     |  |  +-- rx
    |     |  |  |  +-- packets?          uint48
    |     |  |  |  +-- checksum-error?   uint48
    |     |  |  |  +-- mtu-exceeded?     uint48
    |     |  |  |  +-- invalid-packet?   uint48
    |     |  |  |  +-- ttl-expired?      uint48
    |     |  |  +-- no-route?        uint48
    |     |  |  +-- invalid-sid?     uint48
    |     |  |  +-- invalid-label?   uint48
    |     |  +-- local
    |     |     +-- packets?   uint48
    |     |     +-- hw
    |     |        +-- packets?        uint48
    |     |        +-- parity-error?   uint48
    |     +-- policy
    |     |  +-- l2
    |     |  |  +-- frames?   uint48
    |     |  |  +-- acl?      uint48
    |     |  +-- l3
    |     |     +-- packets?      uint48
    |     |     +-- acl?          uint48
    |     |     +-- policer
    |     |     |  +-- packets?   uint48
    |     |     |  +-- bytes?     uint48
    |     |     +-- null-route?   uint48
    |     |     +-- rpf?          uint48
    |     |     +-- ddos?         uint48
    |     +-- no-buffer
    |        +-- class* [id]
    |           +-- id         string
    |           +-- packets?   uint64
    |           +-- bytes?     uint64
    +-- egress
    |  +-- traffic
    |  |  +-- l2
    |  |  |  +-- frames?   uint64
    |  |  |  +-- bytes?    uint64
    |  |  +-- l3
    |  |  |  +-- v4
    |  |  |  |  +-- packets?     uint64
    |  |  |  |  +-- bytes?       uint64
    |  |  |  |  +-- unicast
    |  |  |  |  |  +-- packets?   uint64
    |  |  |  |  |  +-- bytes?     uint64
    |  |  |  |  +-- multicast
    |  |  |  |     +-- packets?   uint64
    |  |  |  |     +-- bytes?     uint64
    |  |  |  +-- v6
    |  |  |     +-- packets?     uint64
    |  |  |     +-- bytes?       uint64
    |  |  |     +-- unicast
    |  |  |     |  +-- packets?   uint64
    |  |  |     |  +-- bytes?     uint64
    |  |  |     +-- multicast
    |  |  |        +-- packets?   uint64
    |  |  |        +-- bytes?     uint64
    |  |  +-- qos
    |  |     +-- class* [id]
    |  |        +-- id         string
    |  |        +-- packets?   uint64
    |  |        +-- bytes?     uint64
    |  +-- discards
    |     +-- l2
    |     |  +-- frames?   uint64
    |     |  +-- bytes?    uint64
    |     +-- l3
    |     |  +-- v4
    |     |  |  +-- packets?     uint64
    |     |  |  +-- bytes?       uint64
    |     |  |  +-- unicast
    |     |  |  |  +-- packets?   uint64
    |     |  |  |  +-- bytes?     uint64
    |     |  |  +-- multicast
    |     |  |     +-- packets?   uint64
    |     |  |     +-- bytes?     uint64
    |     |  +-- v6
    |     |     +-- packets?     uint64
    |     |     +-- bytes?       uint64
    |     |     +-- unicast
    |     |     |  +-- packets?   uint64
    |     |     |  +-- bytes?     uint64
    |     |     +-- multicast
    |     |        +-- packets?   uint64
    |     |        +-- bytes?     uint64
    |     +-- errors
    |     |  +-- l2
    |     |  |  +-- tx
    |     |  |     +-- frames?   uint48
    |     |  +-- l3
    |     |     +-- tx
    |     |        +-- packets?   uint48
    |     +-- policy
    |     |  +-- l3
    |     |     +-- acl?       uint48
    |     |     +-- policer
    |     |        +-- packets?   uint48
    |     |        +-- bytes?     uint48
    |     +-- no-buffer
    |        +-- class* [id]
    |           +-- id         string
    |           +-- packets?   uint64
    |           +-- bytes?     uint64
    +-- control-plane
       +-- ingress
          +-- traffic
          |  +-- packets?   uint48
          |  +-- bytes?     uint48
          +-- discards
             +-- packets?   uint48
             +-- bytes?     uint48
             +-- policy?    uint48

~~~~~~~~~~

{: #discard-tree title="Discard CLassification tree"}

For additional context, Appendix A provides an example of where packets may be discarded in a device.


Requirements {#requirements}
------------
Requirements 1-10 relate to packets forwarded by the device; requirement 11 relates to packets destined to or from the device:

1. All instances of frame or packet receipt, transmission, and discards MUST be reported.
2. All instances of frame or packet receipt, transmission, and discards SHOULD be attributed to the physical or logical interface of the device where they occur.
3. An individual frame MUST only be accounted for by either the L2 traffic class or the L2 discard classes within a single direction, i.e., ingress or egress.
4. An individual packet MUST only be accounted for by either the L3 traffic class or the L3 discard classes within a single direction, i.e., ingress or egress.
5. A frame accounted for at L2 SHOULD NOT be accounted for at L3 and vice versa.  An implementation MUST expose which layers a discard is counted against.
6. The aggregate L2 and L3 traffic and discard classes SHOULD account for all underlying packets received, transmitted, and discarded across all other classes.
7. The aggregate Quality of Service (QoS) traffic and no buffer discard classes MUST account for all underlying packets received, transmitted, and discarded across all other classes.
8. In addition to the L2 and L3 aggregate classes, an individual discarded packet MUST only account against a single error, policy, or no_buffer discard subclass.
9. When there are multiple reasons for discarding a packet, the ordering of discard class reporting MUST be defined.
10. If Diffserv {{RFC2475}} is not used, no_buffer discards SHOULD be reported as class0.
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
- interface/ingress/discards/l3/rx/ttl_expired/packets  

An IPv4 packet discarded on egress due to no buffers would increment:  
- interface/egress/discards/l3/v4/unicast/packets  
- interface/egress/discards/l3/v4/unicast/bytes  
- interface/egress/discards/no_buffer/class_0/packets  
- interface/egress/discards/no_buffer/class_0/bytes  


Example Signal-Cause-Mitigation Mapping {#mapping}
=======================================
{{ex-table}} gives an example discard signal-to-cause-to-mitigation action mapping.  Mappings for a specific network will be dependent on the definition of unintended packet loss for that network.

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
{: #ex-table title="Example Signal-Cause-Mitigation Mapping"}

The 'Baseline' in the 'Discard Rate' column is network dependent.

Yang Module
===========
~~~~~~~~~~
module ietf-packet-discard-reporting {
  yang-version 1.1;
  namespace
    "urn:ietf:params:xml:ns:yang:ietf-packet-discard-reporting";
  prefix plr;

  import ietf-yang-structure-ext {
    prefix sx;
    reference
      "RFC 8791: YANG Data Structure Extensions";
  }

  organization
    "IETF OPSAWG (Operations and Management Area Working Group)";
  contact
    "WG Web:   https://datatracker.ietf.org/wg/opsawg/
     WG List:  mailto:opsawg@ietf.org

     Author:   John Evans
               <mailto:jevanamz@amazon.co.uk>

     Author:   Oleksandr Pylypenko
               <mailto:opyl@amazon.com>

     Author:   Jeffrey Haas
               <mailto:jhaas@juniper.net>

     Author:   Aviran Kadosh
               <mailto:akadosh@cisco.com>

     Author:   Mohamed Boucadair
                  <mailto:mohamed.boucadair@orange.com>";

  description
    "This module defines an information model for packet discard reporting.";

  revision 2024-06-04 {
    description
      "Initial revision.";
    reference
      "draft-ietf-opsawg-discardmodel: An Information Model for Packet Discard Reporting";
  }

  typedef uint48-or-64 {
    description 
      "Union type representing either a 48-bit or 64-bit unsigned integer. 48-bit counters are used for packet and discard counters that increase at a lower rate, while 64-bit counters are used for traffic byte counters that may increase more rapidly."; 
    type union {
      type uint48; type uint64;
    }
  }

  typedef uint48 {
    type uint64 {
      range "0..281474976710655";
    }
  }


  /*
   * Groupings
   */

  grouping ip {
    description
      "IP traffic counters";
    leaf packets {
      type uint64;
      description
        "Number of IPv4 packets";
    }
    leaf bytes {
      type uint64;
      description
        "Number of IPv4 bytes";
    }
    container unicast {
      description
        "IPv4 unicast ingress traffic counters";
      leaf packets {
        type uint64;
        description
          "Number of IPv4 unicast packets";
      }
      leaf bytes {
        type uint64;
        description
          "Number of IPv4 unicast bytes";
      }
    }
    container multicast {
      description
        "IPv4 multicast ingress traffic counters";
      leaf packets {
        type uint64;
        description
          "Number of IPv4 multicast packets";
      }
      leaf bytes {
        type uint64;
        description
          "Number of IPv4 multicast bytes";
      }
    }
  }

  grouping interface {
    description
      "Interface-level packet loss counters";
    container ingress {
      description
        "Ingress counters";
      container traffic {
        description
          "Ingress traffic counters";
        container l2 {
          description
            "Layer 2 ingress traffic counters";
          leaf frames {
            type uint64;
            description
              "Number of L2 frames";
          }
          leaf bytes {
            type uint64;
            description
              "Number of L2 bytes";
          }
        }
        container l3 {
          description
            "Layer 3 ingress traffic counters";
          container v4 {
            description
              "IPv4 ingress traffic counters";
            uses ip;
          }
          container v6 {
            description
              "IPv6 ingress traffic counters";
            uses ip;
          }
        }
        container qos {
          description
            "Quality of Service (QoS) ingress traffic counters";
          list class {
            key "id";
            min-elements 1;
            description
              "QoS class ingress traffic counters";
            leaf id {
              type string;
              description
                "QoS class identifier";
            }
            leaf packets {
              type uint64;
              description
                "Number of packets in the QoS class";
            }
            leaf bytes {
              type uint64;
              description
                "Number of bytes in the QoS class";
            }
          }
        }
      }
      container discards {
        description
          "Ingress packet discard counters";
        container l2 {
          description
            "Layer 2 ingress packet discard counters";
          leaf frames {
            type uint64;
            description
              "Number of discarded L2 frames";
          }
          leaf bytes {
            type uint64;
            description
              "Number of discarded L2 bytes";
          }
        }
        container l3 {
          description
            "Layer 3 ingress packet discard counters";
          container v4 {
            description
              "IPv4 ingress packet discard counters";
            uses ip;
          }
          container v6 {
            description
              "IPv6 ingress packet discard counters";
            uses ip;
          }
        }
        container errors {
          description
            "Ingress packet error counters";
          container l2 {
            description
              "Layer 2 ingress packet error counters";
            container rx {
              description
                "Layer 2 ingress packet receive error counters";
              leaf frames {
                type uint48;
                description
                  "Number of errored L2 frames";
              }
              leaf crc-error {
                type uint48;
                description
                  "Number of frames received with CRC error";
              }
              leaf invalid-mac {
                type uint48;
                description
                  "Number of frames received with invalid MAC address";
              }
              leaf invalid-vlan {
                type uint48;
                description
                  "Number of frames received with invalid VLAN tag";
              }
              leaf invalid-frame {
                type uint48;
                description
                  "Number of invalid frames received";
              }
            }
          }
          container l3 {
            description
              "Layer 3 ingress packet error counters";
            container rx {
              description
                "Layer 3 ingress packet receive error counters";
              leaf packets {
                type uint48;
                description
                  "Number of errored L3 packets";
              }
              leaf checksum-error {
                type uint48;
                description
                  "Number of packets received with checksum error";
              }
              leaf mtu-exceeded {
                type uint48;
                description
                  "Number of packets received exceeding MTU";
              }
              leaf invalid-packet {
                type uint48;
                description
                  "Number of invalid packets received";
              }
              leaf ttl-expired {
                type uint48;
                description
                  "Number of packets received with expired TTL";
              }
            }
            leaf no-route {
              type uint48;
              description
                "Number of packets with no route";
            }
            leaf invalid-sid {
              type uint48;
              description
                "Number of packets with invalid SID";
            }
            leaf invalid-label {
              type uint48;
              description
                "Number of packets with invalid label";
            }
          }
          container hardware {
            description
              "Hardware error counters";
            leaf packets {
              type uint48;
              description
                "Number of local errored packets";
            }
            leaf parity-error {
              type uint48;
              description
                "Number of packets with parity error";
            }
          }
        }
        container policy {
          description
            "Policy-related ingress packet discard counters";
          container l2 {
            description
              "Layer 2 policy ingress packet discard counters";
            leaf frames {
              type uint48;
              description
                "Number of L2 frames discarded due to policy";
            }
            leaf acl {
              type uint48;
              description
                "Number of frames discarded due to L2 ACL";
            }
          }
          container l3 {
            description
              "Layer 3 policy ingress packet discard counters";
            leaf packets {
              type uint48;
              description
                "Number of L3 packets discarded due to policy";
            }
            leaf acl {
              type uint48;
              description
                "Number of packets discarded due to L3 ACL";
            }
            container policer {
              description
                "Policer ingress packet discard counters";
              leaf packets {
                type uint48;
                description
                  "Number of packets discarded by the policer";
              }
              leaf bytes {
                type uint48;
                description
                  "Number of bytes discarded by the policer";
              }
            }
            leaf null-route {
              type uint48;
              description
                "Number of packets discarded due to null route";
            }
            leaf rpf {
              type uint48;
              description
                "Number of packets discarded due to RPF check failure";
            }
            leaf ddos {
              type uint48;
              description
                "Number of packets discarded due to DDoS protection";
            }
          }
        }
        container no-buffer {
          description
            "Ingress packet discard counters due to buffer unavailability";
          list class {
            key "id";
            min-elements 1;
            description
              "Per-QoS class ingress packet discard counters due to buffer unavailability";
            leaf id {
              type string;
              description
                "QoS class identifier";
            }
            leaf packets {
              type uint64;
              description
                "Number of packets discarded due to buffer unavailability";
            }
            leaf bytes {
              type uint64;
              description
                "Number of bytes discarded due to buffer unavailability";
            }
          }
        }
      }
    }
    container egress {
      description
        "Egress counters";
      container traffic {
        description
          "Egress traffic counters";
        container l2 {
          description
            "Layer 2 egress traffic counters";
          leaf frames {
            type uint64;
            description
              "Number of L2 frames";
          }
          leaf bytes {
            type uint64;
            description
              "Number of L2 bytes";
          }
        }
        container l3 {
          description
            "Layer 3 egress traffic counters";
          container v4 {
            description
              "IPv4 egress traffic counters";
            uses ip;
          }
          container v6 {
            description
              "IPv6 egress traffic counters";
            uses ip;
          }
        }
        container qos {
          description
            "Quality of Service (QoS) egress traffic counters";
          list class {
            key "id";
            min-elements 1;
            description
              "QoS class egress traffic counters";
            leaf id {
              type string;
              description
                "QoS class identifier";
            }
            leaf packets {
              type uint64;
              description
                "Number of packets in the QoS class";
            }
            leaf bytes {
              type uint64;
              description
                "Number of bytes in the QoS class";
            }
          }
        }
      }
      container discards {
        description
          "Egress packet discard counters";
        container l2 {
          description
            "Layer 2 egress packet discard counters";
          leaf frames {
            type uint64;
            description
              "Number of discarded L2 frames";
          }
          leaf bytes {
            type uint64;
            description
              "Number of discarded L2 bytes";
          }
        }
        container l3 {
          description
            "Layer 3 egress packet discard counters";
          container v4 {
            description
              "IPv4 egress packet discard counters";
            uses ip;
          }
          container v6 {
            description
              "IPv6 egress packet discard counters";
            uses ip;
          }
        }
        container errors {
          description
            "Egress packet error counters";
          container l2 {
            description
              "Layer 2 egress packet error counters";
            container tx {
              description
                "Layer 2 egress packet transmit error counters";
              leaf frames {
                type uint48;
                description
                  "Number of errored L2 frames during transmission";
              }
            }
          }
          container l3 {
            description
              "Layer 3 egress packet error counters";
            container tx {
              description
                "Layer 3 egress packet transmit error counters";
              leaf packets {
                type uint48;
                description
                  "Number of errored L3 packets during transmission";
              }
            }
          }
        }
        container policy {
          description
            "Policy-related egress packet discard counters";
          container l3 {
            description
              "Layer 3 policy egress packet discard counters";
            leaf acl {
              type uint48;
              description
                "Number of packets discarded due to L3 egress ACL";
            }
            container policer {
              description
                "Policer egress packet discard counters";
              leaf packets {
                type uint48;
                description
                  "Number of packets discarded by the egress policer";
              }
              leaf bytes {
                type uint48;
                description
                  "Number of bytes discarded by the egress policer";
              }
            }
          }
        }
        container no-buffer {
          description
            "Egress packet discard counters due to buffer unavailability";
          list class {
            key "id";
            min-elements 1;
            description
              "Per-QoS class egress packet discard counters due to buffer unavailability";
            leaf id {
              type string;
              description
                "QoS class identifier";
            }
            leaf packets {
              type uint64;
              description
                "Number of packets discarded due to buffer unavailability";
            }
            leaf bytes {
              type uint64;
              description
                "Number of bytes discarded due to buffer unavailability";
            }
          }
        }
      }
    }
    container control-plane {
      description
        "Control plane packet loss counters";
      container ingress {
        description
          "Control plane ingress packet loss counters";
        container traffic {
          description
            "Control plane ingress traffic counters";
          leaf packets {
            type uint48;
            description
              "Number of control plane packets";
          }
          leaf bytes {
            type uint48;
            description
              "Number of control plane bytes";
          }
        }
        container discards {
          description
            "Control plane ingress packet discard counters";
          leaf packets {
            type uint48;
            description
              "Number of discarded control plane packets";
          }
          leaf bytes {
            type uint48;
            description
              "Number of discarded control plane bytes";
          }
          leaf policy {
            type uint48;
            description
              "Number of control plane packets discarded due to policy";
          }
        }
      }
    }
  }

  /*
   * Main Structure
   */
  container packet-discard-reporting {
    description
      "Container for packet discard reporting data.";

    list interface {
      key "name";
      description
        "List of interfaces for which packet discard reporting data is provided.";

      leaf name {
        type string;
        description
          "Name of the interface.";
      }

      uses interface;
    }
  }
}

~~~~~~~~~~

Security Considerations {#security}
=======================
There are no new security considerations introduced by this document.


IANA Considerations {#iana}
===================
There are no new IANA considerations introduced by this document.


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
{{ex-drop}} depicts an example of where and why packets may be discarded in a typical single ASIC, shared buffered type device, where packets ingress on the left and egress on the right.

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
                                         policy/null_route

Unintended                 error/rx/l2   error/l3/rx   no_buffer     error/l3/tx
  Discards:                              error/local
                                         error/l3/no_route
                                         error/l3/rx/ttl_expired

~~~~~~~~~~
{: #ex-drop title="Example of where packets get dropped"}

Discard Class Descriptions {#class_descriptions}
--------------------------

discards/policy/:  
: These are intended discards, meaning packets dropped by a device due to a configured policy. There are multiple sub-classes.

discards/error/l2/rx/:  
: Frames discarded due to errors in the received L2 frame. There are multiple sub-classes, such as those resulting from failing CRC, invalid header, invalid MAC address, or invalid VLAN.

discards/error/l3/rx/:  
: These are discards which occur due to errors in the received packet, indicating an upstream problem rather than an issue with the device dropping the errored packets. There are multiple sub-classes, including header checksum errors, MTU exceeded, and invalid packet, i.e. due to incorrect version, incorrect header length, or invalid options.

discards/error/l3/rx/ttl_expired:  
: There can be multiple causes for TTL-expired (or Hop limit exceeded) discards: i) trace-route; ii) TTL (Hop limit) set too low by the end-system; iii) routing loops. 

discards/error/l3/no_route/:  
: Discards occur due to a packet not matching any route.

discards/error/local/:  
: A device may discard packets within its switching pipeline due to internal errors, such as parity errors. Any errored discards not explicitly assigned to the above classes are also accounted for here.

discards/no_buffer/:  
: Discards occur due to no available buffer to enqueue the packet. These can be tail-drop discards or due to an active queue management algorithm, such as RED {{RED93}} or CODEL {{RFC8289}}.


Implementation Experience {#experience}
=========================
This appendix captures the authors' experience gained from implementing and applying this information model across multiple vendors' platforms, as guidance for future implementers.

1. The number and granularity of classes described in Section 3 represent a compromise.  It aims to offer sufficient detail to enable appropriate automated actions while avoiding excessive detail, which may hinder quick problem identification.  Additionally, it helps constrain the quantity of data produced per interface to constrain data volume and device CPU impacts.  Although further granularity is possible, the scheme described has generally proven to be sufficient for the task of auto-mitigating unintended packet loss.
2. There are multiple possible ways to define the discard classification tree.  For example, we could have used a multi-rooted tree, rooted in each protocol.  Instead, we opted to define a tree where protocol discards and causal discards are accounted for orthogonally.  This decision reduces the number of combinations of classes and has proven sufficient for determining mitigation actions.
3. NoBuffer discards can be realized differently with different memory architectures. Whether a NoBuffer discard is attributed to ingress or egress can differ accordingly.  For successful auto-mitigation, discards due to egress interface congestion should be reported on egress, while discards due to device-level congestion (e.g. due to exceeding the device forwarding rate) should be reported on ingress.
4. Platforms often account for the number of packets discarded where the TTL has expired (or Hop Limit exceeded), and the device CPU has returned an ICMP Time Exceeded message.  There is typically a policer applied to limit the number of packets sent to the device CPU, however, which implicitly limits the rate of TTL discards that are processed.  One method to account for all packet discards due to TTL expired, even those that are dropped by a policer when being forwarded to the CPU, is to use accounting of all ingress packets received with TTL=1.
5. Where no route discards are implemented with a default null route, separate discard accounting is required for any explicit null routes configured, in order to differentiate between interface/ingress/discards/policy/null_route/packets and interface/ingress/discards/errors/no_route/packets.
6. It is useful to account separately for transit packets discarded by transit ACLs or policers, and packets discarded by ACLs or policers which limit the number of packets to the device control plane.
7. It is not possible to identify a configuration error - e.g., when intended discards are unintended - with device packet loss metrics alone.  For example, to determine if ACL drops are intended or due to a misconfigured ACL some other method is needed, i.e., with configuration validation before deployment or by detecting a significant change in ACL drops after a configuration change compared to before.
8. Where traffic byte counters need to be 64-bit, packet and discard counters that increase at a lower rate may be encoded in fewer bits, e.g., 48-bit.
9. Aggregate counters need to be able to deal with the possibility of discontinuities in the underlying counters.
10. In cases where the reporting device is the source or destination of a tunnel, the ingress protocol for a packet may differ from the egress protocol; if IPv4 is tunneled over IPv6 for example.  Some implementations may attribute egress discards to the ingress protocol.
11. While the classification tree is seven layers deep, a minimal implementation may only implement the top six layers.

