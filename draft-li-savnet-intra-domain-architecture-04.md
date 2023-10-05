---
title: Intra-domain Source Address Validation (SAVNET) Architecture
abbrev: Intra-domain SAVNET Architecture
docname: draft-li-savnet-intra-domain-architecture-04
obsoletes:
updates:
date:
category: info
submissionType: IETF

ipr: trust200902
area: Routing
workgroup: SAVNET
keyword: SAV

author:
 -
  ins: D. Li
  name: Dan Li
  organization: Tsinghua University
  email: tolidan@tsinghua.edu.cn
  city: Beijing
  country: China
 -
  ins: J. Wu
  name: Jianping Wu
  organization: Tsinghua University
  email: jianping@cernet.edu.cn
  city: Beijing
  country: China
 -
  ins: M. Huang
  name: Mingqing Huang
  organization: Huawei
  email: huangmingqing@huawei.com
  city: Beijing
  country: China
 -
  ins: L. Chen
  name: Li Chen
  organization: Zhongguancun Laboratory
  email: lichen@zgclab.edu.cn
  city: Beijing
  country: China
 -
  ins: N. Geng
  name: Nan Geng
  organization: Huawei
  email: gengnan@huawei.com
  city: Beijing
  country: China
 -
  ins: L. Qin
  name: Lancheng Qin
  organization: Tsinghua University
  email: qlc19@mails.tsinghua.edu.cn
  city: Beijing
  country: China
 -
  ins: F. Gao
  name: Fang Gao
  organization: Zhongguancun Laboratory
  email: gaofang@zgclab.edu.cn
  city: Beijing
  country: China


normative:
  I-D.ietf-savnet-intra-domain-problem-statement:
  RFC2827:
  RFC3704:
  I-D.huang-savnet-sav-table:
informative:
  RFC5210:
  RFC7039:
  RFC7513:
  IPSG:
    title: Configuring DHCP Features and IP Source Guard
    author: 
    org: Cisco
    date: 2016-01
    target: https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst2960/software/release/12-2_53_se/configuration/guide/2960scg/swdhcp82.html
  cable-verify:
    title: Cable Source-Verify and IP Address Security
    author: 
    org: Cisco
    date: 2021-01
    target: https://www.cisco.com/c/en/us/support/docs/broadband-cable/cable-security/20691-source-verify.html

...

--- abstract

This document proposes the intra-domain SAVNET architecture. It can achieve more accurate Source address validation (SAV) than existing SAV mechanisms that only use the router's local routing information to generate SAV rules. In this architecure, routers in the intra-domain network can generate SAV rules based on not only local routing information but also SAV-specific information. 

This document primarily introduces how to communicate SAV-specific information between routers, and how to generate SAV rules based on SAV-specific information and local routing information. The future intra-domain SAV mechanisms can be developed under this architecture. The detailed designs of new intra-domain SAV mechanisms are not the focus of this document.

--- middle

# Introduction {#sec-intro}
Source address validation (SAV) is important for mitigating source address spoofing and contributing to the Internet security. Source Address Validation Architecture (SAVA) [RFC5210] divides SAV into three checking levels, i.e., access-network SAV, intra-domain SAV, and inter-domain SAV. When an access network does not deploy SAV (such as SAVI {{?RFC7039}}{{?RFC7513}}, Cable Source Verify {{cable-verify}}, and IP Source Guard {{IPSG}}), intra-domain SAV helps block spoofed packets from the access network as close to the source as possible. The concept of intra-domain SAV has been defined in {{I-D.ietf-savnet-intra-domain-problem-statement}}. 

The main task of SAV mechanisms is to generate SAV rules for checking the validity of source addresses of data packets. The information of source addresses/prefixes and their legitimate incoming directions makes up the SAV rules. How to efficiently and accurately learn the information is the core challenge for SAV mechanisms. 
Although many intra-domain SAV mechanisms (such as ACL-based filtering [RFC2827], strict uRPF [RFC3704], and loose uRPF [RFC3704]) have been proposed, they all have the problems of inaccurate validation or high operational overhead in some scenarios because they generate SAV rules using either the router's local routing information or manual configurations {{I-D.ietf-savnet-intra-domain-problem-statement}}. To address these problems, five requirements of future intra-domain mechanisms are proposed in {{I-D.ietf-savnet-intra-domain-problem-statement}}, i.e., automatic update, accurate validation, working in incremental/partial deployment, fast convergence, and necessary security guarantee. 

