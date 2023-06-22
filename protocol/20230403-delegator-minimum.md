---
status: Implemented
flip: FLIP-78
authors: Joshua Hannan (joshua.hannan@dapperlabs.com)
sponsor: Joshua Hannan (joshua.hannan@dapperlabs.com) 
updated: 2023-06-22
---

# Delegator Staking Minimum

## Objective

Introduce a minimum required stake (50 FLOW)
for staking delegators in the Flow protocol and staking smart contracts.

## Motivation

Delegators were proposed as an easy way to participate
in the security of the network and earn staking rewards
without having to run a node or commit much capital for extended periods of time.

While delegators aren't directly critical to the core functioning
of the network, as a group, they provide a signaling function of trusted nodes
and trust in the network generally. Delegators have been staked
in the identity table and choose a specific node to delegate to.
They can delegate any amount of FLOW and will receive rewards proportional to the 
amount of FLOW they delegate.

Unfortunately, the low barriers to entry take a toll on the network. 
Since there are so many delegators in the identity table, epoch transition operations
such as moving tokens between buckets and paying rewards are costly
and time-consuming. The network also has to pay for storage
for each delegator regardless of how much they stake.

Currently, there are over 16k delegators in the network, which means that nearly
50k events are emitted for each delegator when their rewards are paid because each delegator
gets a `FlowToken.TokensWithdrawn`, `FlowToken.TokensDeposited`, and `FlowIDTableStaking.DelegatorRewardsPaid`
event. Over 28k of those are for amounts less than 0.1 FLOW, which is very wasteful for such a small amount.

You can find these amount by using this command on the command line:

```
curl -s https://rest-mainnet.onflow.org/v1/transaction_results/84eca4ff612ef70047d60510710cca872c8a17c1bd9f63686e74852b6382cc84 \
| jq '.events | map(select( .type == "A.1654653399040a61.FlowToken.TokensDeposited" )) | map(.payload | @base64d | fromjson) | map(select((.value.fields[0].value.value | tonumber) > 0.1)) | length'
```

You'll need to multiply the result by three to get the total number of events for each
delegator above the specified rewards amount.

This FLIP proposes increasing the delegator staking minimum to 50 FLOW
to avoid more unnecessary storage usage
and iterations and events for small staking and reward amounts.

The number of staked FLOW proposed for minimum rewards was chosen because it is
a reasonable value to prevent DOS attacks and is less than the minimum stake for access nodes.
It makes creating thousands of empty delegations prohibitively expensive.

It currently will cut down the number of rewards events emitted by almost two-thirds.
It is also still affordable at current FLOW prices.
With current rewards calculations, this means that the minimum amount of rewards
that a delegator can receive in an epoch is 0.1 FLOW.

This value can definitely change in the future if network and economic changes require it.

## User Benefit

This change would benefit everyone using the network because the rewards transaction will
no longer take multiple seconds to execute.

It will also benefit applications who are monitoring the network for events because it will drastically
reduce the number of events in this transaction, making it easier to parse and store.

## Design Proposal

In `FlowIDTableStaking.moveTokens()`, for each delegator, check that their committed balance
is above the minimum. If not, unstake their tokens.

In `FlowIDTableStaking.calculateRewards()`, add a conditional that only 
calculates rewards for a delegator if their `tokensStaked` is greater than 50 FLOW.

The service account committee runs a transaction, 
[`set_minimums.cdc`](https://github.com/onflow/flow-core-contracts/blob/master/transactions/idTableStaking/admin/change_minimums.cdc)
to set the delegator minimum to the desired value.
The key for delegators in the minimums dictionary will be `0`.

### Drawbacks

This makes staking as a delegator slightly more expensive.

### Alternatives Considered

This doesn't fully address the issue of too many iterations and events in the staking
contract, but is an important short term solution
while other more complex changes can be designed and implemented.

These include:
* Reducing the size of events in the protocol (will be live later in 2023)
* removing the unnecessary `FlowToken` events from these transactions completely
  with a cadence flag only usable by the service account to not emit events. (FLIP in progress)
* Breaking up reward payments into batches to reduce event amount per transaction. (currently being explored)

### Performance Implications

This will significantly improve the performance of the epoch rewards payment transaction.

### Dependencies

This change will not affect any standard transactions, scripts, or events.

### Engineering Impact

This is a small change that can be easily tested and maintained. 

### User Impact

* This will be rolled out with an upgrade to the staking contract
and a transaction to set the staking minimum for delegators.

## Related Issues

## Questions and Discussion Topics

Is the staking minimum reasonable for delegators?
