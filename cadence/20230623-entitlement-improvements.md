---
status: implemented
flip: 94
authors: Supun Setunga (supun.setunga@dapperlabs.com)
sponsor: Supun Setunga (supun.setunga@dapperlabs.com)
updated: 2023-06-23
---

# FLIP 94: Entitlements Improvements

## Objective

The main objective of this proposal is to improve entitlement mappings and their use-cases by:
- Allowing use of [entitlement mappings](https://github.com/onflow/flips/blob/main/cadence/20221214-auth-remodel.md#entitlement-mapping-and-nested-values)
  for composite fields.
- Introducing an "Identity mapping", which is a shorthand syntax for mapping a given entitlement to the same entitlement.

## Motivation

### Motivation I: Cannot use entitlement mappings for non-reference fields

With the [FLIP for changing member access semantics](https://github.com/onflow/flips/pull/89),
fields of a composite value would return a non-auth reference when accessed via a reference to the composite value.
However, currently, entitlement mappings can only be used for functions, and reference typed field,
but not to non-reference typed fields.

```cadence
access(M) let refField: auth(M) &T   // OK

access(M) let field: T               // Not allowed
```

As such, there's no way to specify to return an auth-reference for fields through member-access syntax,
if the field is a non-reference.
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

### Bigger Picture

These two changes are also intended on improving the user experience of existing mutability restrictions.
The [Mutability Restrictions Vision](https://github.com/onflow/flips/pull/97) explains how this proposed change
contributes to improving the solution to the existing issue with mutability restrictions,
and how the final solution looks like, with examples.

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

#### Case I: Code only has a reference to the container value

Assume an `auth{Mutable}` reference to the collection `mutableCollectionRef` is available.
Because `ownedNFTs` has an entitlement mapping access (i.e. `access(M)`), `mutableCollectionRef.ownedNFTs` would now
return a reference of type `auth{Insertable} &{UInt64: NFT}`. i.e:

```cadence
var mutableCollectionRef: auth{Mutable} &Collection = ...

var ownedNFTsRef: auth{Insertable} &{UInt64: NFT} = mutableCollectionRef.ownedNFTs
```

This will allow inserting values to the `ownedNFTs` fields, via `mutableCollectionRef.ownedNFTs`.

```cadence
mutableCollectionRef.ownedNFTs.append(<-nft)    // OK
```

This massively reduces the boilerplate could a developer would have to if they are to achieve the same by writing
a function.

#### Case II: Code owns the container value

Assume `collection` is of type `Collection`. i.e. the code owns the value.
Then, there is no change to the current behavior.

```cadence
var collection: @Collection <- ...

// `collection.ownedNFTs` would return the concrete value, which is of type `@{UInt64: NFT}`.
// This is same as existing semantics.
// Also note that, this is an error because it is not possible to move nested resource.
var ownedNFTs: @{UInt64: NFT} <- collection.ownedNFTs
```

### Syntax for identity mapping

The idea of this change is to introduce a syntax to represent "identity mapping", which denotes mapping of an arbitrary 
entitlement `E` to the same entitlement `E`, without having to explicitly specify what the source (domain) and the
target (range) of the mapping are.
This saves the trouble of developers having to map each and every such entitlement explicitly.

The proposed solution consist of two parts.

#### Part I: Built-in 'Identity' entitlement mapping

A built-in entitlement mapping `Identity` is introduced, which maps any arbitrary entitlement to itself.
For example, inplace of using a custom mapping like below, developers can use the `Identity` entitlement mapping.

```cadence
entitlement mapping M {
    Mutable -> Mutable
    Insertable -> Insertable
    E1 -> E1
    E2 -> E2
}

pub resource Foo {
    access(M) fun someFunction()
}
```

Can be also written as:

```cadence
pub resource Foo {
    access(Identity) fun someFunction()
}
```

An added benefit of the proposed syntax is, developers wouldn't need to know or explicitly specify all the possible
entitlements that needs to be included in the mapping, beforehand.

#### Part II: Entitlement composition

There can be situations where a developer wants to re-use existing entitlement mappings, but also, rather than just
using them as-is, it might be needed to combine them with some new mapping entries.

The `include` keyword lets developers include all entries from another mapping, to their own mapping.

For an example:

```cadence
entitlement mapping M {
    include Identity

    Insertable -> Mutable
    Removable -> Mutable
}
```

The above mapping states that, any entitlement will map to itself, but also the two additional mapping entries would
also be in effect.

Entitlement composition is not limited to the `Identity` entitlement.
The `include` keyword can be used to combine any arbitrary set of entitlement mappings.


```cadence
entitlement mapping M1 {
    Removable -> Mutable
}

entitlement mapping M2 {
    Insertable -> Mutable
}

entitlement mapping M3 {
    include M1
    include M2
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
