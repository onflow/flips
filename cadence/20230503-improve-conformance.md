---
status: proposed
flip: 83
authors: Supun Setunga (supun.setunga@dapperlabs.com)
sponsor: Supun Setunga (supun.setunga@dapperlabs.com)
updated: 2023-05-03
---

# FLIP 83: Improve interface conformance

## Objective

This FLIP proposes to relax a restriction associated with interface conformance,
by allowing conditions defined in other interfaces to coexist with a default function implementation coming from a 
different interface.

## Motivation

Assume there are two interfaces, one (i.e: `Vault`) provides a default implementation to the `withdraw` function,
and the other (i.e: `ThrottledVault`) provides restrictions on how much can be withdrawn at a time.

Now suppose an implementation need to conform to both of these interfaces.

```cadence
pub resource interface Vault {
    pub fun withdraw(_ amount: Int): @NFT {
        // some default implementation
    }
}

pub resource interface ThrottledVault {
    pub fun withdraw(_ amount: Int): @NFT {
        pre { amount < 50 }
    }
}

pub resource MyVault: Vault, ThrottledVault {  // Reports an error
}
```

Currently, this reports an error saying ``resource `MyVault` has conflicting requirements for function `withdraw` ``.

## User Benefit

It can be useful to have interfaces that only define restrictions, in the form of pre/post conditions.
Relaxing the current restriction allows user to use this pattern along with default functions.

When interface inheritance is supported, the chance of running into similar situations would be more frequent.

## Design Proposal

Here it is proposed to relax the current restriction and allow conditions defined in other interfaces to coexist
with a default function implementation coming from other interfaces.

```cadence
pub resource MyVault: Vault, ThrottledVault {  // This would be valid
    // `MyVault` would get the default implementation from `Vault` and the conditions from `ThrottledVault`
}
```

### Drawbacks

None

### Alternatives Considered

None

### Performance Implications

None

### Dependencies

None

### Engineering Impact

This change is trivial in the implementation.

### Compatibility

This proposal suggests a relaxation of an existing restriction. Hence, no existing codes would break.

### User Impact

None

## Related Issues

None

## Prior Art

None

## Questions and Discussion Topics

None

