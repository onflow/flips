---
status: accepted 
flip: 85
authors: Daniel Sainati (daniel.sainati@dapperlabs.com)
sponsor: Daniel Sainati (daniel.sainati@dapperlabs.com)
updated: 2023-05-05
---

# FLIP 85: Replace Restricted Types with Interface Set Types

## Objective

Currently in Cadence, a restricted type consists of a type (often confusingly also called
the restricted type) and a set of interface types that serve as its restrictions. Written `T{I1, I2, ...}`,
this type indicates a value of type `T`, but restricted such that only members present on one of the `I`s
can be accessed. This FLIP proposes to remove the `T` portion of the type, changing this type to simply
indicate some value of a type that conforms to each of the `I`s.  

## Motivation

The proposed entitlements changes in [the entitlements FLIP](https://github.com/onflow/flips/pull/54) make
restricted types mostly useless in their current form. Previously they existed as a form of access control for references,
as a restricted reference type could not be downcast, so a restriction set functioned for references like access control, 
limiting what functionality was available to a reference.

The entitlements changes will move this functionality to entitlements, and allow downcasting references, making 
this behavior unnecessary. However, it will still be valuable to allow the specification of "interface set" types, 
for the purposes of polymorphism, so by removing the outer `T` component of the restricted type `T{I1, I2, ...}`, we
can replace this obseleted type form with a simpler interface set type that handles the polymorphism use case. 

## User Benefit

This will simplify the language by removing a redundant and poorly understood feature. 

## Design Proposal

This would remove the "restricted type" portion of the restricted type. Users will no longer be able to write types with a `T` in 
the `T{I}` position of an old restricted type. 

For the new semantics of the interface set type, we can use the existing semantics of the `AnyStruct{X}` and `AnyResource{Y}` restricted types, 
as these contain no information about the type being restricted, and thus function only as restrictions on a generic type. 

The semantics of upcasting and downcasting restricted types will remain the same; they will be covariant in their interface sets. 

Existing references and capabilities could be migrated by replacing the restricted type with the outer type, i.e. converting `T{I}` to `T`. 
In combination with the incoming entitlement changes, where the old "restricted" behavior would be recaptured with entitlements, this would
be able to preserve existing behavior. In particular, in the case of references, the entitlements present on the interfaces would be granted to the
reference type. 

For example, an existing `&Vault{Withdraw}` value would be migrated to a `auth(Withdrawable) &Vault` reference

### Drawbacks

This is not backwards compatible, and will break existing restricted types that have non-trivial values for their outer type. 

### Alternatives Considered

We could simply leave restricted types as-is, which would break nothing, but would leave restricted types in the language
as a vestigial feature without much of a use case. 

### Compatibility

This is not backwards compatible.

### User Impact

As users are already going to be needing to rethink their restricted types as they migrate their contracts to use
entitlements, this is unlikely to significantly increase the engineering burden of upgrading their contracts to Stable
Cadence.
