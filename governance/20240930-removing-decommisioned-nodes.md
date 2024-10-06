---
status: Proposed
flip: GOV-291
authors: Vishal Changrani
sponsors: Vishal Changrani
updated: 2024-09-30
---

# FLIP GOV-291: Removing decommissioned nodes

### Proposal

1. The verification node with node ID `1cf032dd5b3cf9a63943e1f04472bb57cc1240d5cdd86c5fde05b4dabf8b1e7a` is no longer running.
The node was staked through BlockDaemon. The customer needs to unstake the node but is unreachable as per BlockDaemon.
The proposal is to remove the node from the approved list of node IDs via a transaction submitted by the service committee.
This will remove the node from the network in the next epoch.

2. The access node with node ID `6b9038227f6774f632f7481a2d4d82d6da5efc5fbceb1756fbfc590112231a5b` is no longer running.
   The node was staked by Metrika who no longer intends to run the node.
   Even though access nodes no longer have to be added to the approved list of nodes, this node was added to the network before the launch of permissionless access nodes and therefore had to be added to the approved list back then.

### [Issue](https://github.com/onflow/flips/issues/291)

### [Forum Post](fourm post: https://forum.flow.com/t/flip-gov-291-remove-two-nodes-from-the-approved-list-of-nodes/6599)
