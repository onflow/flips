---
status: draft
flip: 211
authors: Bastian Mueller (bastian@dapperlabs.com)
sponsor: Bastian Mueller (bastian@dapperlabs.com)
updated: 2023-10-16
---

# FLIP 211: Reject references to references.

## Objective

This FLIP proposes to reject references to references.

## Motivation

References to references are currently supported, but are not very useful.

With the changes introduced in Stable Cadence, namely entitlements, additional work is required to correctly support this functionality.
In particular, it must be ensured that the entitlements of the resulting reference is a subset of the available entitlements of the referenced reference.
This must be ensured when taking a reference directly, or indirectly by creating a reference to an optional reference.

i.e., the following example programs should be rejected:

```cadence
let ref: &[String] = &["foo"]
let outerRef = &ref as auth(Mutate) &(&[String])
```

```cadence
let ref: &[String] = &["foo"]
let optRef: &[String]? = ref
let outerRef = &optRef as auth(Mutate) &(&[String])?
```

Instead of spending additional work on ensuring correct behavior, remove support for references to references.

## User Benefit

Users will not be able to create references to references anymore.
Given that the creation of references to references likely just caused confusion, rejecting them is going to remove the potential for confusion.

## Design Proposal

The static type checking rules are changed to reject references to references, i.e. types of the form `&TT`,
e.g. the following program will be statically rejected:

```cadence
let ref: &Int = 1
let outerRef = &ref as &&Int
```

The dynamic evaluation rules are changed to reject the creation of references to references,
e.g. the following program will fail at run-time:

```cadence
let ref: &Int = 1
let outerRef = &ref as &AnyStruct
```

### Drawbacks

Given that references to references are not very useful, their current use is likely unintentional, and no actual functionality is lost by the removal.

### Alternatives Considered

As mentioned in the motivation, an alternative would be to fix/extend the current implementation.
