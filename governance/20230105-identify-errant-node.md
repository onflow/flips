# Identify Errant Access Nodes

| Status        | WIP                                            |
:-------------- |:---------------------------------------------------- |
| **FLIP #**    | [NNN](https://github.com/onflow/flow/pull/NNN) (update when you have PR #)|
| **Author(s)** | James Lofgren (james.lofgren@coinbase.com)           | 
| **Sponsor**   | Kshitij Chaudhary (kshitij.chaudhary@dapperlabs.com); Vishal Changrani (vishal.changrani@dapperlabs.com) |
| **Updated**   | 2023-01-31                                           |

# **Objective**

This FLIP defines an off-chain process for node operators to report on potentially malicious access node behavior at the networking layer and prescribes the process of removal of such errant nodes.

# **Background**

Flow is introducing permissionless Access nodes (ANs), allowing anyone to run a stake access node and an API to interact with the chain directly. By running an AN, the operator only needs to trust Execution Nodes (ENs) for the execution state without relying on any other third-party for information, thus getting priority lower-latency access to data.

In the current setup, a separate “allow list” is maintained as a second layer of the onboarding process. Only nodes that are both staked and approved by the service account, i.e. registered on the “allow list”, can operate a node. In a permissionless system, as described in [FLIP 1057](https://github.com/onflow/flips/blob/main/protocol/20220719-automated-slot-assignment.md), Flow would open up AN registration slots, enabling operators to demonstrate their intention to run an AN by simply acquiring a slot. As the number of slots in the registration list is limited, to ensure that attackers don’t acquire all slots and bar others from running a node, a nominal minimum stake (100 Flow) will be required from an interested operator to legitimately acquire a slot.

At the end of the registration cycle, once interested operators have acquired the slots and submitted (*at least*) the minimum required stake, some operators are randomly selected from the list, allowing them to run an AN in the subsequent epoch. This process can be enhanced in the future with an auction process for selecting node operators in a fully permissionless manner. 

The number of nodes selected in each epoch will be in conjunction with the maximum number of permissionless ANs that are allowed to operate in the network at a time, which is initially fixed at 5. This is in addition to the 122 permissioned access nodes already in the system.

**Access Node Configurations**

- Minimum stake: 100 Flow
- Maximum permissionless AN slots per epoch: 5 (revisable)
- Maximum total slots in FlowIDTable: Limited
- Allow list: Not Applicable

# **Motivation**

The launch of permissionless ANs is an important step in the continued decentralization of Flow, but also creates an opportunity for byzantine nodes on the [staking identity table](https://github.com/onflow/flow-core-contracts/blob/master/contracts/FlowIDTableStaking.cdc) to send malicious traffic to other staked nodes. Given the Flow nodes are on the public network, they are susceptible to traditional types of attack like denial of service (DDoS) from any IP. This concern is mitigated through best practices based on the infrastructure setup of individual nodes. In a permissionless AN system however, a malicious staked node could impact network performance and stability through seemingly legitimate communication with peers and non-sensical data, circumventing traditional network controls.

The node software can be viewed at a very high-level as divided into two parts - the networking layer which handles the peer-to-peer communication and the application layer which handles the node specific business logic. For consensus nodes for instance, the application layer would entail proposing blocks, voting on blocks, sealing blocks etc. Protection already exists at the networking layer to reject traffic from any node that is not on the identity list (non-registered node). Hence, if a sender is not registered in the identity table, all messages from it will be dropped by the other nodes in the network.

![Untitled document.jpg](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/42642094-38ef-42aa-8d33-c5d3ad29f071/Untitled_document.jpg)

However, if a permissionlessly onboarded AN spams other nodes with unnecessary messages, it could waste resources (CPU, network, memory etc.) of other staked nodes in the system. The application layer is robust enough to reject malicious traffic from any node (registered or non-registered) such that the security of the chain is not affected. In such a scenario however, network liveness could be impacted.

**There are two sets of mechanism that the operators can take up if they observe malicious traffic** –

1. At the operator infrastructure level, operators could block the errant AN, and
2. At the governance level, an operator could flag the errant AN and report it to the community and the service account administrators, leading to the possibility of the errant node’s ejection from the network.

# **Principles**

First, this FLIP focuses on proposing how operators could communicate about errant behavior of ANs. Additionally, it lays a proposal for ejection of errant nodes, contending that nodes will only be removed from the staking list when they are manually slashed by the service account, or when the operator removes its stake. Automated slashing will be explored in the future.

Second, it is possible that network traffic that might be considered “spam” is a result of a misconfigured node or a bug in the node software, rather than intentionally malicious behavior. The process proposed in this FLIP however does not intend to assign intent, and instead focuses on presenting the community with the relevant information.

Third, “errant networking traffic” will be defined as unusually large amounts of network traffic that is unrelated to expected application layer behavior. This excludes improperly formed transactions or transactions that lead to an invalid state transition, as these may be economically rational in a permissionless system, but should be addressed through other means such as dynamic inclusion fees.

Fourth, depending on the severity and responsiveness of the errant node, separate adjudication by the community and action from the service account may be required. Defining or addressing potentially malicious behavior at the application layer, such as spam transactions with invalid signatures or that lead to invalid state transitions, is out of scope of this FLIP.

Fifth, besides providing clarity on channels of communication, this FLIP ensures that neutrality of node operators is preserved while reporting on an errant node and lays out the expectations of the service account admin resource after the whistle is blown by an operator.

# **Process Proposal**

A mature long-term solution needs to be built in which the protocol itself is able to identify and kick-out errant nodes without off-chain human communication or interaction. In view of the massive engineering effort behind these protocol-level upgrades, in the medium-term, a high level of automation with community operators identifying, reporting and voting on-chain to remove an errant access node should be considered, without any involvement of the service account administrator. This can become a reality once the capability to generate cryptographic evidence of malicious traffic is built, along with a robust on-chain token-weighted voting process for community members to participate in.

In the short-term, a pragmatic, cost-effective, community-observed and community-driven process is being proposed. When a node operator observes an AN sending errant network traffic to other nodes, the following steps may be taken.

1. Individual node operators can protect network liveness through standard network monitoring and infrastructure best practices such as blocking messages, firewall rules, and rate-limiting, until the issue can be resolved.
2. Next, node operators can attempt to contact the errant node through any available channels and communicate the concern to be resolved.
3. In the case of an unresponsive node, a “FLIP-light” can be created by a node operator. While a FLIP involves an elaborate process, a FLIP-light (proposed *template below*) would simply entail an operator reporting on the errant node to other community members. A Discord Channel will be created specifically to facilitate this communication within the community, and node operators will be expected and encouraged to upvote or downvote the reports.
4. Based on the information gathered and community-consensus, the service account administrators will be able to decide if the errant node should be removed from the identity table. This is to prevent the currently limited number of AN slots available from being occupied by errant nodes. The ejection process will take a multi-signatory format at the end of each epoch. If a conclusive decision cannot be reached in such a time span, the node can continue to remain registered on the network for one or more epochs, but non-functional as other nodes continue to block traffic from the allegedly errant AN.

Note that in the short-term, only the service account has the authority to remove nodes from the identity table. If the service account removes a node from the identity table, it will not take effect until the next epoch. After the removal is complete, other nodes in the network will automatically stop accepting messages or connections from the removed node. The removal also creates space and opportunity for new ANs to join the identity table.

# **Template**

A “whistle-blower” must provide the following information when reporting on an errant AN, for community members to conclusively upvote, downvote, and share their feedback.

1. The Network Identity (node ID, networking address, networking public key, staking public key) of the Reporting Node (the “whistleblower”)
2. The Network Identity of the Errant Node
3. Evidence (logs) of the errant network traffic along with the source of information
4. Attempts at resolution with the errant node, including proof of the errant node’s non-responsiveness
5. Network Identity of any other nodes who have experienced similar behavior from the errant node, and would be ratifying/ co-sponsoring the FLIP-light under consideration
6. Any other relevant details such as operating experience of the whistleblower on the Flow network, or other observations about the behavior of the errant node

# **User Benefit**

By providing this framework, inadvertent issues can be communicated to network participants for resolution, and malicious behavior can be identified then adjudicated through a separate governance process. This allows clear communication between the community to handle potential liveness issues if and when they occur.

# **Drawbacks**

This process does not propose any bonded fee by staked nodes to report on an errant node. This could create an opportunity for the censorship of valid traffic from honest network participants by other nodes without penalty. This is the primary reason this FLIP focuses on informing the network of errant behavior rather than on enforcing behavior directly. When staked nodes become fully permissionless, a minimum bond amount from whistle-blowers may be considered.

# **Performance Implications**

As the proposal is an off-chain process and focuses on improving network performance in the case of errant traffic, the network will be better prepared to address this concern.

# **User Impact**

Prior to permissionless ANs, the identity of node providers was always known to the Flow team. Having a trusted set of nodes greatly mitigated the concerns of errant nodes, and if there was an issue it could be relayed directly to the Flow team. With the launch of permissionless ANs, the node operators are anonymous, neither known to the Flow team nor to any other node operator. Over time, the expectation of having known node operators will be reduced, and other forms of communication will be required.

# **Related Issues**

[FLIP-1057](https://forum.onflow.org/t/flip-1057-automated-slot-assignment/3447/) explains the onboarding process, stating how permissionless ANs join the identity table when the number of intentions has exceeded the number of slots available. Also, [FLIP 57](https://github.com/onflow/flips/blob/main/protocol/20230110-accessnode-stake.md) proposes increasing the minimum stake requirement for permissionless ANs to 100 FLOW to prevent slot-exhaustion attacks from malicious operators.

# **Questions and Discussion Topics**

This proposal is meant to serve as a starting point for community discussions on how potential liveness issues can be identified and addressed at the networking layer. We strongly encourage feedback from node operators in the Flow community.
