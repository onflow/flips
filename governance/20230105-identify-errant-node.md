# Title of FLIP

| Status        | WIP                                            |
:-------------- |:---------------------------------------------------- |
| **FLIP #**    | [NNN](https://github.com/onflow/flow/pull/NNN) (update when you have PR #)|
| **Author(s)** | James Lofgren (james.lofgren@coinbase.com)           | 
| **Sponsor**   | James Lofgren (james.lofgren@coinbase.com)             |
| **Updated**   | 2023-01-05                                           |

## Objective

Define an off-chain process to alert node operators of potentially malicious node behavior at the networking layer.

## Motivation

The launch of permissionless Access nodes is an important step in the continued decentralization of Flow, but creates an opportunity for byzantine nodes on the staking identity table to send malicious traffic to staked nodes.

This FLIP is focused on how to inform node operators of potentially errant behavior at the network layer of registered nodes, while achieving the following goals:  

1. Preserve the neutrality of node operators
2. Provide clarity on channels of communication
3. Define expectations of the service account admin resource
4. Define a process that is resistant to an ongoing attack vector

Currently, during the registration for all node types, the network address must be defined and is publicly shared to facilitate network communication. As this information is known, the potential exists for any node to be subjected to general malicious behavior such as DDoS attacks from any IP. This concern is mitigated through best practices based on the infrastructure setup of individual nodes.

The registration of nodes to the identity table allows node software to determine how incoming traffic should be handled by other nodes, with two possible scenarios:

The sender is not registered on the identity table, and no further processing will take place.
The sender is registered on the identity table, and the node software will attempt to process network traffic through the application layer, performing tasks such as confirming proper tx construction and signature verification.

Because network traffic from a registered node is granted further access down the application layer, errant behavior may cause performance degradation for all users or potential liveness failures.

Depending on the severity and responsiveness of the errant node, separate adjudication by the community and action from the service account may be required.

It should be clearly noted that defining or addressing potentially malicious behavior at the application layer, such as spamming transactions with invalid signatures or that lead to invalid state transitions, is out of scope. The proposed changes in FLIP-660 would better serve to mitigate those concerns.

## User Benefit

By providing this framework, inadvertent issues can be communicated to network participants for resolution, or malicious behavior can be identified then adjudicated through a separate governance process. This allows clear communication between the community to handle potential liveness issues if they occur.

## Design Proposal

The primary purpose of this design is to inform the community of errant network traffic that may be causing performance degradation or liveness issues. 

It is very possible that network traffic that might be considered spam is actually a result of a misconfigured node, or a bug in the node software, rather than intentionally malicious behavior. As such, it is important to clarify that this process does not intend to assign intent, and instead focuses on presenting the community with relevant information.

Definition of errant networking traffic: Unusually large amounts of network traffic that is unrelated to expected application layer data. 

This excludes improperly formed transactions or transactions that lead to an invalid state transition, as these may be economically rational in a permissionless system, and should be addressed through other means, such as variable inclusion fees.

In the instance where a registered node is sending errant network traffic to other nodes the following steps may be taken:

Individual node operators can take action to protect network liveness through standard network monitoring and infrastructure best practices. This may include blocking specific network addresses through firewall rules, rate-limiting, etc. until the issue can be resolved.
Node Operators can attempt to contact the errant node through any available channels and communicate the concern to be resolved. 
In the case of an unresponsive node operator, a governance FLIP can be created by a node operator, providing information such as:
The Network Address of the Reporting Node
The Network Address of the Errant Node
If feasible, provide evidence of the errant network traffic
Attempts at resolution with the errant node.
Based on this information, the administrator of the service account should create a proposal where the community can decide if the errant node should be removed from the identity table.

Only the service account has the authority to remove nodes from the identity table. If the service account removes a node from the identity table, it will not take effect until the next epoch, which could be up to 7 days. After the removal is complete, other nodes in the network will automatically stop attempting to process traffic from the removed node in the application layer.


### Drawbacks

Why should this *not* be done? What negative impact does it have?

Opens the opportunity for censorship of valid traffic or griefing of network participants.

Distinguishing spam from economically rational actions assumes intent and threatens the credible neutrality of the network. (example here).

The launch of permissionless deployment of smart contracts increases the complexity of distinguishing spam, as a bad contract design may encourage behavior that is unexpected. 

NFT drops, Maximum Extractable Value (MEV), 

And inclusion fees should be dynamic and include surge pricing to mitigate this concern.

This proposal does not resolve the underlying concern that dynamic inclusion fees are meant to address.

### Alternatives Considered

* Make sure to discuss the relative merits of alternatives to your proposal.

Node operators continue to individually address network spamming.
As the majority of the network is currently permissioned, if an issue did occur the identity may be known, and it is possible that the network spam is unintentionally being sent.
Blocking network traffic outright would need to be some firewall rule based on the infrastructure setup.
Potential rate limiting
As the majority of the network is currently permissioned, if an issue did occur the identity may be known. 
Use only application layer disincentives to effectively address potential liveness attacks
There would be no way to levy inclusion fees at the collection or consensus level because execution must occur to change state

How far down the stack can you get before encountering inclusion fees? At least until the execution node.

Changing IP address, requires submitting a tx and does not take place until the next epoch.This will take time, and it’s better to be prepared in case an issue does occur.

An on-chain alerting mechanism.
The ability to use an on-chain reporting tool could be restricted by the same attack it is trying to report

Network peering on other protocols is limited to a subset of the overall network.
Not currently enough nodes to 

### Note

As an off-chain process, this proposal has removed the following sections: Dependencies, Engineering Impact, Compatibility, and Best Practices.

### Performance Implications

The proposal would be an off-chain process, and would not have an impact on the performance of the network. However, as the proposal focuses on improving network performance in the case of errant traffic, the network will be better prepared to address this concern.

### Tutorials and Examples

Example of FLIP for errant node:


### User Impact

Prior to permissionless Access Nodes, the identity of node providers was always known to the Flow team. Having a trusted set greatly mitigated the concerns of errant nodes, and if there were an issue this could be relayed directly to the Flow team for follow-up with other node operators.

With the launch of permissionless Access nodes, the node operator is neither known to the Flow team nor to any other node operator. Over time, the expectation of having known node operators will be reduced, and other forms of communication will be required.

## Related Issues

What related issues do you consider out of scope for this proposal, 
but could be addressed independently in the future?

## Prior Art

Does the proposed idea/feature exist in other systems and 
what experience has their community had?

This section is intended to encourage you as an author to think about the 
lessons learned from other projects and provide readers of the proposal 
with a fuller picture.

It's fine if there is no prior art; your ideas are interesting regardless of 
whether or not they are based on existing work.

https://twitter.com/SolanaStatus/status/1520508697100926977 
https://twitter.com/SolanaStatus/status/1520554250803236867 
https://twitter.com/SolanaStatus/status/1520579688648876042

Solana Stake-Weighted Quality of Service: https://github.com/solana-labs/solana/pull/25406 


## Questions and Discussion Topics

This proposal is meant to serve as a starting point for community discussions on how potential liveness issues can be identified and addressed at the networking layer. We strongly encourage feedback and 

Reference Links
Permissionless Access Nodes
Static inclusion fee 
Flow Fees
Identity Table
FLIP-660
Node software confirms node entry into the identity table
Epoch
Function to update network address
Publicly known address
