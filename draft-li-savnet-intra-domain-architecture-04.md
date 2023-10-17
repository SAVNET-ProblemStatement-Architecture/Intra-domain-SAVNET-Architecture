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

This document proposes the intra-domain SAVNET architecture. It can achieve more accurate Source address validation (SAV) than existing intra-domain SAV mechanisms that only use the router's local routing information to generate SAV rules. In this architecture, routers in the intra-domain network can generate SAV rules based on not only local routing information but also SAV-specific information. 

This document primarily concentrates on proposing SAV-specific information and introducing how to combine this information with local routing information to generate SAV rules. It provides the high-level framework and practical considerations for developing future intra-domain SAV mechanisms. The detailed designs of new intra-domain SAV mechanisms are not the focus of this document.

--- middle

# Introduction {#sec-intro}
Source address validation (SAV) is important for mitigating source address spoofing and contributing to the Internet security. Source Address Validation Architecture (SAVA) [RFC5210] divides SAV into three checking levels, i.e., access-network SAV, intra-domain SAV, and inter-domain SAV. When an access network does not deploy SAV (such as SAVI {{?RFC7039}}{{?RFC7513}}, Cable Source Verify {{cable-verify}}, and IP Source Guard {{IPSG}}), intra-domain SAV helps block spoofed packets from the access network as close to the source as possible. The concept of intra-domain SAV has been defined in {{I-D.ietf-savnet-intra-domain-problem-statement}}. 

The main task of SAV mechanisms is to generate SAV rules for checking the validity of source addresses of data packets. The information of source addresses/prefixes and their legitimate incoming directions makes up the SAV rules. How to efficiently and accurately learn the information is the core challenge for SAV mechanisms. 
Although many intra-domain SAV mechanisms (such as ACL-based filtering [RFC2827], strict uRPF [RFC3704], and loose uRPF [RFC3704]) have been proposed, they all have the problems of inaccurate validation or high operational overhead in some scenarios because they generate SAV rules using either the router's local routing information or manual configurations {{I-D.ietf-savnet-intra-domain-problem-statement}}. To address these problems, five requirements of future intra-domain mechanisms are proposed in {{I-D.ietf-savnet-intra-domain-problem-statement}}, i.e., automatic update, accurate validation, working in incremental/partial deployment, fast convergence, and necessary security guarantee. 

This document introduces the intra-domain SAVNET architecture to meet the above requirements. Consider that it is difficult to generate accurate SAV rules solely using the router's local routing information in asymmetric routing scenarios, and manual configurations/updates require high operational overhead. The intra-domain SAVNET architecture defines the SAV-specific information that helps routers generate SAV rules accurately and automatically. Routers in the intra-domain network can automatically communicate SAV-specific information through the SAV-specific information communication mechanism. In the incremental/partial deployment scenario where only part of intra-domain routers support the intra-domain SAVNET architecture, SAV-specific information of some routers cannot be available. In this scenario, routers can use both local routing information and received SAV-specific information to avoid improper block problems and reduce improper permit problems. 

This document also provides convergence, security, manageability, and privacy considerations for future intra-domain SAV mechanisms. The detailed designs of new intra-domain SAV mechanisms are not included in this document.

## Requirements Language

{::boilerplate bcp14-tagged}


# Terminology

SAV Rule: The rule that indicates the validity of a specific source IP address or source IP prefix.

SAV Table: The table or data structure that implements the SAV rules and is used for source address validation.

Local Routing Information: The information stored in the router's local RIB or FIB that can be used to infer SAV rules in addition to the routing purpose. 

SAV-specific Information: The information specialized for SAV rule generation that is communicated between routers. For example, it can notify routers the source prefixes of the specific subnet. 

SAV-specific Information Communication Mechanism: The mechanism for communicating SAV-specific information between routers. The mechanism can be a new protocol or an extension to an existing protocol. 

SAV Information Base: A table or data structure for storing SAV-specific information and local routing information. 

SAVNET Agent: The module that generates SAV rules by processing SAV-specific information and local routing information. 

Edge Router: An intra-domain router that is connected to intra-domain subnets.

Border Router: An intra-domain router that is connected to other ASes. A router can be both an edge router and a border router, if it is connected to both subnets and other ASes.

