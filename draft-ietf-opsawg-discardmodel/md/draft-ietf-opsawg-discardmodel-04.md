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

The primary function of a network is to transport packets and deliver them according to service level objectives.  Understanding both where and why packet loss occurs within a network is essential for effective network operation.  Device-reported packet loss provides the most direct signal for network operations to identify the customer impact resulting from unintended packet loss.  This document defines an information model for packet loss reporting, which classifies these signals to enable automated network mitigation of unintended packet loss.

--- middle

Introduction        {#introduction}
============

In automating network operations, a network operator needs to be able to detect anomalous packet loss, diagnose or determine the root cause of the loss, and then apply one of a set of possible actions to mitigate customer-impacting packet loss.  Some packet loss is normal or intended in IP/MPLS networks, however. Hence, precise classification of packet loss signals is crucial both to ensure that anomalous packet loss is easily detected and that the right action or sequence of actions is taken to mitigate the impact, as taking the wrong action can make problems worse.

The existing metrics for reporting packet loss, as defined in {{?RFC1213}} (namely ifInDiscards, ifOutDiscards, ifInErrors, and ifOutErrors), do not provide sufficient precision to automatically identify the cause of the loss and mitigate the impact.  From a network operator's perspective, ifInDiscards can represent both intended packet loss (e.g., packets discarded due to policy) and unintended packet loss (e.g., packets dropped in error). Furthermore, these definitions are ambiguous, as vendors can and have implemented them differently.  In some implementations, ifInErrors accounts only for errored packets that are dropped, while in others, it accounts for all errored packets, whether they are dropped or not.  Many implementations support more discard metrics than these; where they do, they have been inconsistently implemented due to the lack of a standardised classification scheme and clear semantics for packet loss reporting. {{?RFC7270}} provides support for reporting discards per flow in IPFIX using forwardingStatus, however, the defined drop reason codes also lack sufficient clarity (e.g., "For us" Reason Code) to support automated root cause analysis and impact mitigation.

Hence, this document defines an information model for packet loss reporting, aiming to address these issues by presenting a packet loss classification scheme that can enable automated mitigation of unintended packet loss.  In line with {{?RFC3444}}, this information model remains independent of specific implementations or transport protocols.

The specific implementations of this information model (i.e., protocols and associated data models) are outside the scope of this document.  The scope of this document is limited to reporting packet loss at Layer 3 and frames discarded at Layer 2, although the information model might be extended in future to cover segments dropped at Layer 4. This document considers only the signals that may trigger automated mitigation plans and not how they are defined or executed.

{{problem}} describes the problem to be solved. Section 4 describes the information model and requirements with a set of examples.  Section 5 provides examples of discard signal-to-cause-to-auto-mitigation action mapping.  Section 6 presents the information model as an abstract data structure in YANG, in accordance with {{!RFC8791}}.  Appendix A provides an example of where packets may be discarded in a device.  Appendix B details the authors' experience from implementing this model.

Terminology {#terminology}
===========

{::boilerplate bcp14-tagged}

A packet discard is considered to be any packet dropped by a device, which may be intentional (i.e. due to a configured policy, e.g. such as an Access Control List (ACL)) or unintentional (i.e. packets dropped in error).

The meanings of the symbols in the YANG tree diagrams are defined in {{?RFC8340}}.

Symbol "&#124;" is used to denote "or".

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

This document uses YANG to represent the information model for three main reasons. First, YANG, along with its data structure extensions {{!RFC8791}}, allows designers to define the model in an abstract way, decoupled from specific implementations. This abstraction ensures consistency and provides flexibility for diverse potential implementations, with the structure and groupings easily adaptable to data models such as those specific to SNMP {{?RFC1157}}, NETCONF {{?RFC6241}}, RESTCONF {{?RFC8040}}, or IPFIX {{?RFC7011}}.  Second, this approach ensures a lossless translation from the information model to a YANG data model, preserving both semantics and structure. Lastly, YANG capitalises on the community’s broad familiarity with its syntax and use, facilitating easier adoption and evolution.


Structure {#structure}
---------

The classification scheme is defined as a tree that follows the structure: component/direction/type/layer/sub-type/sub-sub-type/.../metric, where:

a. Component can be interface&#124;device&#124;control-plane&#124;flow  
b. Direction can be ingress&#124;egress  
c. Type can be traffic&#124;discards, where traffic accounts for packets successfully received or transmitted, and discards accounts for packet drops  
d. Layer can be l2&#124;l3

~~~~~~~~~~
  structure packet-discard-reporting:
    +-- interface* [name]
       +-- name             string
       +-- ingress
       |  +-- traffic
       |  |  +-- l2
       |  |  |  +-- frames?   uint64
       |  |  |  +-- bytes?    uint64
       |  |  +-- l3
       |  |  |  +-- address-family-stat* [address-family]
       |  |  |     +-- address-family    iana-rt-types:address-family
       |  |  |     +-- packets?          uint64
       |  |  |     +-- bytes?            uint64
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
       |     |  +-- address-family-stat* [address-family]
       |     |     +-- address-family    iana-rt-types:address-family
       |     |     +-- packets?          uint64
       |     |     +-- bytes?            uint64
       |     |     +-- unicast
       |     |     |  +-- packets?   uint64
       |     |     |  +-- bytes?     uint64
       |     |     +-- multicast
       |     |        +-- packets?   uint64
       |     |        +-- bytes?     uint64
       |     +-- errors
       |     |  +-- l2
       |     |  |  +-- rx
       |     |  |     +-- frames?          uint32
       |     |  |     +-- crc-error?       uint32
       |     |  |     +-- invalid-mac?     uint32
       |     |  |     +-- invalid-vlan?    uint32
       |     |  |     +-- invalid-frame?   uint32
       |     |  +-- l3
       |     |  |  +-- rx
       |     |  |  |  +-- packets?          uint32
       |     |  |  |  +-- checksum-error?   uint32
       |     |  |  |  +-- mtu-exceeded?     uint32
       |     |  |  |  +-- invalid-packet?   uint32
       |     |  |  |  +-- ttl-expired?      uint32
       |     |  |  +-- no-route?        uint32
       |     |  |  +-- invalid-sid?     uint32
       |     |  |  +-- invalid-label?   uint32
       |     |  +-- hardware
       |     |     +-- packets?        uint32
       |     |     +-- parity-error?   uint32
       |     +-- policy
       |     |  +-- l2
       |     |  |  +-- frames?   uint32
       |     |  |  +-- acl?      uint32
       |     |  +-- l3
       |     |     +-- packets?      uint32
       |     |     +-- acl?          uint32
       |     |     +-- policer
       |     |     |  +-- packets?   uint32
       |     |     |  +-- bytes?     uint32
       |     |     +-- null-route?   uint32
       |     |     +-- rpf?          uint32
       |     |     +-- ddos?         uint32
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
       |  |  |  +-- address-family-stat* [address-family]
       |  |  |     +-- address-family    iana-rt-types:address-family
       |  |  |     +-- packets?          uint64
       |  |  |     +-- bytes?            uint64
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
       |     |  +-- address-family-stat* [address-family]
       |     |     +-- address-family    iana-rt-types:address-family
       |     |     +-- packets?          uint64
       |     |     +-- bytes?            uint64
       |     |     +-- unicast
       |     |     |  +-- packets?   uint64
       |     |     |  +-- bytes?     uint64
       |     |     +-- multicast
       |     |        +-- packets?   uint64
       |     |        +-- bytes?     uint64
       |     +-- errors
       |     |  +-- l2
       |     |  |  +-- tx
       |     |  |     +-- frames?   uint32
       |     |  +-- l3
       |     |     +-- tx
       |     |        +-- packets?   uint32
       |     +-- policy
       |     |  +-- l3
       |     |     +-- acl?       uint32
       |     |     +-- policer
       |     |        +-- packets?   uint32
       |     |        +-- bytes?     uint32
       |     +-- no-buffer
       |        +-- class* [id]
       |           +-- id         string
       |           +-- packets?   uint64
       |           +-- bytes?     uint64
       +-- control-plane
          +-- ingress
             +-- traffic
             |  +-- packets?   uint32
             |  +-- bytes?     uint32
             +-- discards
                +-- packets?   uint32
                +-- bytes?     uint32
                +-- policy
                   +-- packets?   uint32

~~~~~~~~~~

For additional context, Appendix A provides an example of where packets may be discarded in a device.


Requirements {#requirements}
------------
Requirements 1-10 relate to packets forwarded by the device, while requirement 11 relates to packets destined for or originating from the device:

1. All instances of frame or packet receipt, transmission, and discards MUST be reported.
2. All instances of frame or packet receipt, transmission, and discards SHOULD be attributed to the physical or logical interface of the device where they occur.
3. An individual frame MUST only be accounted for by either the Layer 2 traffic class or the Layer 2 discard classes within a single direction, i.e., ingress or egress.
4. An individual packet MUST only be accounted for by either the Layer 3 traffic class or the Layer 3 discard classes within a single direction, i.e., ingress or egress.
5. A frame accounted for at Layer 2 SHOULD NOT be accounted for at Layer 3 and vice versa.  An implementation MUST indicate which layers a discard is counted against.
6. The aggregate Layer 2 and Layer 3 traffic and discard classes SHOULD account for all underlying packets received, transmitted, and discarded across all other classes.
7. The aggregate Quality of Service (QoS) traffic and no buffer discard classes MUST account for all underlying packets received, transmitted, and discarded across all other classes.
8. In addition to the Layer 2 and Layer 3 aggregate classes, an individual discarded packet MUST only account against a single error, policy, or no_buffer discard subclass.
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
|                                           | or link error       |            |          |             | device out-of-service |
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

The 'Baseline' in the 'Discard Rate' column is both discard class and network dependent.

YANG Module {#module}
===========

The "ietf-packet-discard-reporting" uses the "sx" structure defined in {{!RFC8791}}.

~~~~~~~~~~
  <CODE BEGINS>
module ietf-packet-discard-reporting {
  yang-version 1.1;
  namespace
    "urn:ietf:params:xml:ns:yang:ietf-packet-discard-reporting";
  prefix plr;

  import iana-routing-types {
    prefix "iana-rt-types";
    reference
      "RFC 8294: Common YANG Data Types for the Routing Area";
  }

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
    "This module defines an information model for packet discard
     reporting.

     Copyright (c) 2024 IETF Trust and the persons identified as
     authors of the code.  All rights reserved.

     Redistribution and use in source and binary forms, with or
     without modification, is permitted pursuant to, and subject
     to the license terms contained in, the Revised BSD License
     set forth in Section 4.c of the IETF Trust's Legal Provisions
     Relating to IETF Documents
     (https://trustee.ietf.org/license-info).

     This version of this YANG module is part of RFC XXXX; see the
     RFC itself for full legal notices.";

  revision 2024-06-04 {
    description
      "Initial revision.";
    reference
      "RFC XXXX: An Information Model for Packet Discard Reporting";
  }

  /*
   * Groupings
   */

  grouping basic-packets-64 {
    description
      "Basic grouping with 64-bit packets";
    leaf packets {
      type uint64;
      description
        "Number of L3 packets";
    }
  }

  grouping basic-packets-bytes-64 {
    description
      "Basic grouping with 64-bit packets and bytes";
    uses basic-packets-64;
    leaf bytes {
      type uint64;
      description
        "Number of L3 bytes";
    }
  }

  grouping basic-frames-64 {
    description
      "Basic grouping with 64-bit frames";
    leaf frames {
      type uint64;
      description
        "Number of L2 frames";
    }
  }

  grouping basic-frames-bytes-64 {
    description
      "Basic grouping with 64-bit packets and bytes";
    uses basic-frames-64;
    leaf bytes {
      type uint64;
      description
        "Number of L2 bytes";
    }
  }

  grouping basic-packets-32 {
    description
      "Basic grouping with 32-bit packets";
    leaf packets {
      type uint32;
      description
        "Number of L3 packets";
    }
  }

  grouping basic-packets-bytes-32 {
    description
      "Basic grouping with 32-bit packets and bytes";
    uses basic-packets-32;
    leaf bytes {
      type uint32;
      description
        "Number of L3 bytes";
    }
  }

  grouping basic-frames-32 {
    description
      "Basic grouping with 32-bit frames";
    leaf frames {
      type uint32;
      description
        "Number of L2 frames";
    }
  }

  grouping basic-frames-bytes-32 {
    description
      "Basic grouping with 32-bit packets and bytes";
    uses basic-frames-32;
    leaf bytes {
      type uint32;
      description
        "Number of L2 bytes";
    }
  }

  grouping l2-traffic {
    description
      "Layer 2 traffic counters";
    uses basic-frames-bytes-64;
  }

  grouping ip {
    description
      "Traffic counters";
    list address-family-stat {
      key "address-family";
      description
        "Reports per address family traffic counters.";
      leaf address-family {
        type iana-rt-types:address-family;
        description
          "Specifies an address family.";
      }
      uses basic-packets-bytes-64;
      container unicast {
        description
          "Reports unicast traffic counters.";
        uses basic-packets-bytes-64;
      }
      container multicast {
        description
          "Reports multicast traffic counters.";
        uses basic-packets-bytes-64;
      }
    }
  }

  grouping l3-traffic {
    description
      "A grouping for Layer 3 traffic counters.";
      uses ip;
  }

  grouping qos {
    description
      "A grouping fro Quality of Service (QoS) traffic
       counters.";
    list class {
      key "id";
      min-elements 1;
      description
        "Indicates QoS class traffic counters.";
      leaf id {
        type string;
        description
          "Specifies a QoS class identifier.";
      }
      uses basic-packets-bytes-64;
    }
  }

  grouping traffic {
    description
      "A grouping for overall traffic counters.";
    container l2 {
      description
        "Specifies Layer 2 traffic counters.";
      uses l2-traffic;
    }
    container l3 {
      description
        "Specifies Layer 3 traffic counters.";
      uses l3-traffic;
    }
    container qos {
      description
        "Specifies QoS traffic counters.";
      uses qos;
    }
  }

  grouping control-plane {
    description
      "A grouping for control plane packet counters.";
    container ingress {
      description
        "Specifies control plane ingress counters.";
      container traffic {
        description
          "Reports control plane ingress traffic counters.";
        uses basic-packets-bytes-32;
      }
      container discards {
        description
          "Reports control plane ingress packet discard counters.";
        uses basic-packets-bytes-32;
        container policy {
          description
            "Indicates the number of control plane packets discarded
             due to policy.";
          uses basic-packets-32;
        }
      }
    }
  }

  grouping errors-l2-rx {
    description
      "A grouping for Layer 2 ingress frame errors.";
    container rx {
      description
        "Specifies Layer 2 ingress frame error counters.";
      leaf frames {
        type uint32;
        description
          "Indicates the number of errored Layer 2 frames.";
      }
      leaf crc-error {
        type uint32;
        description
          "Reports the number of frames received with CRC error.";
      }
      leaf invalid-mac {
        type uint32;
        description
          "Reports the number of frames received with an invalid
           MAC address.";
      }
      leaf invalid-vlan {
        type uint32;
        description
          "Reports the number of frames received with an invalid
           VLAN tag.";
      }
      leaf invalid-frame {
        type uint32;
        description
          "Reports the number of invalid frames received.";
      }
    }
  }

  grouping errors-l3-rx {
    description
      "A grouping for Layer 3 ingress packet error counters.";
    container rx {
      description
        "Specifies Layer 3 ingress packet receive error counters.";
      leaf packets {
        type uint32;
        description
          "Indicates the number of errored Layer 3 packets.";
      }
      leaf checksum-error {
        type uint32;
        description
          "Indicates the number of packets received with a checksum
           error.";
      }
      leaf mtu-exceeded {
        type uint32;
        description
          "Reports the number of packets received with an exceeding
           MTU.";
      }
      leaf invalid-packet {
        type uint32;
        description
          "Reports the number of invalid packets received.";
      }
      leaf ttl-expired {
        type uint32;
        description
          "Reports the number of packets received with expired
           TTL.";
      }
    }
    leaf no-route {
      type uint32;
      description
        "Specifies the number of packets with no route.";
    }
    leaf invalid-sid {
      type uint32;
      description
        "Specifies the number of packets with an invalid SID.";
    }
    leaf invalid-label {
      type uint32;
      description
        "Specifies the number of packets with an invalid label.";
    }
  }

  grouping errors-l3-hw {
    description
      "A grouping for hardware error counters.";
    leaf packets {
      type uint32;
      description
        "Reports the number of local errored packets.";
    }
    leaf parity-error {
      type uint32;
      description
        "Reports the number of packets with parity error.";
    }
  }

  grouping errors-rx {
    description
      "A grouping for ingress error counters.";
    container l2 {
      description
        "Reports Layer 2 received frame error counters.";
      uses errors-l2-rx;
    }
    container l3 {
      description
        "Reports Layer 3 received packet error counters.";
      uses errors-l3-rx;
    }
    container hardware {
      description
        "Reports hardware error counters.";
      uses errors-l3-hw;
    }
  }

  grouping errors-l2-tx {
    description
      "A grouping for Layer 2 transmit error counters.";
    container tx {
      description
        "Reports Layer 2 transmit frame error counters.";
      leaf frames {
        type uint32;
        description
          "Reports the number of errored Layer 2 frames when
           transmitting.";
      }
    }
  }

  grouping errors-l3-tx {
    description
      "A grouping for Layer 3 transmit error counters.";
    container tx {
      description
        "Reports Layer 3 transmit packet error counters.";
      leaf packets {
        type uint32;
        description
          "Reports the number of errored Layer 3 packets when
           transmitting.";
      }
    }
  }

  grouping errors-tx {
    description
      "A grouping for egress error counters.";
    container l2 {
      description
        "Reports Layer 2 transmit frame error counters.";
      uses errors-l2-tx;
    }
    container l3 {
      description
        "Reports Layer 3 transmit packet error counters.";
      uses errors-l3-tx;
    }
  }

  grouping policy-l2-rx {
    description
      "A grouping for Layer 2 policy ingress packet discard
       counters.";
    leaf frames {
      type uint32;
      description
        "Specifies the number of Layer 2 frames discarded due
         to policy.";
    }
    leaf acl {
      type uint32;
      description
        "Specifies the number of frames discarded due to
         Layer 2 ACL.";
    }
  }

  grouping policy-l3-rx {
    description
      "A grouping for Layer 3 policy ingress packet discard
       counters.";
    leaf packets {
      type uint32;
      description
        "Specifies the number of Layer 3 packets discarded due
         to policy.";
    }
    leaf acl {
      type uint32;
      description
        "Specifies the number of packets discarded due to
         Layer 3 ACL.";
    }
    container policer {
      description
        "Indicates policer ingress packet discard counters.";
      uses basic-packets-bytes-32;
    }
    leaf null-route {
      type uint32;
      description
        "Indicates the number of packets discarded due to
         null route.";
    }
    leaf rpf {
      type uint32;
      description
        "Indicates the number of packets discarded due to Reverse
         Path Forwarding (RPF) check failure.";
    }
    leaf ddos {
      type uint32;
      description
        "Indicates the number of packets discarded due to DDoS
         protection.";
    }
  }

  grouping policy-rx {
    description
      "A grouping for policy-related ingress packet
       discard counters.";
    container l2 {
      description
        "Reports Layer 2 policy ingress packet discard counters.";
      uses policy-l2-rx;
    }
    container l3 {
      description
        "Reports Layer 3 policy ingress packet discard counters.";
      uses policy-l3-rx;
    }
  }

  grouping policy-l3-tx {
    description
      "A grouping for Layer 3 policy egress packet discard counters.";
    leaf acl {
      type uint32;
      description
        "Reports the number of packets discarded due to Layer 3
         egress ACL.";
    }
    container policer {
      description
        "Reports policer egress packet discard counters.";
      uses basic-packets-bytes-32;
    }
  }

  grouping policy-tx {
    description
      "Policy-related egress packet discard counters";
    container l3 {
      description
        "Specifies Layer 3 policy egress packet discard counters.";
      uses policy-l3-tx;
    }
  }

  grouping interface {
    description
      "A grouping for interface-level packet loss counters.";
    container ingress {
      description
        "Ingress counters";
      container traffic {
        description
          "Specifies ingress traffic counters.";
        uses traffic;
      }
      container discards {
        description
          "Specifies ingress packet discard counters.";
        container l2 {
          description
            "Specifies Layer 2 ingress discards traffic counters.";
          uses l2-traffic;
        }
        container l3 {
          description
            "Indicates Layer 3 ingress discards traffic counters.";
          uses l3-traffic;
        }
        container errors {
          description
            "Reports ingress packet error counters.";
          uses errors-rx;
        }
        container policy {
          description
            "Indicates policy-related ingress packet discard counters.";
          uses policy-rx;
        }
        container no-buffer {
          description
            "Reports ingress packet discard counters due to buffer
             unavailability.";
          uses qos;
        }
      }
    }
    container egress {
      description
        "A grouoing for egress counters.";
      container traffic {
        description
          "Reports egress traffic counters.";
        uses traffic;
      }
      container discards {
        description
          "Repirts egress packet discard counters.";
        container l2 {
          description
            "Specifies Layer 2 egress packet discard counters.";
          uses l2-traffic;
        }
        container l3 {
          description
            "Specifies Layer 3 egress packet discard counters.";
          uses l3-traffic;
        }
        container errors {
          description
            "Indicates Egress packet error counters.";
          uses errors-tx;
        }
        container policy {
          description
            "Indicates Policy-related egress packet discard counters.";
          uses policy-tx;
        }
        container no-buffer {
          description
            "Indicates egress packet discard counters due to buffer
             unavailability.";
          uses qos;
        }
      }
    }
    container control-plane {
      description
        "Reports control plane packet counters.";
      uses control-plane;
    }
  }

  /*
   * Main Structure
   */

  sx:structure packet-discard-reporting {
    description
      "Specifies the abstract structure of packet discard reporting data.";
    list interface {
      key "name";
      description
        "Indicates a list of interfaces for which packet discard reporting
         data is provided.";
      leaf name {
        type string;
        description
          "Indicates the name of the interface.";
      }
      uses interface;
    }
  }
}
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

Where do packets get dropped? 
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

1. The number and granularity of classes described in Section 3 represent a compromise.  It aims to offer sufficient detail to enable appropriate automated actions while avoiding excessive detail, which may hinder quick problem identification.  Additionally, it helps limit the quantity of data produced per interface, thus constraining the data volume and device CPU impacts.  Although further granularity is possible, the scheme described has generally proven to be sufficient for the task of auto-mitigating unintended packet loss.
2. There are many possible ways to define the discard classification tree.  For example, we could have used a multi-rooted tree, rooted in each protocol.  Instead, we opted to define a tree where protocol discards and causal discards are accounted for orthogonally.  This decision reduces the number of combinations of classes and has proven sufficient for determining mitigation actions.
3. NoBuffer discards can be realized differently with different memory architectures. Whether a NoBuffer discard is attributed to ingress or egress can differ accordingly.  For successful auto-mitigation, discards due to egress interface congestion should be reported on egress, while discards due to device-level congestion (e.g. due to exceeding the device forwarding rate) should be reported on ingress.
4. Platforms often account for the number of packets discarded where the TTL has expired (or Hop Limit exceeded), and the device CPU has returned an ICMP Time Exceeded message.  There is typically a policer applied to limit the number of packets sent to the device CPU, however, which implicitly limits the rate of TTL discards that are processed.  One method to account for all packet discards due to TTL expired, even those that are dropped by a policer when being forwarded to the CPU, is to use accounting of all ingress packets received with TTL=1.
5. Where no route discards are implemented with a default null route, separate discard accounting is required for any explicit null routes configured, in order to differentiate between interface/ingress/discards/policy/null_route/packets and interface/ingress/discards/errors/no_route/packets.
6. It is useful to account separately for transit packets discarded by ACLs or policers, and packets discarded by ACLs or policers which limit the number of packets to the device control plane.
7. It is not possible to identify a configuration error - e.g., when intended discards are unintended - with device packet loss metrics alone.  For example, additional context is needed to determine if ACL discards are intended or due to a misconfigured ACL, i.e., with configuration validation before deployment or by detecting a significant change in ACL discards after a configuration change compared to before.
8. Where traffic byte counters need to be 64-bit, packet and discard counters that increase at a lower rate may be encoded in fewer bits, e.g., 32-bit.
9. Aggregate counters need to be able to deal with the possibility of discontinuities in the underlying counters.
10. In cases where the reporting device is the source or destination of a tunnel, the ingress protocol for a packet may differ from the egress protocol; if IPv4 is tunnelled over IPv6 for example.  Some implementations may attribute egress discards to the ingress protocol.
11. While the classification tree is seven layers deep, a minimal implementation may only implement the top six layers.

