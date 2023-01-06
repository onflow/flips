# Identify Errant Access Nodes

| Status        | WIP                                            |
:-------------- |:---------------------------------------------------- |
| **FLIP #**    | [NNN](https://github.com/onflow/flow/pull/NNN) (update when you have PR #)|
| **Author(s)** | James Lofgren (james.lofgren@coinbase.com)           | 
| **Sponsor**   | James Lofgren (james.lofgren@coinbase.com)             |
| **Updated**   | 2023-01-05                                           |

## Objective

Define an off-chain process to alert node operators of potentially malicious node behavior at the networking layer.

## Motivation

The launch of permissionless Access nodes is an important step in the continued decentralization of Flow, but creates an opportunity for byzantine nodes on the [staking identity table](https://github.com/onflow/flow-core-contracts/blob/master/contracts/FlowIDTableStaking.cdc) to send malicious traffic to staked nodes.

**This FLIP is focused on how to inform node operators of potentially errant behavior at the network layer of registered nodes, while achieving the following goals:**  

1. Preserve the neutrality of node operators
2. Provide clarity on channels of communication
3. Define expectations of the service account admin resource
4. Define a process that is resistant to an ongoing attack vector

Currently, during the registration for all node types, the network address must be defined and is [publicly shared](https://flowscan.org/staking/nodes) to facilitate network communication. As this information is known, the potential exists for any node to be subjected to general malicious behavior such as DDoS attacks from any IP. This concern is mitigated through best practices based on the infrastructure setup of individual nodes.

The registration of nodes to the identity table allows node software to determine how incoming traffic should be handled by other nodes, with two possible scenarios:

1. The sender **is not** registered on the identity table, and no further processing will take place.
2. The sender **is** registered on the identity table, and the node software will attempt to process network traffic through the application layer, performing tasks such as confirming proper tx construction and signature verification.

**Because network traffic from a registered node is granted further access down the application layer, errant behavior may cause performance degradation for all users or potential liveness failures.**

Depending on the severity and responsiveness of the errant node, separate adjudication by the community and action from the service account may be required.

_It should be clearly noted that defining or addressing potentially malicious behavior at the application layer, such as spamming transactions with invalid signatures or that lead to invalid state transitions, is out of scope. The proposed changes in [FLIP-660](https://github.com/onflow/flow/blob/master/flips/20211007-transaction-fees.md) would better serve to mitigate those concerns._

## User Benefit

By providing this framework, inadvertent issues can be communicated to network participants for resolution, or malicious behavior can be identified then adjudicated through a separate governance process. This allows clear communication between the community to handle potential liveness issues if they occur.

## Design Proposal

The primary purpose of this design is to inform the community of errant network traffic that may be causing performance degradation or liveness issues. 

It is very possible that network traffic that might be considered spam is actually a result of a misconfigured node, or a bug in the node software, rather than intentionally malicious behavior. **As such, it is important to clarify that this process does not intend to assign intent, and instead focuses on presenting the community with relevant information.**

**Definition of errant networking traffic:** Unusually large amounts of network traffic that is unrelated to expected application layer data. 

This excludes improperly formed transactions or transactions that lead to an invalid state transition, as these may be economically rational in a permissionless system, and should be addressed through other means, such as variable inclusion fees.

In the instance where a registered node is sending errant network traffic to other nodes the following steps may be taken:

1. Individual node operators can take action to protect network liveness through standard network monitoring and infrastructure best practices. This may include blocking specific network addresses through firewall rules, rate-limiting, etc. until the issue can be resolved.
2. Node Operators can attempt to contact the errant node through any available channels and communicate the concern to be resolved.
3. In the case of an unresponsive node, a governance FLIP can be created by a node operator, providing information such as:
    + The Network Address of the Reporting Node
    + The Network Address of the Errant Node
    + If feasible, provide evidence of the errant network traffic
    + Attempts at resolution with the errant node.
4. Based on this information, the administrator of the service account should create a proposal where the community can decide if the errant node should be removed from the identity table. This is to prevent the currently limited number of Access Node slots available from being occupied by errant nodes. 

**Only the service account has the authority to remove nodes from the identity table.** If the service account removes a node from the identity table, it will not take effect until the next epoch, which could be up to 7 days. After the removal is complete, other nodes in the network will automatically stop attempting to process traffic from the removed node in the application layer. This removal also creates space for a new Access Node to join the identity table if there is only a limited number of slots available. 


### Drawbacks

This process does not propose any bonded fee by staked nodes to report an errant node. This could create an opportunity for the censorship of valid traffic or grieifing of honest network participants by other nodes without penalty. This is the primary reason this FLIP focuses on informing the network of behavior, rather than on enforcing behavior directly.

When staked nodes become fully permissionless, it should be a consideration to require a minimum bond amount when raising an alert on an errant node.  

### Alternatives Considered

1. Node operators continue to individually address network spamming, and no further action is taken.

    **Pros:** 
    + As the majority of the network is currently permissioned, if an issue did occur the identity may be known and addresed directly, without requiring additional steps.

    **Cons:**
    + As the number of permissionless nodes increases, there should not be an expectation of known identities or proper behavior from node operators
    + If there is a [limited number](https://github.com/onflow/flips/blob/main/protocol/20220719-automated-slot-assignment.md) of slots available for Access Nodes, without staking requirements or a process for potential removal, these slots may be taken by operators who have no intention of performing their tasks properly.


### Note

As an off-chain process, this proposal has removed the following sections: Dependencies, Engineering Impact, Compatibility, and Best Practices.

### Performance Implications

The proposal would be an off-chain process, and would not have an impact on the performance of the network. However, as the proposal focuses on improving network performance in the case of errant traffic, the network will be better prepared to address this concern.

### Tutorials and Examples

Example of FLIP for errant node: (coming soon)

### User Impact

Prior to permissionless Access Nodes, the identity of node providers was always known to the Flow team. Having a trusted set greatly mitigated the concerns of errant nodes, and if there were an issue this could be relayed directly to the Flow team for follow-up with other node operators.

With the launch of permissionless Access nodes, the node operator is neither known to the Flow team nor to any other node operator. Over time, the expectation of having known node operators will be reduced, and other forms of communication will be required.

## Related Issues

[FLIP-1057](https://forum.onflow.org/t/flip-1057-automated-slot-assignment/3447/) addressing how Access Nodes join the identity table when the number of intentions have exceeded the number of slots available.  

## Prior Art

**Methods of Communication:**

In general, protocols often rely on public communication channels to coordinate network actions. 
+ [Example](https://twitter.com/SolanaStatus/status/1520508697100926977)

**Protocol Changes:**

In some networks, protocol changes have been implemented to prioritize and filter traffic based on the stake weight of nodes. As Access Nodes do not currently require stake, this method could not be used, but may be an option if Access Nodes were to require some stake in the future. 
+ [Solana Stake-Weighted Quality of Service](https://github.com/solana-labs/solana/pull/25406)


## Questions and Discussion Topics

This proposal is meant to serve as a starting point for community discussions on how potential liveness issues can be identified and addressed at the networking layer. We strongly encourage feedback from the Flow community and other node operators.
