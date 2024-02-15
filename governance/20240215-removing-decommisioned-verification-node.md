---
status: proposed
flip: GOV-247
authors: Vishal Changrani
sponsors: Vishal Changrani
updated: 2024-02-15
---

# FLIP GOV-247: Removing Verification node a3fac321587b4835966946cf7a13d5feaeb7ca91a84c85c2d2f861c2d80513ec


### Proposal
The verification node with node ID `a3fac321587b4835966946cf7a13d5feaeb7ca91a84c85c2d2f861c2d80513ec` is no longer running.
The node was staked through Coinbase. However the customer contract has ended with Coinbase.
The customer needs to unstake the node but is unreachable as per Coinbase.
The proposal is to remove the node from the approved list of node IDs via a transaction submitted by the service committee.
This will remove the node from the network in the next epoch.

