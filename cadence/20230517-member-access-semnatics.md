---
status: proposed
flip: 89
authors: Supun Setunga (supun.setunga@dapperlabs.com)
sponsor: Supun Setunga (supun.setunga@dapperlabs.com)
updated: 2023-05-17
---

# FLIP 89: Change Member Access Semantics

## Objective

This proposal suggest to return a reference when accessing a member of a container (field of a composite value,
key of a map, index of an array), if the parent value is also a reference and the accessed member is a container type.

## Motivation   

A previous version of Cadence ("Secure Cadence") restricted the potential foot-gun of mutating container-typed
`let` fields/variables via the
[Cadence mutability restrictions FLIP](https://github.com/onflow/flips/blob/main/cadence/20211129-cadence-mutability-restrictions.md).
However, this was a partial solution, and was only applicable for arrays and dictionaries, but did not solve it for
user-defined types.

There are still ways to mutate such fields by:
- Directly mutating a nested composite typed field.
- Mutating a field by calling a mutating function on the field.
- Mutating the field via a reference.

More details on this problem is described in the [FLIP for improving mutability restrictions](https://github.com/onflow/flips/pull/58).

This proposal is aimed at moving one step closer to solving this problem by returning references for member-access,
so that [Entitlements](https://github.com/onflow/flips/pull/54) can be used to control who can perform mutating
operations via references.

The [Mutability Restrictions Vision](https://github.com/onflow/flips/pull/97) explains how this proposed change
contributes to solving the aforementioned problems, and how the final solution looks like, with examples.

## User Benefit

As mentioned in the previous section, this change enables using Entitlements to prevent accidental mutations via references.

## Design Proposal

This proposal suggests field-access (i.e. `foo.bar`) and index-access (`foo[bar]`) to
return a reference to the targeting (field or element) value, instead of the value itself, if:
  - The parent (i.e. `foo`) is a reference to a container-typed value.
  - The accessed field/element is again container-typed.

where a "container" is any of:
  - Dictionary
  - Array
  - Composite
  - An optional of above

### Resource Example:

For example, consider the below `Collection` resource which has two fields: one (`id`) is String-typed,
and the other (`ownedNFTs`) is dictionary-typed.

```cadence
pub resource Collection {

    // Primitive-typed field
    pub var id: String

    // Dictionary typed field
    pub var ownedNFTs: @{UInt64: NFT}
}
```

#### Case I: Code owns the container value

Assume `collection` is of type `Collection`. i.e. the code owns the value.
Then, there is no change to the current behavior.

```cadence
var collection: @Collection <- ...

// `collection.ownedNFTs` would return the concrete value, which is of type `@{UInt64: NFT}`.
// This is an error because it is not possible to move nested resource.
// This is same as existing semantics.
var ownedNFTs: @{UInt64: NFT} <- collection.ownedNFTs

// `collection.id` would return the concrete string value.
// This is same as existing semantics.
var id: String = collection.id
```

#### Case II: Code only has a reference to the container value

Assume a reference to the collection `collectionRef` is available, and is of type `&Collection`. 
i.e. code doesn't own the value, but has only a reference to the value.

```cadence
var collectionRef: &Collection = ...

// `collectionRef.ownedNFTs` would now return a reference of type `&{UInt64: NFT}`.
var ownedNFTsRef: &{UInt64: NFT} = collectionRef.ownedNFTs

// However, `collectionRef.id` would return the value, since it is a primitive type.
// This is same as existing semantics.
var id: String = collectionRef.id
```

It is also important to note that, the returned reference has no entitlements assigned to them.
i.e: they are **non-**`auth` references.

#### Nested resources

Assume a nested resource collection as bellow.

```cadence
pub resource MasterCollection {
    pub var kittyCollection: @Collection
    pub var topshotCollection: @Collection
}


pub resource Collection {
    pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}
    
    access(Withdraw) fun withdraw(withdrawID: UInt64): @NonFungibleToken.NFT {
        let token <- self.ownedNFTs.remove(key: withdrawID) ?? panic("missing NFT")
        emit Withdraw(id: token.id, from: self.owner?.address)
        return <-token
    }
}
```

Also, suppose a reference to the `MasterCollection` without any entitlements (i.e: non-auth) is available/borrowed.

```cadence
var masterCollectionRef: @MasterCollection = ...
```

Previously, it was possible to withdraw from the inner collection, despite the `masterCollectionRef` not having the
`Withdraw` entitlement.

```cadence
masterCollectionRef.kittyCollection.withdraw(24)    // OK
```

With the proposed changes, this will be an error, since `masterCollectionRef.kittyCollection` is now return a 
non-auth reference, and calling `withdraw` is prohibited because that reference doesn't have the `Withdraw` entitlement. 

```cadence
masterCollectionRef.kittyCollection.withdraw(24)    // Static Error
```

### Struct Example:

Similarly, consider the below `NFTView` struct which has two fields: one (`id`) is UInt64-typed,
and the other (`royalties`) is array-typed.

```cadence
pub struct NFTView {

    // Primitive-typed field
    pub var id: UInt64

    // Array typed field
    pub var royalties: [Royalty]
}
```

#### Case I: Code owns the container value

Assume `nftView` is of type `NFTView`. i.e. the code owns the value.
Then, there is no change to the current behavior.

```cadence
var nftView: NFTView = ...

// `nftView.royalties` would return the concrete value, which is of type `[Royalty]`.
// This is same as existing semantics.
var royalties: [Royalty] = nftView.royalties

// `nftView.id` would return the concrete `UInt64` value.
// This is same as existing semantics.
var id: UInt64 = collection.id
```

#### Case II: Code only has a reference to the container value

Assume a reference to the nftView `nftViewRef` is available, and is of type `&NFTView`.
i.e. code doesn't own the value, but has only a reference to the value.

```cadence
var nftViewRef: &Collection = ...

// `nftViewRef.royalties` would now return a reference of type `&[Royalty]`.
var royaltiesRef: &[Royalty] = nftViewRef.royalties

// However, `nftViewRef.id` would return the value, since it is a primitive type.
// This is same as existing semantics.
var id: String = nftViewRef.id
```

Similar to resources, the returned reference has no entitlements assigned to them.
i.e: they are **non-**`auth` references.

### Drawbacks

None

### Alternatives Considered

None

### Performance Implications

There is no direct performance impact by this change.

However, since accessing a nested objects through a reference would now be returning reference,
it will avoid the copying of nested objects, which may increase the performance for certain use-cases.

### Dependencies

None

### Engineering Impact

This change has medium complexity in the implementation.

### Compatibility

This is a backward incompatible change.

### User Impact

Many deployed contracts might be affected by this change.

## Related Issues

None

## Prior Art

None

## Questions and Discussion Topics

None