---
status: Proposed 
flip: 258
authors: Joshua Hannan (joshua.hannan@flowfoundation.org)
         Cody Bouche (cody@evaluate.market)
sponsor: Joshua Hannan (joshua.hannan@flowfoundation.org) 
updated: 2024-03-11 
---

# Non-Fungible Token Storage Requirement

## Objective

Enforce that NFT projects store their NFTs in their `Collection`
in an `access(contract)` dictionary field and provide a default function
for iterating over the NFT IDs in the collection.

## Motivation

The original NFT standard enforced that `Collection`s had a field for storing NFTs:
```cadence
pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}
```

In the V2 NFT Standard, it was decided that we wanted to give projects more flexibility
for how they store their NFTs, so this requirement was removed.
We didn't realize at the time that this requirement actually had benefits like
allowing more sophisticated and standardized scripts that interacted with collections,
like iterating through IDs in a collection without having
to load all of them into memory at once.

## Design Proposal

[The conversation that triggered this](https://discord.com/channels/613813861610684416/621847426201944074/1227401046926692443)
is in Discord.
A [pull request with the suggested code changes](https://github.com/onflow/flow-nft/pull/211) 
is in the flow-nft github repository.


This FLIP proposes adding the `ownedNFTs` field back to the standard:

```cadence
access(contract) var ownedNFTs: @{UInt64: {NonFungibleToken.NFT}}
```

It also proposes adding a function to `NonFungibleToken.Collection`, `forEachID()`,
that allows iterating through the IDs in a collection without having
to load all of them into memory first:
```cadence
/// Returns an iterator that allows callers to iterate
/// through the list of owned NFT IDs in a collection
/// without having to load the entire list first
access(all) fun forEachID(_ f: fun (UInt64): Bool): Void {
    self.ownedNFTs.forEachKey(f)
}
```

The function specification provides a default implementation so that projects
don't have to implement the function themselves.

## User Benefit

With a standard specification for how to store NFTs, developers
can have more confidence in understanding any given NFT project's storage.

Without this storage specification, there would be no way to get all the IDs
in a large collection without hitting the computation limit.

This also allows the standard to define more sophisticated functionality that
interacts with stored NFTs, like we have done with adding the `forEachID()` function.
In the future, there may be other functionality that is enabled
by having standardized storage.

## Drawbacks

This change could be breaking for some contracts who
have already done their Cadence 1.0 upgrades.
We are confident that this will be negligible, if not zero, because
the NFT migration guide has already told developers to change their `ownedNFTs` field
to `access(contract)`, so they would still be compatible.

This also makes things a little more awkward for developers who do want to define
a more sophisticated system for storing NFTs
because they will still have to have the `ownedNFTs` field,
but it doesn't prevent them at all because they can simply ignore the field requirement
and do whatever else they want.

## Alternatives Considered

1. Keep the standard the same:
    * This would make some projects like evaluate not be able to function properly who rely on the iterator feature for their app.
2. Only introduce the `forEachID()` function with a default implementation but without the storage requirement
    * This would not potentially be a breaking change but would leave it up
    to the projects to implement the function, which is not reliable.
    We are far enough from the upgrade that this change should be acceptable.
3. Introduce a Cadence feature that allows getting slices from the IDs array
    * There isn't time to do this before the Cadence 1.0 upgrade.

## Performance Implications

The `forEachID()` also has a limit (200k) on how many IDs it can access,
but that is much larger than almost any collection on Flow will ever be.

## Dependencies


### Engineering Impact

* Build and test time will stay the same. they are relatively small changes.

### Best Practices


### Tutorials and Examples

* Check out the Cadence 1.0 migration guide for instructions on how to update contracts to the new standards:
  * https://cadence-lang.org/docs/cadence_migration_guide/

### Compatibility

* FCL, emulator, and other such tools should not be affected.

### User Impact

* The upgrade will go out on the networks at the same time as Cadence 1.0 (Crescendo) if approved.
* Users who are preparing for Cadence 1.0 will need to upgrade to the latest version
of the Cadence 1.0 CLI and/or emulator to test with the new version of the NFT Standard.