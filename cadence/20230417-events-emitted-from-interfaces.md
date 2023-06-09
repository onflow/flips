---
status: proposed
flip: 111
authors: Deniz Mert Edincik (deniz@edincik.com) / Supun Setunga (supun.setunga@dapperlabs.com)
sponsor: Deniz Mert Edincik (deniz@edincik.com) / Supun Setunga (supun.setunga@dapperlabs.com)
updated: 2023-06-09
---

# FLIP 111: Emit events from function conditions

## Objective

The objective of this proposal is to allow emitting events from function pre/post conditions.

## Motivation

Currently, though interfaces can declare events, they act as a type-requirement for the implementations,
and cannot be used to emit events using `emit` statement.
Also, there is no way for the interfaces to guarantee the emission of an event.
Instead, the implementation of the interface needs to re-declare the same event, and then emit events as needed.

The current approach not only put the burden on the implementor to emit the event,
but also can lead to potential issues with events, such as:
- Emission of incorrect events
- Events with improper parameters
- Multiple emissions of the same event
- Omission of necessary events

## User Benefit

Interfaces can ensure the emission of proper events, and can take that burden off the implementation.

## Design Proposal

As per the objectives, the solution proposed in this FLIP is to allow emitting events inside pre/post conditions.
This allows interfaces to guarantee the emission of events, when a function (defined in the interface) is called.

### Example:

Suppose the `FungibleToken` is a contract, and has a `TokenWithdrawal` event declared in it.

With function pre/post conditions being able to emit events, the `withdraw` function would look like below.

```cadence 
pub contract FungibleToken {
   
    pub event TokenWithdrawal(type: Type, amount: UFix64, from: Address?)
   
    pub resource interface Provider{ 
        pub fun withdraw(amount: UFix64): @{FungibleToken.Vault} { 
            post { 
                result.balance == amount: "Withdrawal amount must be the same as the balance of the withdrawn Vault"
                emit TokenWithdrawal(type:type, amount:amount, from:from) 
            } 
        } 
    }
}
```

Here, everytime the `withdraw` is invoked, an `TokenWithdrawal` event will be emitted.
Thus, the interface can ensure the proper event is emitted, and can take that burden off the implementation.

### Drawbacks

The proposed solution doesn't possess any known drawbacks. 

### Alternatives Considered

N/A

### Performance Implications

There will be no impact on performance since this simply move the emission of events from one location to another.

### Dependencies

None

### Engineering Impact

This feature would be relatively simple to implement.

### Best Practices

This introduces a new best practice, allowing interface owners the option to emit their own events.
Additionally, contracts can define their own events that complement the existing ones.

### Compatibility

This solution is backwards compatible.

Existing contracts would still emit their old events in addition to these new events, so any event listeners would not
be affected by these changes except for if they wanted to listen to new contracts that did not emit their own events.

### User Impact

None

## Related Issues

https://github.com/onflow/cadence/issues/2069

## Prior Art

N/A

## Questions and Discussion Topics

- Is it still a requirement to allow interfaces to declare concrete events (not just type requirements),
and being able to use those events in `emit` statements?