This document introduces the intra-domain SAVNET architecture to meet the above requirements. Consider that it is difficult to generate accurate SAV rules solely using the router's local routing information in asymmetric routing scenarios, and manual configurations/updates require high operational overhead. The intra-domain SAVNET architecture defines the SAV-specific information that helps routers generate SAV rules accurately and automatically. Routers in the intra-domain network can automatically communicate SAV-specific information through the SAV-specific information communication mechanism. In the incremental/partial deployment scenario where only part of intra-domain routers support the intra-domain SAVNET architecure, SAV-specific information of some routers cannot be available. In this scenario, routers can use both local routing information and received SAV-specific information to aviod improper block problems and reduce improper permit problems. 

This document also provides the convergence, security, manageability, and privacy considerations for future inta-domain SAV mechanisms. The detailed designs of new intra-domain SAV mechanisms are not included in this document.

## Requirements Language

{::boilerplate bcp14-tagged}


# Terminology
SAV Rule: The rule that indicates the validity of a specific source IP address or source IP prefix.

SAV Table: The table or data structure that implements the SAV rules and is used for source address validation in the data plane. 

Local Routing Information: The information stored in the router's local RIB or FIB that can be used to infer SAV rules in addition to the routing purpose. 

SAV-specific Information: The information specialized for SAV rule generation that is communicated between routers. For example, it can notify routers the source prefixes of the specific subnet or incoming directions of the specific source prefix. 

SAV-related Information: Any information that can be used for generating SAV rules. Generally, SAV-related information includes local routing information and SAV-specific information. 

SAV-specific Information Communication Mechanism: The mechanism for communicating SAV-specific information between routers. The mechanism can be a new protocol or an extension to an existing protocol. 

SAV Information Base: A table or data structure for storing SAV-related information. 

Edge Router: An intra-domain router that is connected to intra-domain subnets.

Border Router: An intra-domain router that is connected to other ASes. A router can be an edge router or a border router, if it is connected to both subnets and other ASes.

False Positive: The validation results that the packets with legitimate source IP addresses are considered invalid due to inaccurate SAV rules. False positive may induce improper block problems if routers block the "invalid" packets. 

False Negative: The validation results that the packets with spoofed source IP addresses are considered valid due to inaccurate SAV rules. False negative may induce improper permit problems if routers accept the "valid" packets. 


# Intra-domain SAVNET Architecture {#sec-arch}

## Overview {#sec-arch-overview}

The proposed architecture is shown in {{fig-arch}}. In the architecture, there is a communication channel connecting two entities, i.e., Source Entity and Validation Entity:

- Source Entity sends SAV-specific information to Validation Entity. It can be an edge router in the intra-domain network.

- Validation Entity receives the SAV-specific information, processes SAV-related information, generates SAV rules, and conducts SAV. It can be either an edge router or a border router in the intra-domain network.  

