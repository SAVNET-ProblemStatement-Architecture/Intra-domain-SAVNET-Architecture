---
title: Intra-domain Source Address Validation (SAVNET) Architecture
abbrev: Intra-domain SAVNET Architecture
docname: draft-li-savnet-intra-domain-architecture-05
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
  ins: L. Qin
  name: Lancheng Qin
  organization: Tsinghua University
  email: qlc19@mails.tsinghua.edu.cn
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
  ins: L. Chen
  name: Li Chen
  organization: Zhongguancun Laboratory
  email: lichen@zgclab.edu.cn
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

This document proposes the intra-domain SAVNET architecture, which achieves accurate source address validation (SAV) in an intra-domain network by an automatic way. Compared with uRPF-like SAV mechanisms that only depend on routers' local routing information, SAVNET routers generate SAV rules by using both local routing information and SAV-specific information exchanged among routers, resulting in more accurate SAV validation in asymmetric routing scenarios. Compared with ACL rules that require manual efforts to accommodate to network dynamics, SAVNET routers learn the SAV rules automatically in a distributed way.

--- middle

# Introduction {#sec-intro}
Source address validation (SAV) is important for mitigating source address spoofing and thus contributes to the Internet security. In the Source Address Validation Architecture (SAVA) [RFC5210], SAV is divided into three checking levels, i.e., access-network SAV, intra-domain SAV, and inter-domain SAV. When an access network does not deploy SAV (such as SAVI {{?RFC7039}}{{?RFC7513}}, Cable Source Verify {{cable-verify}}, and IP Source Guard {{IPSG}}), intra-domain SAV helps block spoofed packets from an access network as close to the source as possible {{I-D.ietf-savnet-intra-domain-problem-statement}}. 

The main purpose of the intra-domain SAV mechanism for an AS A, is to protect the outgoing packets of a subnet of AS A from forging their source addresses as other subnets' addresses or other ASes' addresses, as well as protect the incoming packets to AS A from forging their source addresses as AS A's addresses. The main task of the intra-domain SAV mechanism is to generate the correct mapping relationship between a source address (prefix) and the valid incoming interface(s), called SAV rules. The core challenge of the intra-domain SAV mechanism is how to efficiently and accurately learn the mapping relationship. Although many existing intra-domain SAV mechanisms (such as ACL-based filtering [RFC2827], strict uRPF [RFC3704], and loose uRPF [RFC3704]) have been proposed, they suffer from either inaccurate mapping in asymmetric routing scenraios, or high operational overhead in dynamic networks. The key cause is that exsiting mechanisms generate the SAV rules by a router's local routing information or by manual inputs. In {{I-D.ietf-savnet-intra-domain-problem-statement}}, five requirements for a new intra-domain SAVNET architecture are proposed.

This document introduces the intra-domain SAVNET architecture to meet the five requirements. The key idea of intra-domain SAVNET is to generate SAV rules in routers based on SAV-specific information exchanged among routers, instead of depending on local routing information like in existing mechanisms. It achieves accurate SAV validation, because the SAV-specific information exchanged among routers carry necessary information for asymmetric routing scenraio which cannot be learned by local routing information. It achieves automatic SAV rule update, because the SAV-specific information exchange is triggered when there is topology change or prefix change. 

In the incremental/partial deployment scenario where only part of intra-domain routers support the intra-domain SAVNET, some SAV-specific information for building the complete SAV rules will be missing. In this case, local routing information can still be used to fill the gap. 

## Requirements Language

{::boilerplate bcp14-tagged}


# Terminology

Local Routing Information: The information in a router's local RIB or FIB that can be used to infer SAV rules.

SAV-specific Information: The information specialized for SAV rule generation, which is exchanged among routers.

SAV-related Information: The information used by a router to make SAV decisions. For intra-domain SAV, SAV-related information includes both local routing information and SAV-specific information.

SAV-specific Information Communication Mechanism: The mechanism for exchanging SAV-specific information between routers. It can be a either a new protocol or an extension to an existing protocol.

SAV Information Base: A table or data structure in a router which stores SAV-specific information and local routing information.

SAV Rule: The rule in a router that describes the mapping relationship between a source address (prefix) and the valid incoming interface(s). It is used by a router to make SAV decisions and is inferred from the SAV Information Base.

SAVNET Router: A router which runs intra-domain SAVNET.

