---
status: Implemented
flip: FLIP-
authors: Joshua Hannan (joshua.hannan@dapperlabs.com)
sponsor: Joshua Hannan (joshua.hannan@dapperlabs.com) 
updated: 2023-04-03
---

# Access Node Staking Minimum

## Objective

Introduce a flag in Cadence that is only allowed to be used by the service account
that disables event emission while it is enabled.

## Motivation

Since there are so many delegators in the identity table, epoch transition operations
such as moving tokens between buckets and paying rewards are costly to the network.

Currently, there are hundreds of node operators and over 16k delegators in the network, which means that nearly
50k events are emitted for each delegator when their rewards are paid because each delegator
gets a `FlowToken.TokensWithdrawn`, `FlowToken.TokensDeposited`, and `FlowIDTableStaking.DelegatorRewardsPaid`
event. Additionally, events from the move tokens transaction and end staking transaction also
emit many `FlowToken` events. These events are completely meaningless because they
are only moving tokens between vaults in the `FlowIDTableStaking` account,
so the `owner` field always has the same value.

These events cost gas and time and also make querying the result of the block difficult
because there are so many of them.

In pay rewards specifically Over 34k of those are the meaningless `FlowToken` events.

You can find these amounts by using this command on the command line:

```
curl -s https://rest-mainnet.onflow.org/v1/transaction_results/84eca4ff612ef70047d60510710cca872c8a17c1bd9f63686e74852b6382cc84 \
| jq '.events | map(select( .type == "A.1654653399040a61.FlowToken.TokensDeposited" )) | map(.payload | @base64d | fromjson) | map(select((.value.fields[0].value.value | tonumber) > 0.0)) | length'
```

You'll need to multiply the result by two to get the total number of events for each
delegator above the specified rewards amount.

This FLIP proposes adding a flag to Cadence that is only usable by the service account
that turns off event emissions while it is enabled. This would allow the staking
contract to avoid emitting the meaningless `FlowToken` events.

## User Benefit

This change would benefit the network because the epoch transition operations would complete faster
and the event payload would be much smaller and less cluttered with meaningless events.

## Design Proposal

Propose a pragma for cadence that can be enable by the staking contract every time
a transfer between vaults happens within the staking contract.

Add code in the staking contract to enable/disable this for transfers between vaults.

### Drawbacks

- This gives the service account a privilege that regular developers do not have,
which some believe is unfair.

- This doesn't only slightly changes the time complexity of the rewards operation.
If n is the number of nodes and delegators in the network, this change will reduce
it from O(3n) to O(n), which is still just O(n) either way,
but a reduction by a third is still somewhat meaningful, especially in combination
with the other solutions that are being worked on.

- There may be some apps that still rely on these `FlowToken` events to monitor
the balance of the staking account. This would break those monitors, but they could
easily update to just query the balance directly whenever they need to.

### Alternatives Considered

Other work being done to improve the efficiency of the protocol smart contracts
is being worked on.

These include:
* Reducing the size of events in the protocol (will be live later in 2023)
* Setting a minimum stake for delegators (https://github.com/onflow/flips/pull/78)
* Breaking up reward payments into batches to reduce event amount per transaction. (currently being explored)

### Performance Implications

This would improve the performance of event listeners and the network as a whole.

### Dependencies

This change will affect anyone who is monitoring the `FlowToken`
events emitted by the epoch contracts.

### Engineering Impact

Still needs to be defined

### User Impact

No user impact

## Related Issues

https://github.com/onflow/flips/pull/78

## Questions and Discussion Topics

What would be required for implementing this in the Cadence language?

Is it reasonable for the service account to be able to do this?