~~~
+-------------------+               +---------------------------------------+
|   Source Entity   | Communication |           Validation Entity           |
| +---------------+ | Channel       | +---------------+   +---------------+ |
| |  Source       +------------------->  Validation   |   |    RIB/FIB    | |
| |  Speaker      | |(SAV-specific  | |  Receiver     |   |               | |
| +---------------+ | Information ) | +---------------+   +---------------+ |
|                   |               |         |SAV-specific   |Local Routing|
|                   |               |         |Information    |Information  |
|                   |               |      +--v---------------v--------+    |
|                   |               |      |            SAV            |    |
|                   |               |      |           Agent           |    |
|                   |               |      +---------------------------+    |
+-------------------+               +---------------------------------------+
~~~
{: #fig-arch title="The intra-domain SAVNET architecture"}

An edge router can act as a Source Entity to send its SAV-specific information to other routers, and it can also act as a Validation Entity to receive SAV-specific information from other routers at the same time.

Source Speaker in Source Entity is responsible to send SAV-specific information to Validation Receiver in Validation Entity through the communication channel. To this end, a new SAV-specific information communication mechanism should be developed to deliver the SAV-specific information from Source Entity to Validation Entity. SAV Agent in Validation Entity consolidates and processes SAV-related information from Validation Receiver and local RIB/FIB, and finally generates SAV rules. 

In the following, we introduce the SAV-related information, SAV-specific information communication mechanism, and SAV agent, respectively. Finally, we use two use cases to illustrate that intra-domain SAVNET architecture can improve the accuracy of SAV upon existing intra-domain SAV mechanisms.

## SAV-related Information
SAV-related information represents any information that is useful for inferring or generating SAV rules. There are two kinds of SAV-related information in the intra-domain network: local routing information and SAV-specific information. 

Local routing information is used for forwarding rule computation, which is stored in RIB/FIB. Although it is not specialized for SAV, it can also be used to infer SAV rules in existing uRPF-based SAV mechanisms, such as strict uRPF and loose uRPF.

SAV-specific information is specialized for SAV. For example, it can help the Validation Entity identify the source prefixes of intra-domain subnets or the incoming directions of source prefixes:

- Source prefixes of intra-domain subnets. The Source Entity that is directly connected to subnets can inform the Validation Entity of its locally known source prefixes of its subnets.

- Incoming directions of source prefixes. 
The Source Entity can inform the Validation Entity of the direction it chooses to forward traffic to the Validation Entity. For the Validation Entity, this direction is the legitimate incoming direction of source prefixes of the Source Entity.

By learning the SAV-specific information of the Source Entity, the Validation Entity can generate more accurate SAV rules than solely using its local routing information. In this way, false positive problems can be avoided and false negative problems can be reduced.

## SAV-specific Information Communication Mechanism
The SAV-specific communication mechanism is for propagating SAV-specific information from Source Entity to Validation Entity. Since there is no off-the-shelf mechanism to achieve this function, a new SAV-specific communication mechanism is needed. It can be a new protocol or an extension to an exsiting protocol. This document does not present the details of the protocol design or protocol extensions. In the following, we describe the necessary features of SAV-specific communication mechanism.

The SAV-specific communication mechanism SHOULD define the data structure or format of communnicated SAV-specific information, and the operations of communication (such as communication channel establishment and communication channel termination). In addition, the mechanism SHOULD require Source Entity to inform Validation Entity of the updates of SAV-specific information in a timely manner, so that Validation Entity can quickly update SAV rules. 

The session of the SAV-specific communication mechanism SHOULD meet the following requirements: 

- The session can be a long-time session or a temporary one, but it SHOULD provide sufficient assurance of transmission reliability and timeliness, so that Validation Entity can update its SAV rules in time. 

- Authentication can be conducted before session establishment. Authentication is optional but the ability of authentication SHOULD be available. 

## SAV Agent {#sec-arch-agent}
{{fig-sav-agent}} shows the workflow of SAV Agent. SAV Information Manager in SAV Agent consolidates SAV-specific informaion from Validation Receiver and local routing inforation from RIB/FIB, and stores these SAV-related information in SAV Information Base. The stored information will be disseminated to SAV Rule Generator. After that, SAV rules (e.g., tuples like <prefix, interface set, validity state>) will be generated and stored in SAV Table {{I-D.huang-savnet-sav-table}}. SAV Information Base also provides the support of diagnosis. Operators can look up the information in the base for monitoring or troubleshooting. 

~~~
SAV-specific informaion from Validation Receiver,
and local routing inforation from RIB/FIB
                    |
    +---------------|---------------+
    | SAV Agent     |               |
    | +-------------\/------------+ |
    | | SAV Information Manager   | |
    | | +-----------------------+ | |
    | | | SAV Information Base  | | |
    | | +-----------------------+ | |
    | +-------------+-------------+ |
    |               |               |
    |   SAV-related | information   |
    |               |               |
    | +-------------\/------------+ |
    | | SAV Rule Generator        | |
    | | +-----------------------+ | |
    | | | SAV Table             | | |
    | | +-----------------------+ | |
    | +---------------------------+ |
    +-------------------------------+
~~~
{: #fig-sav-agent title="The workflow of SAV agent"}

The SAV Information Base stores local routing information as well as SAV-specific information from multiple Source Entities. When SAV Rule Generator generates rules based on the stored information, rule conflicts may appear. For example, for a given source prefix, the information from different information sources may indicate different legitimate incoming interfaces. 

To solve the problem of rule conflicts, priority can be set for information from different information sources. In general, since SAV-specific information is specialized for SAV and helps generate more accurate SAV rules than local routing information, SAV-specific information should have a higher priority than local routing information. When rule conflicts exist, the rule generated based on the information with a higher priority will override that generated based on the information with a lower priority. If two conflicting rules are generated based on the information with the same priority, they can be merged to one rule (i.e., taking a union set). When the settings of priority change, the affected information MUST be reprocessed for updating SAV rules. 

For example, consider that one Validation Entity receives the SAV-specific information about prefix P1 from one Source Entity through the communication channel. Based on the SAV-specific information, it generates the SAV rule of <P1, {intf1}, valid>. If the SAV rule generated based on local routing information is <P1, {intf2}, valid>, only the SAV rule of <P1, {intf1}, valid> will be finally stored in SAV Table because SAV-specific information has a higher priority than local routing information. If the SAV rule generated based on SAV-specific information from another Source Entity is <P1, {intf2}, valid>, <P1, {intf1, intf2}, valid> is the final SAV rule because they have the same priority.

In the incremental/partial deployment scenario, some edge/border routers in the intra-domain network may not support the SAV-specific information communication mechanism. Therefore, SAV-specific information of some routers is not available to the router acting as the Validation Entity. Following the working principle of SAV agent, in this case, local routing information can be used as a supplement of SAV-specific information for SAV rule generation. For example, the Validation Entity can block packets with non-global or non-routable source addresses by using its local routing information.


## Use Cases of Intra-domain SAVNET Architecture
Two use cases are given below to illustrate that intra-domain SAVNET architecture can work better than existing intra-domain SAV mechanisms. 

### Use Case 1: Validating Outbound Packets from a Multi-homed Subnet at Edge Routers

{{fig-use-case1}} shows an asymmetric routing in the multi-homed subnet scenario. This scenario has been proposed in {{I-D.ietf-savnet-intra-domain-problem-statement}}. Subnet 1 has prefix 10.0.0.0/15 and is attached to two edge routers, i.e., Router 1 and Router 2. Due to the inbound load balance strategy of Subnet 1, Router 1 only learns the route to sub prefix 10.1.0.0/16 from Subnet 1, while Router 2 only learns the route to the other sub prefix 10.0.0.0/16 from Subnet 1. After that, Router 1 or Router 2 learns the route to the other sub prefix through the intra-domain routing protocol. The FIBs of Router 1 and Router 2 are shown in the figure. Assume Subnet 1 may send outbound packets with source addresses in sub prefix 10.0.0.0/16 to Router 1 for outbound load balance. 

~~~
+-------------------------------------------------------------+
|                                                      AS     |
|                        +----------+                         |
|                        | Router 3 |                         |
| FIB on Router 1        +----------+  FIB on Router 2        |
| Dest         Next_hop   /\      \    Dest         Next_hop  |
| 10.1.0.0/16  Subnet 1   /        \   10.0.0.0/16  Subnet 1  |
| 10.0.0.0/16  Router 3  /         \/  10.1.0.0/16  Router 3  |
|                +----------+     +----------+                |
|                | Router 1 |     | Router 2 |                |
|                +-----+#+--+     +-+#+------+                |
|                        /\         /                         |
|   Outbound traffic with \        / Inbound traffic with     |
|   source IP addresses    \      /  destination IP addresses |
|   of 10.0.0.0/16          \    \/  of 10.0.0.0/16           |
|                           Subnet 1                          |
|                        (10.0.0.0/15 )                       |
|                                                             |
+-------------------------------------------------------------+
~~~
{: #fig-use-case1 title="A use case of outbound SAV"}

In this case, as described in {{I-D.ietf-savnet-intra-domain-problem-statement}}, strict uRPF at Router 1 will improperly block legitimate packets with source addresses in prefix 10.0.0.0/16 from Subnet 1 at interface '#', because it only accepts packets with source addresses in prefix 10.1.0.0/16 from Router 1's interface '#'.

If the proposed architecture is implemented in the network, Router 2 can inform Router 1 that prefix 10.0.0.0/16 also belongs to Subnet 1 by sending SAV-specific information to Router 1. Then, Router 1 learns that Subnet 1 owns both prefix 10.1.0.0/16 and prefix 10.0.0.0/16. Therefore, Router 1 will not block packets with source addresses in prefix 10.0.0.0/16 at interface '#', so improper block can be avoided. 

### Use Case 2: Validating Inbound Packets from Other ASes at Border Routers

{{fig-use-case2}} shows the inbound SAV scenario which is proposed in {{I-D.ietf-savnet-intra-domain-problem-statement}}. Router 3 and Router 4 perform intra-domain SAV at interface '#' to block inbound packets with spoofed internal source addresses.

~~~
                + Packets with              + Packets with
                | spoofed P1/P2             | spoofed P1/P2
+--------------|---------------------------|----------+
|  AS          \/                          \/         |
|          +--+#+-----+               +---+#+----+    |
|          | Router 3 +-------------->| Router 4 |    |
|          +----------+               +----------+    |
|             /     \                      |          |
|            /       \                     |          |
|          \/         \/                   \/         |
|  +----------+     +----------+      +----------+    |
|  | Router 1 |     | Router 2 |      | Router 5 |    |
|  +----------+     +----------+      +----------+    |
|        \              /                   |         |
|         \            /                    |         |
|          \          /                     \/        |
|           Subnet 1(P1)               Subnet 2(P2)   |
+-----------------------------------------------------+
~~~
{: #fig-use-case2 title="A use case of inbound SAV"}

As described in {{I-D.ietf-savnet-intra-domain-problem-statement}}, if Router 3 and Router 4 deploy ACL-based ingress filtering, the operational overhead of manually maintaining and updating ACL rules will be extremely high when there are multiple inbound validation points. 

If the proposed architecture is implemented in the network, Router 1, Router 2, and Router 5 will inform Router 3 and Router 4 of the source prefixes of Subnet 1 and Subnet 2 by sending SAV-specific information. Then, Router 3 and Router 4 can automatically generate and update accurate SAV rules at interface '#'. 


# Convergence Considerations
When the SAV-specific information or local routing information changes, the SAV agent MUST be able to detect the change in time and update SAV rules with the new SAV-related information. Since SAV-specific information is originated from the Source Entity, it requires the Source Entity MUST send the updated SAV-specific information to the Validation Entity in a timely manner. For example, in {{fig-use-case2}}, if Subnet 2 has a new source prefix P3, Router 5 MUST inform Router 3 and Router 4 of the new source prefix of Subnet 2 immediately. 

Even though, there will still be delays of message delivery (sometimes re-transmission delay due to packet loss) and information processing. Therefore, during the convergence process, the SAV rules for checking packets are possibly inaccurate, which may result in false positive or false negative. Existing uRPF-based SAV mechanisms that solely use local routing information are also faced with similar convergence problems. Inaccurate validation may appear during the convergence of routing, which is inevitable in practice.

To mitigate the impact of convergence problems and aviod improper block, future intra-domain SAV mechanisms MUST carefully consider the factors that may affect the convergence performance.


# Incremental/Partial Deployment Considerations
Although an intra-domain network mostly has one administration, incremental/partial deployment may still exist due to phased deployment or the limitations coming from multi-vendor supplement. As described in {{sec-arch-agent}}, under incremental/partial deployment, if complete SAV-specific information is unavailable, SAV rules can be generated by combing available SAV-specific information and local routing information. 

The implementation of Validation Entity is RECOMMENDED to support flexible validation modes such as interface-based prefix allowlist, interface-based prefix blocklist, and prefix-based interface allowlist {{I-D.huang-savnet-sav-table}}. The first two modes are interface-scale, and the last one is device-scale. Under incremental/partial deployment, the device of Validation Entity SHOULD take on the proper validation mode according to the deploying of Source Entities. For example, if Validation Entity is able to get the complete set of legitimate source prefixes arriving at a given interface, interface-based prefix allowlist can be enabled at the given interface, and false positive will not exist. 

In addition, the SAV filtering at the router can be also performed incrementally. The router can first take conservative actions on the validated data packets. That is to say, the router will not directly discard packet with an invalid result in the beginning of deployment. It can conduct sampling action for measurement analysis at first, and then conducts rate-limiting action or redirecting action for packets with invalid results. These conservative actions will not result in serious consequences under false positive validation results, while still providing protection for the network. Finally, filtering action is enabled only after confirming that there are no false positives.


# Security Considerations
In many cases, an intra-domain network can be considered as a trusted domain. There will be no threats within the domain. 

However, in some other cases, devices within the domain do not trust each other. Operators SHOULD be aware of potential threats involved in deploying the architecture. Typically, the potential threats and solutions are as follows: 

- Entity impersonation. 
  - Potential solution: Mutual authentication SHOULD be conducted before session establishment between two entities. 
  - Gaps: Impersonation may still exist due to credential theft, implementation flaws, or entity being compromised. Some other security mechanisms can be taken to make such kind of impersonation difficult. Besides, the entities SHOULD be monitored so that misbehaved entities can be detected. 

- Message blocking. 
  - Potential solution: Acknowledgement mechanisms MUST be provided in the session between a speaker and a receiver, so that message losses can be detected. 
  - Gaps: Message blocking may be a result of DoS/DDoS attack, man-in-the-middle (MITM) attack, or congestion induced by traffic burst. Acknowledgement mechanisms can detect message losses but cannot avoid message losses. MITM attacks cannot be effectively detected by acknowledgement mechanisms. 

- Message alteration. 
  - Potential solution: An authentication field can be carried by each message so as to ensure message integrity. 
  - Gaps: More overhead of control plane and data plane will be induced. 

- Message replay. 
  - Potential solution: Authentication value can be computed by adding a sequence number or timestamp as input. 
  - Gaps: More overhead of control plane and data plane will be induced. 

The above security threats SHOULD be considered when designing the new intra-domain SAV mechanism.


# Manageability Considerations
The architecture provides a general framework for communicating SAV-specific information between routers and generating SAV rules based on SAV-specific information and local routing information. Protocol-independent mechanisms SHOULD be provided for operating and managing SAV-related configurations. For example, a YANG data model for SAV configuration and operation is necessary for the ease of management. 

SAV may affect the normal forwarding of data packets. The diagnosis approach and necessary logging information SHOULD be provided. SAV Information Base SHOULD store some information that may not be useful for rule generation but is helpful for management.  

The SAV-specific information communication mechanism SHOULD have monitoring and troubleshooting functions, which are necessary for efficiently operating the architecture. 


# Privacy Considerations
An intra-domain network is mostly operated by a single organization or company, and the advertised SAV-specific information is only used within the network. Therefore, the architecture does not import critical privacy issues in usual cases. 


# IANA Considerations

This document has no IANA requirements.

# Acknowledgements

Many thanks to the valuable comments from: Igor Lubashev, Alvaro Retana, Aijun Wang, Joel Halpern, Jared Mauch, Kotikalapudi Sriram, Rüdiger Volk, Jeffrey Haas, Xiangqing Chang, Changwang Lin, etc.

--- back



