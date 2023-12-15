---
status: implemented 
flip: 210
authors: Daniel Sainati (daniel.sainati@dapperlabs.com)
sponsor: Daniel Sainati (daniel.sainati@dapperlabs.com)
updated: 2023-10-16
---

# FLIP 210: Improve Entitlement Mapping Syntax

## Objective

This proposes a syntax change to make entitlement-mapped fields more visually distinct from entitled fields.

## Motivation

User feedback has indicated that entitlement mappings are too easy to confuse with entitlements, 
and since one of them has default behavior for public access and the other prevents it, 
it can be difficult to reason about what fields and functions are publicly available in the presence
of both entitled and mapped fields.

For example, given some definitions:

```
entitlement X
entitlement Y
entitlement mapping M {
    X -> Y
}
resource R {
    access(X) let foo: T
    access(M) let bar: T
}
```

It is not immediately obvious that given a reference `r` of type `&R`, `r.foo` would fail statically but `r.bar` would succeed and produce a `&T` reference.

## User Benefit

This will make it easier for users to distinguish between entitled members and mapped members. 

## Design Proposal

Currently, an entitlement mapped field or an entitlement mapped reference is specified by using the
name of a mapping (e.g. `M`) in an `access` or `auth` modifier (e.g. `access(M)` or `auth(M)`).  

Instead, we would require the keyword `mapping` to be present in the modifiers to indicate
that `M` is a mapping here, rather than an entitlement. 
So the above modifiers would become `access(mapping M)` and `auth(mapping M)`

### Drawbacks

This makes the creation of entitlement mapped fields and functions 16 characters more verbose. I.e. a function
that was previously declared as `access(M) fun foo(): auth(M) &T` would instead have to be declared as 
`access(mapping M) fun foo(): auth(mapping M) &T`, which is more verbose. 

### Alternatives Considered

We have not discussed alternative syntactic changes, but are open to suggestions. 
