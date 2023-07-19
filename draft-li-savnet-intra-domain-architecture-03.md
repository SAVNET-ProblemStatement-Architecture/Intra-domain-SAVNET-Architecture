---
title: Intra-domain Source Address Validation (SAVNET) Architecture
abbrev: Intra-domain SAVNET Architecture
docname: draft-li-savnet-intra-domain-architecture-03
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
  I-D.li-savnet-intra-domain-problem-statement:
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

This document proposes an intra-domain source address validation (intra-domain SAVNET) architecture. Devices in the intra-domain network can communicate with each other and share SAV-specific information besides routing information, which helps to generate SAV rules automatically and accurately. 

--- middle

# Introduction {#sec-intro}
Source address validation (SAV) is important for mitigating source address spoofing and contributing to the Internet security. Source Address Validation Architecture (SAVA) [RFC5210] divides SAV into three checking levels, i.e., access-network SAV, intra-domain SAV, and inter-domain SAV. When an access network does not deploy SAV (such as SAVI {{?RFC7039}}{{?RFC7513}}, Cable Source Verify {{cable-verify}}, and IP Source Guard {{IPSG}}), intra-domain SAV helps block spoofed packets from the access network as close to the source as possible. The concept of intra-domain SAV in this document keeps consistent with that in {{I-D.li-savnet-intra-domain-problem-statement}}. 

The main task of SAV mechanisms is to generate SAV rules for checking the validity of incoming packets. The information of source addresses/prefixes and their incoming directions makes up the SAV rules. How to efficiently and accurately learn the information is the core challenge for SAV mechanisms. 

There are many intra-domain SAV mechanisms available {{I-D.li-savnet-intra-domain-problem-statement}}. Some of them such as strict uRPF [RFC3704] and loose uRPF [RFC3704] generate SAV rules by using local routing information (e.g., the Forwarding Information Base). However, they may have false positive problems under asymmetric routing. To improve the accuracy of SAV, additional SAV-specific information must be used to generate correct SAV rules. But manually configuring SAV-specific information or directly configuring/modifying SAV rules (e.g., ACL-based filtering [RFC2827]) on routers is not practical for complex and dynamic networks, since the operational overhead can be extremely high. 

This document proposes an intra-domain source address validation (intra-domain SAVNET) architecture. Under the architecture, devices can communicate with each other and share SAV-specific information besides routing information. SAV rules can be generated accurately and automatically based on the learned SAV-related information. The proposed architecture aims to meet the requirements proposed in {{I-D.li-savnet-intra-domain-problem-statement}}. In addition, this document presents some considerations for partial deployment, security, manageability, and privacy. 

Intra-domain SAVNET architecture shows the high-level designs for future intra-domain SAV mechanisms. The protocols or protocol extensions for implementing the architecture are not in the scope of this document. 


## Terminology
SAV Rule: The rule that indicates the validity of a specific source IP address or source IP prefix.

SAV Table: The table or data structure that implements the SAV rules and is used for source address validation in the data plane. 

SAV-related Information: Any information that is useful for getting or generating SAV rules. 

SAV-specific Information: The information specialized for SAV that explicitly or implicitly indicates the accurate incoming directions of source addresses. 

SAV Information Base: A table or data structure for storing SAV-related information. 

False Positive: The validation results that the packets with legitimate source IP addresses are considered "invalid" due to inaccurate SAV rules. 

False Negative: The validation results that the packets with spoofed source IP addresses are considered "valid" due to inaccurate SAV rules. 


## Requirements Language

{::boilerplate bcp14-tagged}

# Design Goals
The intra-domain SAVNET architecture is to enhance the intra-domain SAV and aims to achieve the following goals:

- Automatic Update. The routers after initial configurations can adapt to dynamic routing changes automatically, so that the operational overhead can be controlled.

- Accurate Validation. The real incoming interfaces of source prefixes need to be completely learned, and improper block can be avoided. By trying to exclude non-real incoming interfaces from the valid interface group, improper permit can be reduced. 

- Working in Incremental/Partial Deployment. The architecture should be no worse than existing mechanisms under incremental/partial deployment. 

Manual configuration should be relatively infrequently used for avoiding high operational overhead. Existing SAV mechanisms mostly rely on routing information for automatically generating SAV rules. However, route decices the outgoing interface of source address while SAV focuses on the incoming interface. Therefore, in some cases, routing information cannot support accurate rule generation. The information specifically useful to SAV but may not useful for routing is lacked in existing SAV mechanisms. 