SAVNET Agent: The agent in a SAVNET router that is responsible for communicating SAV-specific information, processing SAV-related information, and generating SAV rules.

Edge Router: An intra-domain router for an AS that is directly connected to a subnet of the AS.

Border Router: An intra-domain router for an AS that is connected to other ASes. A router in an AS can be both an edge router and a border router, if it is connected to both the AS's subnets and other ASes.

Improper Block: The validation results that the packets with legitimate source addresses are blocked improperly due to inaccurate SAV rules.

Improper Permit: The validation results that the packets with spoofed source addresses are permitted improperly due to inaccurate SAV rules.


# Intra-domain SAVNET Architecture {#sec-arch-overview}

{{fig-arch}} shows the overview of intra-domain SAVNET architecture. A SAVNET router can be an edge router, a border router, both an edge router and a border router, or other routers (i.e., neither an edge router nor a border router). Every SAVNET router has a SAVNET Agent that is responsible for actions related to SAV.

## Roles of SAVNET Routers

A SAVNET router can act as one or two roles in the intra-domain SAVNET architecture, namely, source entity to send its SAV-specific information to other SAVNET routers, or/and validation entity to receive SAV-specific information from other SAVNET routers.

### Source Entity

When a SAVNET router acts as source entity, the information sender of its SAVNET Agent sends its SAV-specific information to other SAVNET routers that act as validation entity. For example, an edge router acting as source entity can obtain its SAV-specific information about the connected subnets and send out this information through communication channel. 

### Validation Entity

When a SAVNET router acts as validation entity, the information receiver of its SAVNET Agent receives SAV-specific information from other routers that act as source entity. Then, its SAVNET Agent processes the received SAV-specific information as well as its local routing information to generate SAV rules. For example, an edge router or a border router acting as validation entity can generate SAV rules and use SAV rules to perform SAV on inbound or outbound packets.

