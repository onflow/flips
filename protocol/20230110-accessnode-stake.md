---
status: draft 
flip: NNN (do not set)
authors: Joshua Hannan (joshua.hannan@dapperlabs.com)
sponsor: Joshua Hannan (joshua.hannan@dapperlabs.com) 
updated: 2023-01-10 
---

# Access Node Staking Minimum

## Objective

Introduce a minimum required stake for access nodes
in the Flow protocol and staking smart contract.

## Motivation

Access nodes were proposed to be an easy way to run a Flow node
without having to commit much capital for extended periods of time.
The access node is a staked node that receives and indexes
all protocol state data and serves [the Access API.](https://developers.flow.com/nodes/access-api)

While access nodes aren't directly critical to the core functioning
of the network, they serve an important role of being the point
of access for transaction submission, state querying, and more.
They are also capable of malicious behavior that can negatively affect
users and the network. Until now, access nodes have been staked
in the identity table, but haven't been required to actually stake any FLOW
to participate.

This FLIP proposes increasing the access node minimum to a value
that reflects its importance to the network and prevents certain vulnerabilities
in the smart contract that are a result of having a zero minimum stake requirement.

The Flow community is getting ready to allow anyone to run access nodes
without permission from the consensus committee. As part of these changes,
the staking smart contract will be updated to remove the approval restriction
for access nodes and introduce the concept of a candidate node list,
which refers to nodes who have registered and committed sufficient stake
to be a node operator for the next epoch,
but aren't currently in the identity table.

The protocol uses this list to randomly select nodes to be node operators,
and there is a limit to the number of nodes who can be candidate nodes for each role.
Because access nodes' minimum stake requirement is currently zero,
an attacker can register any number of nodes until the limit is reached,
effectively blocking other users from registering access nodes
until the next epoch when the candidate list is reset,
at which point they could just do it again.

To attempt to address this vulnerability while also keeping access nodes relatively cheap,
this FLIP proposes that the access node minimum stake requirement is set to .
This requirement will be treated the same as the minimum stake requirements
for other node types, meaning that access nodes will be removed if their committed
stake is less than the minimum.

**This proposal does not enable access nodes to receive rewards**.
While access nodes will have to stake FLOW, they will still not receive any rewards.

## User Benefit

This change would not have any direct benefit to users,
but would prevent attackers from blocking access to node registrations
by exploiting the vulnerability.

## Design Proposal

Remove any restrictions in the contract that do not count access nodes
as staked or prevent them from staking, such as in
`getTotalStaked`](https://github.com/onflow/flow-core-contracts/blob/master/contracts/FlowIDTableStaking.cdc#L1406)

The service account committee runs a transaction, 
[`set_minimums.cdc`](https://github.com/onflow/flow-core-contracts/blob/master/transactions/idTableStaking/admin/change_minimums.cdc)
to set the access node minimum to the desired value.

### Drawbacks

This makes running an access node slightly more expensive.

### Alternatives Considered

* Keeping the stake requirement at zero, but requiring a deposit at registration time
that is returned to the node operator at the end of the epoch.
This was discarded because it is only a temporary solution that adds
more work but doesn't have much benefit in the short term.

### Performance Implications

There should be no performance changes with this update.

### Dependencies

This change will not affect any standard transactions or scripts.
Users who are registering access nodes will need to be made aware that 
they must commit tokens for their node now.

### Engineering Impact

This is a small change that can be easily tested and maintained. 

### User Impact

* This will be rolled out with an upgrade to the staking contract
and a transaction to set the staking minimum for access nodes.

## Related Issues

This proposal is related to the automated slot selection proposal
and its related PRs:
* https://github.com/onflow/flow-core-contracts/pull/321
* https://github.com/onflow/flow-core-contracts/pull/324
* https://github.com/onflow/flow-core-contracts/pull/328

That proposal contains much bigger changes to how nodes are selected
for each epoch as well as some code refactors to improve performance
of the epoch smart contracts.

## Questions and Discussion Topics

Need to define the values for slot limits, candidate node limits,
and the access node staking minimum amount.
