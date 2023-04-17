# Events emitted from interfaces

| Status        | (Proposed / Rejected / Accepted / Implemented)       |
:-------------- |:---------------------------------------------------- | |
**FLIP #**    | [NNN](https://github.com/onflow/flow/pull/NNN) (update when you
have PR #)| | **Author(s)** | Deniz Mert Edincik (deniz@edincik.com)
| | **Sponsor**   |                                                      | |
**Updated**   | 2023-04-17                                           | |

## Objective

The objective of this proposal is to enable interfaces to emit events.

## Motivation

At present, Cadence allows the declaration of events within interfaces, but it
does not permit the emission of events directly from these interfaces. Instead,
it relies on the implementation to emit events as needed.  This approach can
lead to potential issues with contracts, including the emission of incorrect
events, events with improper parameters, multiple emissions of the same event,
or the omission of necessary events. 

By viewing interfaces as a set of governing rules for implementation,
incorporating event restrictions could serve as an effective solution to these
challenges.

## User Benefit

This will establish a more secure environment for the user, ensuring that the
emitted event is a direct outcome of an action specified within the interface.

## Design Proposal

For instance, a sample extract from the FungibleToken contract can be currently
represented as:


```cadence 

pub contract FungibleToken {
   
    pub event TokenWithdrawal(type: Type, amount: UFix64, from: Address?)
   
    pub resource interface Provider{ 
        pub fun withdraw(amount: UFix64): @{FungibleToken.Vault} 
        { 
            post 
            { 
                result.balance == amount: "Withdrawal amount 
                must be the same as the balance of the withdrawn Vault" 
            } 
        } 
    }
 
```

We can enhance our approach by defining a trampoline function, which not only
eliminates the need for altering the cadence but also offers a more efficient
method, as this also allows us to emit various events based on specific conditions
present in the `emitTokenWithdrawal` function.



```cadence 

pub contract FungibleToken {
   
    pub event TokenWithdrawal(type: Type, amount: UFix64, from: Address?)
    
    //Temporary trampoline to emit event 
    access(contract) fun emitTokenWithdrawal(type: Type, amount: UFix64, from: Address?): Bool{ 
         emit TokenWithdrawal(type:type, amount:amount, from:from) 
         return true 
    }

     pub resource interface Provider{ 
         pub fun withdraw(amount: UFix64): @{FungibleToken.Vault} { 
             post { 
                 result.balance == amount: "Withdrawal
                 amount must be the same as the balance of the withdrawn Vault"
                
                FungibleToken.emitTokenWithdrawal(
                    type:result.getType(),
                    amount: result.balance, 
                    from: self.owner?.address) 
                } 
            } 
        } 
    }
}
```


### Drawbacks

I don't see any drawback currently. 


### Alternatives Considered

Alternative solution would be allowing emitting events from post conditions such
as:

```cadence 

pub contract FungibleToken {
   
    pub event TokenWithdrawal(type: Type, amount: UFix64, from: Address?)
   
    pub resource interface Provider{ 
        pub fun withdraw(amount: UFix64): @{FungibleToken.Vault} { 
            post { 
                result.balance == amount: "Withdrawal amount
                        must be the same as the balance of the withdrawn Vault" 
                emit TokenWithdrawal(type:type, amount:amount, from:from) 
            } 
        } 
    }
 
```


### Performance Implications

There will be no impact on performance since we are simply relocating the
emission of events from one location to another.


### Dependencies

I have attempted to present a solution that is as compatible with existing
systems as possible. Naturally, this would necessitate updates to documentation
and tutorials.

### Engineering Impact

The Flow team should be responsible for maintaining this feature and
incorporating it into standard contracts.

### Best Practices

This introduces a new best practice, allowing interface owners the option to
emit their own events. Additionally, contracts can define their own events that
complement the existing ones.

### Compatibility

This solution should be entirely backwards compatible.

### User Impact


## Related Issues

https://github.com/onflow/cadence/issues/2069

## Prior Art


## Questions and Discussion Topics