To improve existing SAV mechanisms and achieve the above goals, allowing routers to exhange SAV-specific information is necessary, which is the main idea of the architecture. 


# SAV-related Information
SAV-related Information represents any information that is useful for getting or generating SAV rules. There are largely two kinds of SAV-related information. 

## Routing Information
Routing information is used for forwarding rule computation, which mostly stored in RIB/FIB. It is not specialized for SAV but can be used to some extent for SAV. However, existing SAV mechanisms that conduct SAV solely based on routing information (e.g., strict uRPF and loose uRPF) may have high false positive or false negative rates in complex routing scenarios.

## SAV-specific Information
SAV-specific information is specialized for SAV, which explicitly or implicitly indicates the accurate incoming directions of source addresses. For example, it can explicitly determine the source prefixes for a specific subnet or the incoming directions for a specific source prefix. By using SAV-specific information besides routing information, the accuracy of SAV rules can be improved. More specifically, the false positive can be avoided and the false negative can be reduced. Note that, SAV-specific information can also provide SAV rules directly instead of the materials for rule generation. 


# Intra-domain SAVNET Architecture {#sec-arch}
The proposed architecture is shown in {{fig-arch}}. In the architecture, there is a communication channel connecting two entities, i.e., Source Entity and Validation Entity. Source Entity is the information source. It has some SAV-related information that can be useful for Validation Entity to generate SAV rules. The SAV-related information is transmitted through the channel between the two entities. 

An entity can be a router, a server or some other SAV-equipped devices. A device can act as a Source Entity, a Validation Entity, or both of them. 

~~~
    +-------------------+             +-------------------+
    |   Source Entity   |Communication| Validation Entity |
    | +---------------+ |   Channel   | +---------------+ |
    | |  Source       +-----------------+  Validation   | |
    | |  Speaker      | |             | |  Receiver     | |
    | +-------+-------+ |             | +-------+-------+ |
    |         |         |             |         |         |
    |         |         |             |         |         |
    | +-------+-------+ |             | +-------+-------+ |
    | |  SAV-related  | |             | |  SAV          | |
    | |  Information  | |             | |  Agent        | |
    | +---------------+ |             | +---------------+ |
    +-------------------+             +-------------------+
