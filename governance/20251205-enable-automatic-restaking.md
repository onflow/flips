---
status: Proposed
flip: GOV-353
authors: Joshua Hannan (joshua.hannan@flowfoundation.org)
editors:
---

# FLIP GOV-353: Enable Automatic Restaking of Staking Rewards

## Abstract

Enable core contract feature to automatically restake rewarded tokens for nodes and delegators each epoch instead of requiring stakers to do it manually.

## Background

[Flow Epochs and Staking Documentation](https://developers.flow.com/protocol/staking)

Rewards to node operators and delegators are paid at the end of every epoch, which is once a week at the moment. Currently, these rewards will sit there until the user manually does something with them, either withdrawing or restaking them. This can be cumbersome to most stakers who simply want to continue staking their tokens every week and compound their rewards. Many users over the years have asked for functionality to automatically restake their rewards for them.

Flow originally decided to not include this feature out of an abundance of caution about crypto regulations. Given recent developments in the past five years related to crypto regulations, and especially considering that most other Proof-of-Stake blockchains support this feature with no negative repercussions, the Flow foundation is proposing enabling this feature following community discussion and voting.

## Proposal

The Flow staking protocol maintains five buckets of tokens for each staker, listed near the end of [this document](https://developers.flow.com/protocol/staking/epoch-terminology):

* Tokens Committed
* Tokens Staked
* Tokens Unstaking
* Tokens Unstaked
* Tokens Rewarded

At the end of every epoch, committed tokens moved to staked, any unstaking requests move from staked to unstaking, and unstaking moves to unstaked. Rewards are paid after all these movements happen and are deposited into a staker's Rewards bucket.

With this change, during this automatic token movement between buckets at the end of every epoch, for any given staker, the protocol will transfer all the tokens in their rewards bucket to their staked bucket.

Because rewards are paid every epoch after tokens are moved between buckets, this means that each staking rewards payment will still stay in the rewards bucket for one week after they have been paid. This means:

* Reward tokens that the staker's / delegator's does not withdraw from their rewards bucket into their account within one week will automatically be staked. Once the tokens are staked, they are subject to the standard 1-2 week unbonding period. 

* Users can withdraw (a portion of or all of) the rewards immediately within one week after the tokens have been rewarded. This moves the tokens out of the rewards bucket. Only the tokens remaining in the rewards bucket will be automatically restaked at the next epoch switchover. 

We believe that this will be the least disruptive to any existing stakers who may be doing something different than restaking every week.

In addition to being a convenience for users, this will also ensure that there isn't rewarded FLOW just sitting in rewards buckets doing nothing and will increase the total amount of FLOW staked overall, further securing the network.

## Other Options

### Opt-in/Opt-out automatic restaking

One option considered was making this feature opt-in, so users could choose whether they want this. 

Another option could be to make this automatic by default, but allow users to opt-out.

Both of these options would still have most of the benefits described above, but would allow some users that maybe have unique tax considerations to have control over what is done with their rewards. It is also being considered as an option.

One potential problem with allowing users to choose is that Cadence's upgrade restrictions make including this quite cumbersome. We would have to create a separate record from the existing staking records that are accessed separately from the other records, which, with the large number of existing stakers, could significantly affect the performance and cost of existing epoch operations. We would like to avoid this for the protocol and users if possible.

Additionally, users still have a week to withdraw their rewards, they still have the opportunity to "opt-out" every week by withdrawing their rewards if they want.

### Longer Period between Restakes

If one week feels like it is not enough time to give users to decide whether or not to restake or withdraw, it is possible to extend this to any arbitrary number of weeks between restaking. If a month or more is preferable, this could be included and would not affect performance and cost like making the feature opt-in or opt-out would.

### Scheduled Transactions for Restaking

Another option was to introduce [a scheduled transaction handler](https://github.com/onflow/flow-core-contracts/pull/564) that allows users to schedule transactions for themselves to restake their rewards. This also allows users to opt-in, but is unnecessarily complex for something that really should just be a built-in network feature.

## Next Steps

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
