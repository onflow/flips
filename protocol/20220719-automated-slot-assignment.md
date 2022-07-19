---
status: Draft
flip: TBD
title: Automated Slot Assignment
forum: TBD
authors: Paul Gebheim (paul.gebheim@dapperlabs.com)
editor: 
---

# Automated Slot Assignment

## Abstract

Flow would benefit from an automated method for allowing node operators to signal their intention to run a node and to then have their node included in the staking table in a subsequent epoch. The idea of *staking slots* is to build an automated process for including new nodes in the staking table, while managing the max number of nodes per node type. This process can be extended in the future with an auction mechanism for selecting node operators in a fully permissionless way.

## Overview

The changes required to allow automatic staking slot assignment will be within the [FlowIDTableStaking.cdc](https://github.com/onflow/flow-core-contracts/blob/master/contracts/FlowIDTableStaking.cdc#L788) contract. The modifications to this contract will need to support the random selection, from a list of potential node operators, to fill the total number of nodes for a specific type. Today this contract uses an allowlist of node ids in conjunction with the list of actually registered nodes to effect a maximum number of staking slots. After this change, each node type will also keep track of its maximum number of slots.

## Requirements

- Configuration should be done on a per node type basis
    - Each node type must have a configurable number of new slots which are released each week, this number must be changeable by service account. Unfilled slots do not roll over.
    - Each node type must have a configurable *maximum* number of slots. Upon reaching this number of slots the system will cease to release new slots each epoch until such time that the service account increases the maximum number of slots.
    - Each node type must have a list of potential node operators who have filled a staking commitment for that node, and registered.
    - Each node type must have an allow list for node ids which can limit node operators to ones which have been pre-approved.
- Must be able to register a node ID and deposit a minimum stake to be eligible for the next staking slot.
    - Note: **Minimum stake** may be 0
    - If a node type has a non-empty allow list, any registered node id which isn’t in the allowlist will be rejected at node assignment time and their stake returned. This allows operators who want to stake their commitment and then submit a governance proposal to be added to the whitelist a way to signal their intention.
- At the end of each epoch, nodes which have requested a slot and deposited the minimum stake should be assigned to the available staking slots.
    - If `count(pending nodes) <= count(released slots)` then assign all pending nodes to available slots
    - else; using `unsafe_random` select from `pending nodes` to fill the available slots.
        
        

## Example Configurations

### Access Nodes

- Minimum stake: 0
- Maximum slots per epoch: 20
- Maximum total slots: TBD
- Allowlist: empty

### Consensus Nodes

- Minimum stake: 500,000 FLOW
- Maximum slots per epoch: 5
- Maximum slots: current + 5 (i.e. we only wanna add 6 slots)
- Allowlist: will be set with vetted operators who apply to run a consensus node through a governance proposal.

## Discussion and next steps

The structure laid out above creates the basic building blocks for having an autonomous system where the network can automatically select new node operators. This doesn’t solve all problems, of course, and those will need to be addressed in subsequent FLIPs.

Items this proposal doesn’t address:

- Automated Slashing - Under this proposal, nodes will only be removed from the staking list when they are manually slashed by the Flow service account, or when that operator removes their stake. Research into opportunities for automated slashing conditions will come in the future.
- Expansion of the number of nodes in the network per node type. Adding more nodes to the network is still primarily bottlenecked by the specifics of current Flow implementation. The structure proposed allows the maximum counts of nodes to be set by the service account to match the number of nodes feasible supported by the current implementation.
- Staking Auctions - As more node types become byzantine fault-tolerant it will become important that node selection is done in a fully permissionless manner. At that time this proposal can be extended with a proposal for implementing an Auction mechanic, where instead of nodes being randomly assigned to free staking slots, the slots can be auctioned off.

## **Resources**

- [Core Contracts: Flow ID Staking Table](https://github.com/onflow/flow-core-contracts/blob/master/contracts/FlowIDTableStaking.cdc)
