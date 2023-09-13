# Mutability Restrictions

## Motivation

A previous version of Cadence ("Secure Cadence") restricted the potential foot-gun of mutating container-typed
`let` fields/variables via the
[Cadence mutability restrictions FLIP](https://github.com/onflow/flips/blob/main/cadence/20211129-cadence-mutability-restrictions.md).

However, there are still ways to mutate such fields by:
- Directly mutating a nested composite typed field.
- Mutating a field by calling a mutating function on the field.
- Mutating the field via a reference.

More details on this problem is described in the [FLIP for improving mutability restrictions](https://github.com/onflow/flips/pull/58).

## Problem in a nutshell

Consider the below example:

```cadence
pub resource MasterCollection {
    pub let kittyCollection: @Collection
    pub let topshotCollection: @Collection
}

pub resource Collection {
    pub(set) var id: String
    pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}

    pub fun deposit(token: @NonFungibleToken.NFT) { ... }
}
```

For owned values, despite the inner resources are declared with `let`:

1) Can directly mutate `id` field of the nested resource.

    ```cadence
    var masterCollection: MasterCollection <- ...
    
    masterCollection.kittyCollection.id = "NewID"
    ```

2) Can call a mutating function on the nested field.

    ```cadence
    var masterCollection: MasterCollection <- ...
    
    masterCollection.kittyCollection.deposit(<-nft)
    ```

3) Can take a reference to the inner `ownedNFTs`, and update it.

    ```cadence
    var masterCollection: MasterCollection <- ...
    
    let ownedNFTsRef = &masterCollection.kittyCollection.ownedNFTs as &{UInt64: NonFungibleToken.NFT}
    destroy ownedNFTsRef.insert(key: 1234, <-nft)
    ```

Similarly, for values that are not owned, i.e. when only a reference to the master collection is available,
the same set of problems exists.

```cadence
var masterCollectionRef: &MasterCollection = ...

// Directly updating the field
masterCollectionRef.kittyCollection.id = "NewID"

// Calling a mutating function
masterCollectionRef.kittyCollection.deposit(<-nft)

// Updating via the reference
let ownedNFTsRef = &masterCollectionRef.kittyCollection.ownedNFTs as &{UInt64: NonFungibleToken.NFT}
destroy ownedNFTsRef.insert(key: 1234, <-nft)
```

## Solution

In order to solve these existing problems, three FLIPs have been proposed, that tackles the problem in tandem:
- [Change member-access semantics](https://github.com/onflow/flips/pull/89)
- [Introducing built-in entitlements](https://github.com/onflow/flips/pull/86)
- [Improve entitlement mappings](https://github.com/onflow/flips/pull/94)

More details on each of these changes can be found in their respective FLIPs.
Here we are more interested in how they all work together to solve the aforementioned problems.

Let's take a look at the same example, this time utilizing all three proposed changes.

```cadence
pub resource MasterCollection {
    access(KittyCollectorMapping) let kittyCollection: @Collection
    access(TopshotCollectorMapping) let topshotCollection: @Collection
}

pub resource Collection {
    pub(set) var id: String
    access(Identity) var ownedNFTs: @{UInt64: NonFungibleToken.NFT}

    access(Insertable) fun deposit(token: @NonFungibleToken.NFT) { ... }
}


// Entitlements and mappings for `kittyCollection`

entitlement KittyCollector

entitlement mapping KittyCollectorMapping {
    KittyCollector -> Insertable
    KittyCollector -> Removable
}

// Entitlements and mapings for `topshotCollection`

entitlement TopshotCollector

entitlement mapping TopshotCollectorMapping {
    TopshotCollector -> Insertable
    TopshotCollector -> Removable
}
```

#### Owned values

All mutations are possible for owned values.

```cadence
var masterCollection: MasterCollection <- ...

// Directly updating the field
masterCollection.kittyCollection.id = "NewID"

// Directly updating the dictionary.
// Note that this wasn't possible before.
destroy masterCollection.kittyCollection.ownedNFTs.insert(key: 1234, <-nft)
destroy masterCollection.kittyCollection.ownedNFTs.remove(key: 1234)

// Calling a mutating function
masterCollection.kittyCollection.deposit(<-nft)

// Updating via the reference
let ownedNFTsRef = &masterCollectionRef.kittyCollection.ownedNFTs as &{UInt64: NonFungibleToken.NFT}
destroy ownedNFTsRef.insert(key: 1234, <-nft)
```

It is true that this still allows mutating nested fields, even the parent is declared with `let`.
But on contrary, `let` fields by-definition states that no new value can be assigned, but isn't designed to prevent
inner mutations.
This is the same behavior for most existing languages including, but not limited to, Java, Swift, etc.

Also, if someone owns the outer value, they can create a reference to an inner field with any desired entitlement,
which then allows mutations as defined in the entitlement-access.
Thus, it doesn't make much sense to prevent mutations to the owned values, since it'll be anyway possible by taking
a reference with required entitlements.

#### Reference values

On the other hand, when a reference is created, entitlements are used to control what can be accessed and what
can be modified, via the reference.
Therefore, the end goal is to streamline the use of entitlements to control mutations for nested fields as well,
when accessed through a reference.

#### Reference values, with no entitlements

Without entitlements, all fields return non-auth references. i.e. All fields are read only.

```cadence
var masterCollectionRef: &MasterCollection <- ...

// Error: Cannot update the field. Doesn't have sufficient entitlements.
masterCollectionRef.kittyCollection.id = "NewID"

// Error: Cannot directly update the dictionary. Doesn't have sufficient entitlements.
destroy masterCollectionRef.kittyCollection.ownedNFTs.insert(key: 1234, <-nft)
destroy masterCollectionRef.ownedNFTs.remove(key: 1234)

// Error: Cannot call mutating function. Doesn't have sufficient entitlements.
masterCollectionRef.kittyCollection.deposit(<-nft)

// Error: `masterCollectionRef.kittyCollection.ownedNFTs` is already a non-auth reference.
// Thus cannot update the dictionary. Doesn't have sufficient entitlements.
let ownedNFTsRef = &masterCollectionRef.kittyCollection.ownedNFTs as &{UInt64: NonFungibleToken.NFT}
destroy ownedNFTsRef.insert(key: 1234, <-nft)
```

#### Reference values, with entitlements

Entitlement states what mutations can be done on which fields.
Here, if an `auth{KittyCollector}` reference is available for `&MasterCollection`, then the returned `kittyCollection`
reference is entitled to `Insertable` and `Removable` operations.
Given `ownedNFT` field has `access(Identity)`, any entitlement available for `kittyCollection` field will be available
to the nested `ownedNFT` field as well.

Thus, all the following operations are valid.

```cadence
var masterCollectionRef: auth{KittyCollector} &MasterCollection <- ...

// Directly updating the field
masterCollectionRef.kittyCollection.id = "NewID"

// Updating the dictionary
destroy masterCollectionRef.kittyCollection.ownedNFTs.insert(key: 1234, <-nft)
destroy masterCollectionRef.kittyCollection.ownedNFTs.remove(key: 1234)

// Calling a mutating function
masterCollectionRef.kittyCollection.deposit(<-nft)
```

Note that, given the reference has entitlements only for `KittyCollector`, and because `TopshotCollectorMapping` has
no entries to that maps `KittyCollector` to anything, the accessing the field `topshotCollection` would return a
non-auth reference.
This doing any updates to the `topshotCollection` would be restricted.

```cadence
var masterCollectionRef: auth{KittyCollector} &MasterCollection <- ...

masterCollectionRef.topshotCollection.ownedNFTs.insert(key: 1234, <-nft)
```