~~~
+---------------------+             +----------------------------------+
|    Source Entity    |             |         Validation Entity        |
|     (Router A)      |             |            (Router B)            |
|                     |             |                                  |
| +-----------------+ |             | +-----------------+  +---------+ |
| |   SAVNET Agent  | |Communication| |   SAVNET Agent  <--+ RIB/FIB | |
| | +-------------+ | |Channel      | | +-------------+ |  +---------+ |
| | | Information +---------------------> Information | | Local Routing|
| | | Sender      | | |(SAV-specific| | | Receiver    | | Information  |
| | +-------------+ | | Information)| | +-------------+ |              |
| +-----------------+ |             | +-----------------+              |
|                     |             |                                  |
+---------------------+             +----------------------------------+
~~~
{: #fig-arch title="Intra-domain SAVNET architecture"}

## SAV-related Information {#sec-information}

### Local Routing Information

Local routing information is used for computing packet forwarding rules, which is stored in the local RIB/FIB.  Although it is not specialized for SAV, it is widely used to infer SAV rules in existing uRPF-based SAV mechanisms, such as strict uRPF and loose uRPF. A SAVNET router acting as validation entity can directly obtain local routing information from the router's RIB/FIB, when the corresponding SAV-specific information is missing.

### SAV-specific Information

SAV-specific information is specialized for SAV. A SAVNET router acting as source entity can obtain local SAV-specific information based on local routing information and local interface configurations. A SAVNET router acting as validation entity can obtain SAV-specific information of other SAVNET routers that act as source entity through the SAV communication channel.

By learning the SAV-specific information from source entity, validation entity can generate more accurate SAV rules than solely using its local routing information, which lead to more accurate SAV. For example, a router that is directly connected to subnets can act as source entity and inform the other routers of its locally known source prefixes of its subnets. Other routers acting as validation entity can finally consolidate such SAV-specific information and local routing information to obtain the complete source prefixes of intra-domain subnets.

### SAV-specific Information Communication Mechanism

The SAV-specific communication mechanism is for building the communication channel and propagating SAV-specific information from source entity to validation entity.  Since there is no off-the-shelf mechanism to achieve this function, a new SAV-specific communication mechanism is needed. It can be a new protocol or an extension to an existing protocol. This document does not present the details of the protocol design or protocol extensions. In the following, we describe the necessary features of SAV-specific communication mechanism.

The SAV-specific communication mechanism SHOULD define the data structure or format of SAV-specific information, and the operations of communication (such as communication channel establishment and communication channel termination). In addition, the mechanism SHOULD require source entity to inform validation entity of the updates of SAV-specific information in a timely manner, so that validation entity can update SAV rules based on the latest information.

In order to ensure the convergence and security of the communication, the session of the SAV-specific communication mechanism SHOULD meet the following requirements:

- The session can be a long-time session or a temporary one, but it SHOULD provide sufficient assurance of transmission reliability and timeliness, so that validation entity can update its SAV rules in time.

- Authentication can be conducted before session establishment. Authentication is optional but the ability of authentication SHOULD be available.

## SAV Rule Generation {#sec-arch-agent}

{{fig-sav-agent}} shows the SAV rule generation process of the SAVNET router acting as validation entity. The SAV Information Manager of SAVNET Agent consolidates SAV-specific information from Information Receiver and local routing information from RIB/FIB into the SAV Information Base. Then, it sends the consolidated information to the SAV Rule Generator. The SAV Rule Generator will generate SAV rules (e.g., tuples like <prefix, interface set, validity state>) based on the consolidated information. 

SAV Information Manager also provides the support of diagnosis. Operators can look up the information in SAV Information Base for monitoring or troubleshooting purpose. 

~~~
+--------------------------------------------------------+
|                      SAVNET Agent                      |
|                                                        |
| SAV-specific information     Local routing information |
| from Information Receiver    from RIB/FIB              |
|                |                   |                   |
|                |                   |                   |
|            +---v-------------------v----+              |
|            | SAV Information Manager    |              |
|            | +------------------------+ |              |
|            | | SAV Information Base   | |              |
|            | +------------------------+ |              |
|            +----------------------------+              |
|                          |                             |
|                          | SAV-related information     |
|                          |                             |
|            +-------------v--------------+              |
|            | SAV Rule Generator         |              |
|            | +------------------------+ |              |
|            | |        SAV Rules       | |              |
|            | +------------------------+ |              |
|            +----------------------------+              |
+--------------------------------------------------------+
~~~
{: #fig-sav-agent title="Workflow of SAV rule generation"}

As mentioned in {{sec-arch-overview}}, validation entity can be either an edge router or a border router. According to the definition of intra-domain SAV proposed in {{I-D.ietf-savnet-intra-domain-problem-statement}}, the edge router is responsible for validating packets received from the connected subnet and blocking those with source addresses of other subnets. While the border router is responsible for validating packets received from other ASes and blocking those with source addresses in internal source prefixes. Therefore, the working mechanisms of SAVNET Agents of edge router and border router are different, especially in the incremental/partial deployment scenario.

For an edge router acting as validation entity, it should first combine the local routing information and the SAV-specific information to obtain the complete source prefixes of each connected subnet. Then, it binds the source prefixes of the subnet to its interface connected to the subnet, and conduct SAV at this interface. Packets arriving at the interface will be considered invalid and should be blocked, if their source addresses do not belong to the subnet. In the incremental/partial deployment scenario where not all edge routers connected to the same subnets deploy SAV-specific information communication mechanism, an edge router acting as validation entity may not be able to identify complete source prefixes of the subnet. To avoid improper block problems in this case, validation entity is recommended to use less strict SAV rules. For example, it can choose to only block packets with non-global or non-routable source addresses by using its local routing information.

For a border router acting as validation entity, it should combine the local routing information and the SAV-specific information to collect all internal source prefixes, i.e., the source prefixes of all internal subnets. Then, it binds the internal source prefixes to the interfaces connected to other ASes and conducts SAV at these interfaces. Packets arriving at these interfaces will be considered invalid and should be blocked, if their source addresses belong to internal source prefixes. In the incremental/partial deployment scenario where not all edge routers in the intra-domain network deploy SAV-specific information communication mechanism, a border router acting as validation entity may not be able to get complete internal source prefixes. In this scenario, the border router can still block inbound packets with source addresses belonging to the learned internal source prefixes. In addition, if the border router also implements inter-domain SAVNET, its intra-domain SAVNET Agent SHOULD send the intra-domain SAV-specific information to its inter-domain SAVNET Agent, helping the inter-domain SAVNET Agent generate inter-domain SAV rules or inter-domain SAV-specific information.



# Use Cases {#sec-use-case}

This section uses two use cases to illustrate that intra-domain SAVNET can achieve more accurate and efficient SAV than existing intra-domain SAV mechanisms, both in the inbound and outbound directions.

## Use Case 1: Validating Outbound Packets from a Multi-homed Subnet at Edge Routers {#sec-use-case1}

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

If intra-domain SAVNET is implemented in the network, Router 2 can inform Router 1 that prefix 10.0.0.0/16 also belongs to Subnet 1 by sending SAV-specific information to Router 1. Then, by combining both local routing information and received SAV-specific information, Router 1 learns that Subnet 1 owns both prefix 10.1.0.0/16 and prefix 10.0.0.0/16. Therefore, Router 1 will accept packets with source addresses in prefix 10.1.0.0/16 and prefix 10.0.0.0/16 at interface '#', so improper block can be avoided. 

## Use Case 2: Validating Inbound Packets from Other ASes at Border Routers {#sec-use-case2}

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

If intra-domain SAVNET is implemented in the network, Router 1, Router 2, and Router 5 will inform Router 3 and Router 4 of the source prefixes of Subnet 1 and Subnet 2 by sending SAV-specific information. The SAV-specific information communication will be triggered if topology or prefix changes. For example, if Subnet 2 has a new source prefix P3, Router 5 will inform Router 3 and Router 4 of the new source prefix of Subnet 2 immediately. After receiving SAV-specific information from other routers, Router 3 and Router 4 can identify internal source prefixes and detect their changes. In this way, Router 3 and Router 4 can automatically generate and update SAV rules at interface '#' to block inbound packets with internal source addresses. 

# Meeting the Design Requirements of Intra-domain SAVNET

Intra-domain SAVNET architecture is proposed to meet the five design requirements proposed in {{I-D.ietf-savnet-intra-domain-problem-statement}}.

## Accurate Validation

In the asymmetric routing scenario shown in {{fig-use-case1}}, the edge router cannot identify the complete source prefixes of the connected subnet only using the router's local routing information. As a result, strict uRPF has improper block problems in the case of asymmetric routing, because it only uses local routing information to generate SAV rules.

Intra-domain SAVNET uses SAV-specific information and requires routers to exchange SAV-specific information among each other. The SAVNET router acting as validation entity can combine SAV-specific information with local routing information to generate accurate SAV rules. The use case in {{fig-use-case1}} has shown that intra-domain SAVNET can achieve more accurate SAV validation compared with strict uRPF in asymmetric routing scenarios.


## Automatic Update

In real intra-domain networks, the topology or prefixes of networks may change dynamically. The SAV mechanism MUST automatically update the SAV rules as the network changes. However, ACL-based SAV mechanism requires manual efforts to accommodate to network dynamics, resulting in high operational overhead.

Intra-domain SAVNET allows SAVNET routers to exchange the changes of SAV-specific information among each other automatically. After receiving updated SAV-specific information from source entity, SAVNET routers acting as validation entity can generate and update their SAV rules accordingly. The use case in {{sec-use-case2}} has shown that intra-domain SAVNET can achieve automatic update.

## Incremental/Partial Deployment {#sec-incre}

Although an intra-domain network mostly has one administration, incremental/partial deployment may still exist due to phased deployment or multi-vendor supplement. In phased deployment scenarios, SAV-specific information of non-deploying routers is not available, so SAVNET routers acting as validation entity may not obtain some needed SAV-specific information.

As described in {{sec-arch-agent}}, the architecture can adapt to incremental/partial deployment. If complete SAV-specific information cannot be learned by only SAV-specific information, local routing information can be used to fill the gap. To mitigate the impact of phased deployment, it is also RECOMMENDED that edge routers connected to the same subnet can simultaneously adopt intra-domain SAVNET so that the complete source prefixes of the subnet can be obtained by other routers. For example, in {{fig-use-case1}}, Router 1 and Router 2 are recommended to be upgraded to SAVNET routers together so that the two routers can obtain complete source prefixes of Subnet 1 and generate accurate SAV rules. 

In addition, SAVNET routers acting as validation entity are RECOMMENDED to support flexible validation modes and perform SAV filtering incrementally to smooth the transition from partial to full deployment.

- Validation entity is RECOMMENDED to support flexible validation modes such as interface-based prefix allowlist, interface-based prefix blocklist, and prefix-based interface allowlist {{I-D.huang-savnet-sav-table}}. The first two modes are interface-scale, and the last one is device-scale. Under incremental/partial deployment, validation entity SHOULD take on the proper validation mode according to the deploying of source entity. For example, if validation entity is able to get the complete set of legitimate source prefixes arriving at a given interface, interface-based prefix allowlist can be enabled at the given interface, and improper block will not exist. 

- Validation entity is RECOMMENDED to performed SAV-invalid filtering incrementally. The router can first take conservative actions on the validated data packets. That is to say, the router will not directly discard packets with invalid results in the beginning of deployment. It can conduct sampling action for measurement analysis at first, and then conducts rate-limiting action or redirecting action for packets with invalid results. These conservative actions will not result in serious consequences if some legitimate packets are mistakenly considered invalid, while still providing protection for the network. Finally, filtering action is enabled only after confirming that there are no improper block problems.

## Convergence {#sec-converge}

When the SAV-specific information or local routing information changes, the SAVNET Agent MUST be able to detect the changes in time and update SAV rules with the latest information. Otherwise, outdated SAV rules may cause legitimate packets to be blocked or spoofed packets to be accepted.

Intra-domain SAVNET requires routers to update SAV-specific information and SAV rules in a timely manner. Since SAV-specific information is originated from source entity, it requires that the source entity MUST timely send the updated SAV-specific information to validation entity. Consider that both routing information and SAV-specific information of a subnet are originated and advertised to other routers in the network by the edge router connected to the subnet, SAV-specific information SHOULD have a similar propagation speed as routing information. Therefore, the converging speed of intra-domain SAVNET is also similar as routing protocol.

## Security {#sec-security}

Typically, routers in an intra-domain network can trust each other because they would not compromise intra-domain control-plane architectures and protocols.

However, in some unlikely cases, some routers may do harm to other routers within the same domain. Operators SHOULD be aware of potential threats involved in deploying the architecture. Some potential threats and solutions are as follows: 

- Entity impersonation. 
  - Potential solution: Mutual authentication SHOULD be conducted before session establishment between two entities. 
  - Gaps: Impersonation may still exist due to credential theft, implementation flaws, or entity being compromised. Some other security mechanisms can be taken to make such kind of impersonation difficult. Besides, the entities SHOULD be monitored so that misbehaved entities can be detected. 

- Message blocking. 
  - Potential solution: Acknowledgement mechanisms MUST be provided in the session between a sender and a receiver, so that message losses can be detected. 
  - Gaps: Message blocking may be a result of DoS/DDoS attack, man-in-the-middle (MITM) attack, or congestion induced by traffic burst. Acknowledgement mechanisms can detect message losses but cannot avoid message losses. MITM attacks cannot be effectively detected by acknowledgement mechanisms. 

- Message alteration. 
  - Potential solution: An authentication field can be carried by each message so as to ensure message integrity. 
  - Gaps: More overhead of control plane and data plane will be induced. 

- Message replay. 
  - Potential solution: Authentication value can be computed by adding a sequence number or timestamp as input. 
  - Gaps: More overhead of control plane and data plane will be induced. 

To meet the security requirement, the above security threats SHOULD be considered when designing the new intra-domain SAV mechanism.

# Manageability Considerations
The architecture provides a general framework for communicating SAV-specific information between routers and generating SAV rules based on SAV-specific information and local routing information. Protocol-independent mechanisms SHOULD be provided for operating and managing SAV-related configurations. For example, a YANG data model for SAV configuration and operation is necessary for the ease of management. 

SAV may affect the normal forwarding of data packets. The diagnosis approach and necessary logging information SHOULD be provided. SAV Information Base SHOULD store some information that may not be useful for SAV rule generation but is helpful for management.  
The SAV-specific information communication mechanism SHOULD have monitoring and troubleshooting functions, which are necessary for efficiently operating the architecture. 


# Privacy Considerations
An intra-domain network is mostly operated by a single organization or company, and the advertised SAV-specific information is used within the network. Therefore, the architecture will not import critical privacy issues in usual cases. 


# IANA Considerations

This document has no IANA requirements.

# Acknowledgements

Many thanks to the valuable comments from: Igor Lubashev, Alvaro Retana, Aijun Wang, Joel Halpern, Jared Mauch, Kotikalapudi Sriram, Rüdiger Volk, Jeffrey Haas, Xiangqing Chang, Changwang Lin, etc.

--- back



