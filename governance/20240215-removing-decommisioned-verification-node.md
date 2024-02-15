---
status: proposed
flip: GOV-5
authors: Vishal Changrani
sponsors: Vishal Changrani
updated: 2024-02-15
---

# FLIP GOV-247: Removing Verification node 268572e59ccc469592bb434715ac3c10649d681e6eace6fd11d00d1247ef1fdc

### Proposal
The verification node with node ID `268572e59ccc469592bb434715ac3c10649d681e6eace6fd11d00d1247ef1fdc` is no longer running.
The node was staked through Coinbase. However the customer contract has ended with Coinbase.
The customer needs to unstake the node but is unreachable as per Coinbase.
The proposal is to remove the node from the approved list of node IDs via a transaction submitted by the service committee.
This will remove the node from the network in the next epoch.

