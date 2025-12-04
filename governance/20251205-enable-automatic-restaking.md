---
status: Proposed
flip: GOV-30
authors: Joshua Hannan (joshua.hannan@flowfoundation.org)
editors:
---

# FLIP GOV-30: Enable Automatic Restaking of Staking Rewards

## Abstract

Enable core contract feature to automatically restake rewarded tokens for nodes and delegators each epoch instead of requiring stakers to do it manually.

## Background

Rewards to node operators and delegators are paid at the end of every epoch to each staker's reward bucket. Currently, these rewards will sit there until the user manually does something with them, either withdrawing or restaking them. This can be cumbersome to most stakers who simply want to continue staking their tokens every week. Many users over the years have asked for functionality to automatically restake their rewards for them.

Flow originally decided to not include this feature out of an abundance of caution about crypto regulations. Given recent developments in the past five years related to crypto regulations, and especially considering that most other Proof-of-Stake blockchains support this feature with no negative repercussions, the Flow foundation is proposing enabling this feature following community discussion and voting.

## Proposal

When tokens are moved between buckets at the end of every epoch, for any given staker, transfer all their rewarded tokens to their staked bucket and update all the relevant staking metadata for that user.

Because rewards are paid every epoch after tokens are moved between buckets, this means that staking rewards that are paid to users will still stay in their rewards bucket after they are paid every week. This will allow users who want to withdraw rewards every week instead of restaking them to still be able to do without having to wait for the 1-2 week unstaking period. We believe that this will be the least disruptive to any existing stakers who maybe doing something different than restaking every week.

In addition to being a convenience for users, this will also ensure that there isn't unstaked FLOW just sitting in rewards buckets and will increase the total amount of FLOW staked overall, further securing the network.

As part of the discussion of this FLIP, we will be sharing it with all the node operators in the network and posting in relevant social channels for other Flow users to see. We will get feedback from all affected parties to ensure that nobody is negatively affected by this change.

If the FLIP is approved, we propose a staged rollout for automatic restaking on Flow testnet and mainnet as follows:

| Date | Action | Status |
|-|-|-|
| Thursday, December 4th, 2025 | Publish FLIP and begin community discussion | :hourglass_flowing_sand: Pending |
| Wednesday, January 21st, 2026 | Assuming No pushback from community, mark FLIP as approved | :hourglass_flowing_sand: Pending |
| Wednesday, January 28th, 2026 | Enable automatic restaking on testnet | :hourglass_flowing_sand: Pending |
| Wednesday, February 4th, 2026 | Enable automatic restaking on mainnet |:hourglass_flowing_sand: Pending|

## Resources
- 