Improper Block: The validation results that the packets with legitimate source addresses are blocked improperly due to inaccurate SAV rules.

Improper Permit: The validation results that the packets with spoofed source addresses are permitted improperly due to inaccurate SAV rules.


# Intra-domain SAVNET Architecture Overview {#sec-arch-overview}

{{fig-arch}} shows the overview of intra-domain SAVNET architecture. In the architecture, there is a communication channel connecting two entities, i.e., Source Entity and Validation Entity:

- Source Entity sends SAV-specific information to Validation Entity. In the intra-domain network, it can be an edge router (i.e., the router connected to intra-domain subnets) that can obtain the SAV-specific information about intra-domain subnets. 

- Validation Entity receives and processes the SAV-specific information from Source Entity, generates SAV rules, and conducts SAV. In the intra-domain network, it can be either an edge router or a border router (i.e., the router connected to other ASes) that can perform SAV on inbound or outbound packets. 

A router can be an edge router, a border router, both an edge router and a border router, or other router (i.e., neither an edge router nor a border router). An edge router can act as a Source Entity to send its SAV-specific information to other routers, and it can also act as a Validation Entity to receive SAV-specific information from other routers at the same time.

~~~
+-------------------+               +---------------------------------------+
|   Source Entity   | Communication |           Validation Entity           |
| +---------------+ | Channel       | +---------------+   +---------------+ |
| |  Information  +------------------->  Information  |   |    RIB/FIB    | |
| |  Speaker      | |(SAV-specific  | |  Receiver     |   |               | |
| +---------------+ | Information ) | +---------------+   +---------------+ |
|                   |               |         |SAV-specific   |Local Routing|
|                   |               |         |Information    |Information  |
|                   |               |      +--v---------------v--------+    |
|                   |               |      |           SAVNET          |    |
|                   |               |      |           Agent           |    |
|                   |               |      +---------------------------+    |
+-------------------+               +---------------------------------------+
~~~
{: #fig-arch title="The intra-domain SAVNET architecture"}

Information Speaker in Source Entity is responsible to send SAV-specific information to Information Receiver in Validation Entity through the communication channel. To this end, a SAV-specific information communication mechanism should be developed to build and maintain the communication channel. SAVNET Agent in Validation Entity consolidates and processes SAV-specific information received from Information Receiver as well as local routing information from local RIB/FIB, and finally generates SAV rules. 

In the following, {{sec-information}} introduces the local routing information and SAV-specific information, and the SAV-specific information communication mechanism. {{sec-arch-agent}} introduces the basic workflow of SAVNET Agent. Then, {{sec-use-case}} uses two representative use cases to illustrate that intra-domain SAVNET architecture can improve the accuracy and automation capability of SAV upon existing intra-domain SAV mechanisms. Finally, this document provides convergence, incremental/partial deployment, security, manageability, and privacy considerations for designing new intra-domain SAV mechanisms.

# Local Routing Information and SAV-specific Information {#sec-information}

As described in {{sec-arch-overview}} and shown in {{fig-arch}}, the SAVNET Agent in Validation Entity uses both local routing information and SAV-specific information to generate SAV rules. This section introduces both types of information and how they can be obtained.

## Local Routing Information

Local routing information is used for forwarding rule computation, which is stored in RIB/FIB. Although it is not specialized for SAV, it can also be used to infer SAV rules in existing uRPF-based SAV mechanisms, such as strict uRPF and loose uRPF. This information can be directly obtained from the router's RIB/FIB.

## SAV-specific Information

SAV-specific information is specialized for SAV. The Source Entity can obtain local SAV-specific information based on local routing information and local interface configurations. The Validation Entity can obtain the SAV-specific information of the Source Entity through the communication channel between Source and Validation Entities.

For example, SAV-specific information can help the Validation Entity identify the source prefixes of intra-domain subnets. The Source Entity that is directly connected to subnets can inform the Validation Entity of its locally known source prefixes of its subnets. The Validation Entity can consolidate such SAV-specific information and local routing information to obtain the complete source prefixes of intra-domain subnets. 

By learning the SAV-specific information of the Source Entity, the Validation Entity can generate more accurate SAV rules than solely using its local routing information. In this way, improper block problems can be avoided and improper permit problems can be reduced.

### SAV-specific Information Communication Mechanism

The SAV-specific communication mechanism is for building the communication channel and propagating SAV-specific information from Source Entity to Validation Entity. Since there is no off-the-shelf mechanism to achieve this function, a new SAV-specific communication mechanism is needed. It can be a new protocol or an extension to an existing protocol. This document does not present the details of the protocol design or protocol extensions. In the following, we describe the necessary features of SAV-specific communication mechanism.

The SAV-specific communication mechanism SHOULD define the data structure or format of communicated SAV-specific information, and the operations of communication (such as communication channel establishment and communication channel termination). In addition, the mechanism SHOULD require Source Entity to inform Validation Entity of the updates of SAV-specific information in a timely manner, so that Validation Entity can quickly update SAV rules. 

In order to ensure the convergence and security of the communication, the session of the SAV-specific communication mechanism SHOULD meet the following requirements: 

- The session can be a long-time session or a temporary one, but it SHOULD provide sufficient assurance of transmission reliability and timeliness, so that Validation Entity can update its SAV rules in time. 

- Authentication can be conducted before session establishment. Authentication is optional but the ability of authentication SHOULD be available. 

# SAVNET Agent {#sec-arch-agent}

SAVNET Agent is the module in Validation Entity that generates SAV rules by processing local routing information and received SAV-specific information. {{fig-sav-agent}} shows the workflow of SAVNET Agent. The SAV Information Manager of SAVNET Agent consolidates SAV-specific information from Information Receiver and local routing information from RIB/FIB into the SAV Information Base. Then, it sends the consolidated information to the SAV Rule Generator. The SAV Rule Generator will generate SAV rules (e.g., tuples like <prefix, interface set, validity state>) based on the consolidated information. It then stores SAV rules in the SAV Table {{I-D.huang-savnet-sav-table}}. SAV Information Manager also provides the support of diagnosis. Operators can look up the information in SAV Information Base for monitoring or troubleshooting purpose. 

~~~
SAV-specific information from Information Receiver,
and local routing information from RIB/FIB
                    |
    +---------------|----------------+
    | SAVNET Agent  |                |
    | +-------------\/-------------+ |
    | | SAV Information Manager    | |
    | | +------------------------+ | |
    | | | SAV Information Base   | | |
    | | +------------------------+ | |
    | +-------------+--------------+ |
    | SAV-specific  |  local routing |
    | information   |  information   |
    |               |                |
    | +-------------\/-------------+ |
    | | SAV Rule Generator         | |
    | | +------------------------+ | |
    | | | SAV Table              | | |
    | | +------------------------+ | |
    | +----------------------------+ |
    +--------------------------------+
~~~
{: #fig-sav-agent title="The workflow of SAVNET Agent"}

As mentioned in {{sec-arch-overview}}, the Validation Entity can be either an edge router or a border router. According to the definition of intra-domain SAV proposed in {{I-D.ietf-savnet-intra-domain-problem-statement}}, the edge router is responsible for validating packets received from the connected subnet and blocking those with source addresses of other subnets. While the border router is responsible for validating packets received from other ASes and blocking those with source addresses in internal source prefixes. Therefore, the working mechanism of SAVNET Agents of edge router and border router should be different, especially in the incremental/partial deployment scenario.

For the SAVNET Agent of an edge router, it should first combine the local routing information and the SAV-specific information to obtain the complete source prefixes of each connected subnet. Then, it binds the source prefixes of the subnet to its interface connected to the subnet, and conduct SAV at this interface. Packets arriving at the interface will be considered invalid and should be blocked, if their source addresses do not belong to the subnet.
In the incremental/partial deployment scenario where not all edge routers connected to the same subnets deploy SAV-specific information communication mechanism, an edge router acting as a Validation Entity may not be able to identify complete source prefixes of the subnet. To avoid improper problems in this case, the Validation Entity is recommended to use less strict SAV rules. For example, it can choose to only block packets with non-global or non-routable source addresses by using its local routing information. 

For the SAVNET Agent of a border router, it should combine the local routing information and the SAV-specific information to collect all internal source prefixes, i.e., the source prefixes of all internal subnets. Then, it binds the internal source prefixes to the interfaces connected to other ASes and conducts SAV at these interfaces. Packets arriving at these interfaces will be considered invalid and should be blocked, if their source addresses belong to internal source prefixes. 
In the incremental/partial deployment scenario where not all edge routers in the intra-domain network deploy SAV-specific information communication mechanism, a border router acting as a Validation Entity may not be able to get complete internal source prefixes. In this scenario, the border router can still block inbound packets with source addresses belonging to the learned internal source prefixes. 
In addition, if the border router also implements inter-domain SAVNET architecture, its intra-domain SAVNET Agent can send the intra-domain SAV-specific information to its inter-domain SAVNET Agent, helping the inter-domain SAVNET Agent generate inter-domain SAV rules or inter-domain SAV-specific information.


# Use Cases of Intra-domain SAVNET Architecture {#sec-use-case}

This section uses two use cases to illustrate that intra-domain SAVNET architecture can achieve more accurate and efficient SAV than existing intra-domain SAV mechanisms, both in the inbound and outbound directions.

## Use Case 1: Validating Outbound Packets from a Multi-homed Subnet at Edge Routers

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

In this case, as described in {{I-D.ietf-savnet-intra-domain-problem-statement}}, strict uRPF at Router 1 will improperly block legitimate packets with source addresses in prefix 10.0.0.0/16 from Subnet 1 at interface '#', because it only accepts packets with source addresses in prefix 10.1.0.0/16 from Router 1's interface '#' according to its local routing information.

If intra-domain SAVNET architecture is implemented in the network, Router 2 can inform Router 1 that prefix 10.0.0.0/16 also belongs to Subnet 1 by sending SAV-specific information to Router 1. Then, by combining both local routing information and received SAV-specific information, Router 1 learns that Subnet 1 owns both prefix 10.1.0.0/16 and prefix 10.0.0.0/16. Therefore, Router 1 will accept packets with source addresses in prefix 10.1.0.0/16 and prefix 10.0.0.0/16 at interface '#', so improper block can be avoided. 

## Use Case 2: Validating Inbound Packets from Other ASes at Border Routers

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

As described in {{I-D.ietf-savnet-intra-domain-problem-statement}}, if Router 3 and Router 4 deploy ACL-based ingress filtering, the operator needs to manually generate and update ACL rules at Router 3 and Router 4 when internal source prefixes change. The operational overhead of manually maintaining and updating ACL rules will be extremely high, especially when there are multiple inbound validation points. 

If intra-domain SAVNET architecture is implemented in the network, Router 1, Router 2, and Router 5 will inform Router 3 and Router 4 of the source prefixes of Subnet 1 and Subnet 2 by sending SAV-specific information. After receiving the SAV-specific information, Router 3 and Router 4 can identify internal source prefixes and detect their changes. In this way, Router 3 and Router 4 can automatically generate and update SAV rules at interface '#' to block inbound packets with internal source addresses. 

# How Does Intra-domain SAVNET Architecture Satisfy the Design Requirements？

This document presents an intra-domain SAVNET architecture that matches the five requirements proposed in {{I-D.ietf-savnet-intra-domain-problem-statement}}:

- For accurate validation, intra-domain SAVNET architecture proposes the SAV-specific information. Routers in the intra-domain network can combine SAV-specific information with local routing information to generate accurate SAV rules. For example, by collecting SAV-specific information and local routing information, edge routers can accurately identify the complete source prefixes of the connected subnet, and border routers can accurately identify the complete internal source prefixes.

- For automatic update, intra-domain SAVNET architecture allows routers to communicate SAV-specific information between each other automatically. After receiving SAV-specific information from Source Entities, the Validation Entity can generate and update its SAV rules accordingly.

- For working in incremental/partial deployment, intra-domain SAVNET architecture requires routers to use both local routing information and learnt SAV-specific information to generate SAV rules, in order to avoid improper block when some SAV-specific information is not available. In addition, Validation Entities are proposed to support flexible validation modes and perform SAV filtering incrementally to smooth the transition from partial to full deployment. More practical recommendations for incremental/partial deployment can be found in {{sec-incre}}.

- For fast convergence, intra-domain SAVNET architecture requires routers to update SAV-specific information and SAV rules in a timely manner. For the Source Entity, it MUST send its updated SAV-specific information to the Validation Entity timely. For the Validation Entity, it MUST detect the changes of local routing information and received SAV-specific information in time and update SAV rules with the latest information. More convergence considerations for designing future intra-domain SAV mechanisms can be found in {{sec-converge}}.

- For necessary security guarantee, intra-domain SAVNET architecture suggests that the new intra-domain SAV mechanisms SHOULD consider and avoid some potential security threats, such as entity impersonation. More security considerations for designing future intra-domain SAV mechanisms can be found in {{sec-security}.

# Incremental/Partial Deployment Considerations {#sec-incre}

Although an intra-domain network mostly has one administration, incremental/partial deployment may still exist due to phased deployment or the limitations coming from multi-vendor supplement. In the phased deployment, it is RECOMMENDED that the edge routers connected to same subnet can be upgraded together so that the complete source prefixes of the subnet can be obtained by other routers. 
For example, in {{fig-use-case1}}, Router 1 and Router 2 are recommended to be upgraded together so that the two routers can obtain complete source prefixes of Subnet 1 and generate accurate SAV rules. 

As described in {{sec-arch-agent}}, the architecture can adapt to the incremental/partial deployment scenario. Under incremental/partial deployment, if complete SAV-specific information is unavailable, SAV rules can be generated by combining available SAV-specific information and local routing information. 

The implementation of Validation Entity is RECOMMENDED to support flexible validation modes such as interface-based prefix allowlist, interface-based prefix blocklist, and prefix-based interface allowlist {{I-D.huang-savnet-sav-table}}. The first two modes are interface-scale, and the last one is device-scale. Under incremental/partial deployment, the Validation Entity SHOULD take on the proper validation mode according to the deploying of Source Entities. For example, if Validation Entity is able to get the complete set of legitimate source prefixes arriving at a given interface, interface-based prefix allowlist can be enabled at the given interface, and improper block will not exist. 

In addition, the SAV filtering at the router can be also performed incrementally. The router can first take conservative actions on the validated data packets. That is to say, the router will not directly discard packet with an invalid result in the beginning of deployment. It can conduct sampling action for measurement analysis at first, and then conducts rate-limiting action or redirecting action for packets with invalid results. These conservative actions will not result in serious consequences if some legitimate packets are mistakenly considered invalid, while still providing protection for the network. Finally, filtering action is enabled only after confirming that there are no improper block problems.

# Convergence Considerations {#sec-converge}

When the SAV-specific information or local routing information changes, the SAVNET Agent MUST be able to detect the changes in time and update SAV rules with the latest information. Since SAV-specific information is originated from the Source Entity, it requires the Source Entity MUST send the updated SAV-specific information to the Validation Entity in a timely manner. For example, in {{fig-use-case2}}, if Subnet 2 has a new source prefix P3, Router 5 MUST inform Router 3 and Router 4 of the new source prefix of Subnet 2 immediately. 

Consider that both routing information and SAV-specific information of a subnet are originated and advertised to other routers in the network by the edge router connected to the subnet. Thus, SAV-specific information has similar propagation speed as routing information. 

Even though, there will still be delays of message delivery (sometimes re-transmission delay due to packet loss) and information processing. Therefore, during the convergence process, the SAV rules for checking packets are possibly inaccurate, which may result in improper block or improper permit. Existing uRPF-based SAV mechanisms that solely use local routing information are also faced with similar convergence problems. Inaccurate validation may appear during the convergence of routing, which is inevitable in practice.

To mitigate the impact of convergence problems and avoid improper block, future intra-domain SAV mechanisms MUST carefully consider the factors that may affect the convergence performance.

# Security Considerations {#sec-security}

Typically, routers in an intra-domain network can trust each other because they would not compromise intra-domain control-plane architectures and protocols.

However, in some unlikely cases, some routers may do harm to other routers within the same domain. Operators SHOULD be aware of potential threats involved in deploying the architecture. Some potential threats and solutions are as follows: 

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



