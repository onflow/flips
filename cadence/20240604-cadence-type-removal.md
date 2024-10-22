---
status: proposed
flip: 275 
authors: Daniel Sainati (daniel.sainati@flowfoundation.org)
sponsor: Daniel Sainati (daniel.sainati@flowfoundation.org)
updated: 2024-10-22
---

# FLIP 275: Removal of Types in Contract Updates

## Objective

This adds a new pragma to Cadence that when processed by the contract update validator will
allow a type to be removed from a contract, and also prevent a type with the same name from being
added to that contract later. 

## Motivation

As originally outlined in https://github.com/onflow/cadence/issues/3210, users have been asking for
the ability to remove types from their contracts as they become outdated or otherwise unnecessary. 
The Cadence 1.0 upgrade in particular had exacerbated this need, as the changes to access control have
rendered a number of types obsolete.

## User Benefit

This will allow users to remove deprecated or unnecessary type definitions from their contracts.

## Design Proposal

The proposal is to introduces a new `#removedType` pragma to Cadence.
This pragma takes a single argument, an identifier specifying the type that got removed. 
Importantly, this identifier is not qualified, i.e. to remove a type `R` from a contract `C`, `#removedType(R)`
would be used, rather than `#removedType(C.R)`.
This is because the pragma is added at the same scope at which the type to be removed was originally defined. 
So, for example, to remove a resource `R` from `C` defined like so:

```cadence
access(all) contract C {

    access(all) resource R {
        // ... resource body
    }

    // ... contract body
}
```

the pragma would be placed here:

```cadence
access(all) contract C {

   #removedType(R)
   
    // ... contract body
}
```

Placing the pragma at the top level, or nested inside another definition in `C` has no effect, and does not permit the removal of `R`.

When used correctly, the pragma is processed by the contract update validator and allows an update to `C` to be 
performed that does not include a definition of the resource type `R`, even though the old version of `C` did include `R`.
In fact, when the `#removedType(T)` pragma is present, the contract update validator will reject any update to the 
contract that includes a definition of `T` at the same scope level as the pragma.
This is to prevent the type from being added back later with incompatible changes that would otherwise circumvent the
contract update validation restrictions.

Additionally, once present in a contract, the `#removedType(T)` pragma may never be removed, as this would allow the type
it removed to be re-added, once again potentially circumventing the update validation. 

Lastly, the `#removedType` pragma may only be used with concrete types (`struct`s, `resource`s, `enum`s, and `attachment`s),
not with interfaces. 
This is due to the fact that a removed interface cannot be removed from any existing conformance lists in which it is present,
and thus removing an interface would irrevocably break any downstream types that inherited from it. 

### Draft Implementation

A prototype implementation was added in https://github.com/onflow/cadence/pull/3376 and https://github.com/onflow/cadence/pull/3380.
This is currently disabled by default, and has been added only as a proof of concept.

### Drawbacks

Removing a type definition will break any downstream code that uses that type until uses of the type are removed.
In some cases this is more feasible than in others; if a resource definition `R` is removed, removing a field of type
`R` from a downstream contract is much easier than, say, fixing an attachment `A` that is designed to be used with `R`.
In the latter case, the attachment `A` is likely unusable and will also have to be removed.

However, it is worth mentioning that this is no different from someone removing a function definition from a type,
and any downstream code that depends on that particular function being broken, which is already possible. 
As such, the backward compatibility of deployed code is not guaranteed by (and is not an objective of) the contract
update validation anyway.

Additionally, any existing stored values of the removed type will be broken and un-usable forever. 
Because we do not currently possess a way to remove broken values from storage, the inability to load these values also
means they will sit in storage forever.
This change significantly increases the priority of implementing a solution for deleting broken values in storage. 

### Alternatives Considered

As outlined in https://github.com/onflow/cadence/issues/3210, an alternative pragma specifying a type replacement, 
rather than a removal was considered. We elected to do the first, easier solution due to time constraints, but are open 
to considering the second given sufficiently good reason. 

## Questions and Discussion

* Is it worth enabling the removal of interface types as well as regular composites?