~~~
{: #fig-arch title="The intra-domain SAVNET architecture"}

## Source Speaker and Validation Receiver
As shown in {{fig-arch}}, Source Speaker resides in Source Entity, and Validation Receiver is in Validation Entity. Either Source Speaker or Validation Receiver is an abstracted interface which represents a union of multiple protocol speakers/receivers as illustrated in {{fig-speaker}}. These protocol speakers/receivers can advertise/receive the messages carrying SAV-related information. The followings are some kinds of the protocol speakers/receivers: 

- Configuration Speaker/Receiver. The configuration Speaker/Receiver can be the protocol interface of YANG, FlowSpec, or management protocols for SAV. Any SAV-related information can be obtained from configuration interface. 

- Routing Protocol Speaker/Receiver. This kind of speakers/receivers are same as the ones in routing protocols (e.g., OSPF). Routing information is mainly obtained from routing protocol speaker. 

- SAV Protocol Speaker/Receiver. This is a newly defined in the document. Generally, the speaker/receiver can reside in completely new protocols or protocol extensions (e.g., routing protocol extensions) for advertising and receiving SAV-specific information. 

SAV-specific information can be obtained through configuration and SAV protocol. For a validation entity, if complete SAV-specific information is available, all SAV rules can be generated based on only SAV-specific information. Routing information is not needed. If a part of SAV-specific information  can be obtained, the information can be a complementary of routing information for rule generation. 

~~~
    +--------------------+
    | Speaker/Receiver   |
    | +----------------+ |
    | |Configuration   <------------
    | |Speaker/Receiver| |         |
    | +----------------+ |         |
    | +----------------+ |         ---> Interact
    | |Routing Protocol<--------------> with other
    | |Speaker/Receiver| |         ---> speakers
    | +----------------+ |         |
    | +----------------+ |         |
    | |SAV Protocol    <------------
    | |Speaker/Receiver| |
    | +----------------+ |
    +--------------------+
~~~
{: #fig-speaker title="Source speaker or validation receiver"}


## Communication Channel
The communication channel is constructed between the Source Speaker and Validation Receiver. The primary purpose of the channel is for Source Entity to advertise SAV-related information to Validation Entity. In the channel, there can be multiple sessions maintained by the entities belonging to configuration, routing protocol, and/or SAV protocol. The concrete manner of constructing a session depends on the actual protocol speakers/receivers, but the following requirements SHOULD be satisfied: 

- The session can be a long-time session or a temporary one, but it SHOULD provide sufficient assurance of transmission reliability and timeliness, so that Validation Entity can update local rules in time. 

- Authentication can be conducted before session establishment. Authentication is OPTIONAL but the ability of authentication SHOULD be available. 


## SAV Agent
{{fig-sav-agent}} shows SAV Agent. SAV Information Manager in SAV Agent parses the messages received by Validation Receiver. The SAV-related information carried in the messages will be stored in SAV Information Base. The information of Source Speaker, protocol speaker, timestamp, and other useful things will also be recorded together. The recorded information will be disseminated to SAV Rule Generator. SAV rules (e.g., tuples like <prefix, interface set, validity state>) will be generated and stored in SAV Table {{I-D.huang-savnet-sav-table}}. Besides rule generation, SAV Information Base also provides the support of diagnosis. Operators can look up the information in the base for protocol monitoring or troubleshooting. 

~~~
                Messages from
                Validation Receiver
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
{: #fig-sav-agent title="SAV agent"}

It can be noticed that SAV Information Base stores SAV-related information from different Source Entities and different protocol speakers. When SAV Rule Generator generates rules based on the stored information, rule conflicts may appear. For example, for a given prefix, the information from different entities or speakers may indicate a different list of valid incoming interfaces. 

To solve the problem of rule conflicts, each Source Entity or protocol speaker can be set with a priority. So, the SAV-related information from the entity or protocol speaker is also tagged with a priority. In general, since SAV-specific information is specilized for SAV and helps generate more accurate SAV rules than routing information, SAV-specific information has a higher priority than routing information. When rule conflicts exist, the rule based on the information with a higher priority will override that based on the information with a lower priority. If two conflicted rules are based on the information with the same priority, they can be merged to one rule (i.e., taking a union set). When the settings of priority change, the affected information MUST be reprocessed for updating rules. 

Consider that one Validation Entity receives the information about prefix P1 from one Source Entity through two protocol speakers. Based on the information from speaker 1, the rule is <P1, {intf1}, valid>. While based on the information from speaker 2, the rule is <P1, {intf2}, valid>. Next, consider two cases of priority settings. In case 1, speaker 1 has a higher priority than speaker 2. Then, the rule of <P1, {intf1}, valid> will be finally stored in SAV Table. In case 2, two speakers have the same priority. Then, <P1, {intf1, intf2}, valid> is the output rule. 


# Connectivity Models
This section presents some examples of connectivity models of the architecture. 

## Example 1: Multiple Source Entities to One Validation Entity
One Validation Entity can collect SAV-related information from multiple Source Entities as shown in {{fig-connect-model-n2one}}. For each Source Entity, Validation Entity maintains a communication channel for receiving messages. 

~~~
    +-----------------+
    | Source Entity 1 |---------
    +-----------------+         \
                                 \
    +-----------------+           \ +-------------------+
    | Source Entity 2 |-------------| Validation Entity |
    +-----------------+           / +-------------------+
            ...                  /
    +-----------------+         /
    | Source Entity n |---------
    +-----------------+
~~~
{: #fig-connect-model-n2one title="Example 1: Multiple source entities to one validation entity"}

## Example 2: One Source Entity to Multiple Validation Entities
One Source Entity can provide information for multiple Validation Entities as shown in {{fig-connect-model-one2n}}. Source Entity will maintain a communication channel with each Validation Entity. 

By combining Example 1 and Example 2, it can be seen that the connectivity model can also be "multiple Source Entities to multiple Validation Entities". 

~~~
                                  +---------------------+
                        ----------| Validation Entity 1 |
                       /          +---------------------+
                      /
    +---------------+/            +---------------------+
    | Source Entity |-------------| Validation Entity 2 |
    +---------------+\            +---------------------+
                      \                      ...
                       \          +---------------------+
                        ----------| Validation Entity n |
                                  +---------------------+
~~~
{: #fig-connect-model-one2n title="Example 2: One source entity to multiple validation entities"}

## Example 3: One Acting as both Source and Validation Entity
As mentioned previously, a device can be both Source and Validation Entity. In {{fig-connect-model-relay}}, the middle entity is such a device. It can receive information messages from the top Source Entity and can advertise information messages to the bottom Validation Entity. The middle entity can also relay the messages from the top Source Entity to the bottom Validation Entity, which is just like routing information spreading. 

~~~
    +-------------------+
    | Source Entity     |
    +-------------------+
              |
    +-------------------+
    | Validation&Source |
    | Entity            |
    +-------------------+
              |
    +-------------------+
    | Validation Entity |
    +-------------------+
~~~
{: #fig-connect-model-relay title="Example 3: One acting as both source and validation entity"}


# Convergence Considerations
Network topologies may change sometimes and result in the change of SAV-related information. Convergence of the architecture is needed. Source Entity MUST advertise the updates of SAV-related information to Validation Entity in time. Then, Validation Entity MUST update local SAV rules immediately. Even so, there will still be delay of message delivery (sometimes re-transmission delay due to packet loss) and information processing. Therefore, during the convergence process, the SAV rules for checking packets are possibly inaccurate, which may result in severes false positive or too much false negative. 

Existing routing information-based SAV mechanisms like strict uRPF is also faced with convergence problem. Inaccurate validation may appear during the convergence of routing, which is inevitable in practice. However, the proposed architecture involves SAV protocols that are especially for SAV. The convergence process can be slowed down due to the existence of SAV protocols. 

The protocol for implementing the architecture MUST carefully consider the convergence problems, so that normal packet forwarding won't be impacted too much. There are some potential work directions for dealing with convergence problems: 

- Taking full use of routing information. Routing information usually provides most of the SAV-related information for rule generation, and SAV-specific information is relatively a small set of information for supplementing missed information. Therefore, most of SAV rules can be updated during routing convergence if routing information is fully taken for rule generation. 

- Advertising and processing the information first that will probably result in false positive. This is because false positive is more harmful to the network than false negative. It is reasonable to allocate more resources for eliminating false positive first. 


# Partial/Incremental Deployment Considerations
Although an intra-domain network mostly has one administration, partial/incremental deployment may still exist due to phased deployment or the limitations coming from multi-vendor supplement. Under partial/incremental deployment, routing information usually can be easily obtained by Validation Entity. The main challenge is that Validation Entity may not be able to get sufficient SAV-specific information. In this case, SAV rules for some source prefixes are generated solely based on routing information, which cannot ensure the accuracy. False positive problems may appear with inaccurate SAV rules. 

The implementation of Validation Entity is RECOMMENDED to support flexible validation modes such as interface-based prefix allowlist, interface-based prefix blocklist, and prefix-based interface allowlist {{I-D.huang-savnet-sav-table}}. The first two modes are interface-scale, and the last one is device-scale. Under partial/incremental deployment, the device of Validation Entity SHOULD take on the proper validation mode according to the deploying of Source Entities. For example, if Validation Entity is able to get the complete set of legitimate source prefixes arriving at a given interface, interface-based prefix allowlist can be enabled at the given interface, and false positive will not exist. 

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

When implementing the architecture in an extended protocol, the existing security mechanisms of the protocol can be taken. 


# Manageability Considerations
The architecture provides a general design for collecting SAV-related information and generating accurate SAV rules. Protocol-independent mechanisms SHOULD be provided for operating and managing SAV-related configurations. For example, a YANG data model for SAV configuration and operation is necessary for the ease of management. 

SAV may affect the normal forwarding of data packets. The diagnosis approach and necessary logging information SHOULD be provided. SAV Information Base SHOULD store some information that may not be useful for rule generation but is helpful for management.  

Messages carrying SAV-related information come from different protocol speakers. Each corresponding protocol SHOULD have monitoring and troubleshooting mechanisms, which is necessary for efficiently operating the architecture. 


# Privacy Considerations
The advertised SAV-related information mainly indicates the incoming directions of source prefixes. Devices under the architecture will learn more forwarding information of data packets. 

An intra-domain network is mostly operated by a single organization or company, and the advertised information is only used within the network. Therefore, the architecture does not import critical privacy issues in usual cases. 

The architecture makes the forwarding information in the network clearer, which can be helpful for network management such as fault diagnosis and traffic visualization. 


# IANA Considerations

This document has no IANA requirements.

# Acknowledgements

Many thanks to the valuable comments from: Igor Lubashev, Alvaro Retana, Aijun Wang, Joel Halpern, Jared Mauch, Kotikalapudi Sriram, Rüdiger Volk, Jeffrey Haas, Xiangqing Chang, Changwang Lin, etc.

--- back



