---
status: proposed
flip: 94
authors: Supun Setunga (supun.setunga@dapperlabs.com)
sponsor: Supun Setunga (supun.setunga@dapperlabs.com)
updated: 2023-05-31
---

# FLIP 94: Entitlements Improvements

## Objective

The main objective of this proposal is to improve entitlement mappings and their use-cases by:
- Allowing use of [entitlement mappings](https://github.com/onflow/flips/blob/main/cadence/20221214-auth-remodel.md#entitlement-mapping-and-nested-values)
  for composite fields.
- Introducing an "Identity mapping", which is a shorthand syntax for mapping a given entitlement to the same entitlement.

## Motivation

### Motivation I: Cannot use entitlement mappings for fields

With the [FLIP for changing member access semantics](https://github.com/onflow/flips/pull/89),
fields of a composite value would return a non-auth reference when accessed via a reference to the composite value.
However, currently, entitlement mappings can only be used for functions, but not for fields.
As such, there's no way to specify to return an auth-reference for fields through member-access syntax.
Users would have to write delegator functions to achieve the same, which adds unnecessary boilerplate code.

### Motivation II: Mapping entitlements to themselves is cumbersome

There can be use-cases where entitlements are needed to be delegated down to nested objects.
In other words, entitlement mapping may need to map an arbitrary entitlement `E` to the same entitlement `E`.
e.g:

```cadence
entitlement mapping M {
    Mutable -> Mutable
    Insertable -> Insertable
    Removable -> Removable
}
```

However, currently it is a bit cumbersome to write such a mapping, and also, it would require knowing all possible
entitlements beforehand.

## User Benefit

Provides a simple and shorthand syntax to achieve above-mentioned use-cases and thus, prevents developers from writing
a lot of boilerplate code.

## Design Proposal

### Allow entitlement mappings on fields

The proposal is to allow entitlement mappings on composite fields.
For example, below code would now be valid.

```cadence
entitlement mapping M {
    Mutable -> Insertable
}

pub resource Collection {
    access(M) var ownedNFTs: @{UInt64: NFT}
}
```

Assume an `auth{Mutable}` reference to the collection `mutableCollectionRef` is available.

```cadence
var mutableCollectionRef: auth{Mutable} &Collection = ...

```

Because `ownedNFTs` has an entitlement mapping access (i.e. `access(M)`), `mutableCollectionRef.ownedNFTs` would now
return a reference of type `auth{Insertable} &{UInt64: NFT}`. i.e:

```cadence
var ownedNFTsRef: auth{Insertable} &{UInt64: NFT} = mutableCollectionRef.ownedNFTs
```

This will allow inserting values to the `ownedNFTs` fields, via `mutableCollectionRef.ownedNFTs`.

```cadence
mutableCollectionRef.ownedNFTs[1234] = nft    // OK
```

This massively reduces the boilerplate could a developer would have to if they are to achieve the same by writing
a function.

### Syntax for identity mapping

The idea of this change is to introduce a syntax to represent "identity mapping", which denotes mapping of an arbitrary 
entitlement `E` to the same entitlement `E`, without having to explicitly specify what the source (domain) and the
target (range) of the mapping are. This saves the trouble of developers having to map each and every such entitlement
explicitly.

The proposed syntax is:

```cadence
entitlement mapping M {
    include Identity
}
```

Here, `Identity` is a built-in entitlement mapping which maps any arbitrary entitlement to itself.
The `include` keyword includes all entries from a given mapping (`Identity` in this case).

Thus, above mapping is equivalent to, but not limited to:

```cadence
entitlement mapping M {
    Mutable -> Mutable
    Insertable -> Insertable
    E1 -> E1
    E2 -> E2
}
```

Apart from the syntax being very compact, another advantage of the proposed syntax is, developers wouldn't need to know
or explicitly specify all the possible entitlements that needs to be included in the mapping, beforehand.

Identity mapping can be combined with other mapping entries. e.g:

```cadence
entitlement mapping M {
    include Identity

    Mutable -> Insertable
    Mutable -> Removable
}
```

### Drawbacks

This is an improvement to the existing feature.
There aren't any known drawbacks at this point of time.

### Alternatives Considered

N/A

### Performance Implications

N/A

### Best Practices

N/A


### Compatibility

This is a feature addition to the existing entitlements feature.

### User Impact

This is a feature addition to the existing entitlements feature.

## Related Issues

N/A

## Prior Art

N/A

## Questions and Discussion Topics

N/A
