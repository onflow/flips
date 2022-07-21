---
status: Draft
flip: GOV-2
title: Enable Automatic Rewards Payouts
forum: TBD
authors: Paul Gebheim (paul.gebheim@dapperlabs.com)
editor: 
---

## Abstract

Enable core contract feature to automatically calculate and payout rewards during system transaction at end of each epoch.

## Background

Rewards to node operators and delegators have been paid via a service account initiated transaction before the end of each weekly epoch. The core contracts now support automatic payouts, and the service account is able to enable automatic payout calculations to happen at the end of each epoch. This automatic rewards payout system has been operating without issue on Flow testnet for two weeks.

## Proposal

We propose a staged rollout for automatic reward payouts on Flow mainnet as follows:


| Date | Action | Status |
|-|-|-|
| Wednesday, July 20th | Set [reward rate for automatic rewards](https://github.com/onflow/service-account/pull/152) using $1.05^\frac{1}{52} - 1= 0.00093871$| :white_check_mark: Done |
| Thursday, July 28th | Service account will initiate transaction which calculates and pays out rewards. | :hourglass_flowing_sand: Pending |
| Thursday, August 4th | Service account will initiate transaction which calculates and pays out rewards, and [enable automatic payouts](https://github.com/onflow/flow-core-contracts/blob/master/transactions/epoch/admin/set_automatic_rewards.cdc) in the future |:hourglass_flowing_sand: Pending|
| Wednesday, August 10th | First automatic reward payout will occur during the epoch end |:hourglass_flowing_sand: Pending|

## Resources
- [Transaction: Set Reward Rate](https://github.com/onflow/service-account/pull/152)
- [Transaction: Enable Automatic Rewards](https://github.com/onflow/flow-core-contracts/blob/master/transactions/epoch/admin/set_automatic_rewards.cdc)
- [Core Contracts: Automatic Rewards Calculation Logic](https://github.com/onflow/flow-core-contracts/blob/master/contracts/epochs/FlowEpoch.cdc#L484-L505) 
