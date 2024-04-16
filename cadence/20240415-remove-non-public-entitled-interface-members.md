---
status: proposed
flip: 262
authors: Bastian Müller (bastian.mueller@flowfoundation.org)
updated: 2024-04-15
---

# FLIP 262: Remove non-public/entitled interface members

## Objective

This FLIP proposes to remove support for interface members (fields and functions)
with non-public or non-entitled access modifiers (`access(self)`, `access(contract)`,
`access(account)`) to avoid confusion and potential footguns.

## Motivation

Currently, it is possible to define interface members with
[any access modifier](https://cadence-lang.org/docs/1.0/language/access-control).

However, it is often unclear or misunderstood what it means to declare an interface member
with a non-public or non-entitled access modifier (`access(self)`,
`access(contract)`, `access(account)`).
In particular, there is confusion around how such members can be accessed from different places,
e.g. in the contract defining the interface, in the contract implementing the interface.

It is also often unclear or misunderstood what access modifiers may be used in the composite type
conforming to / implementing the interface.

Currently, the access modifier of a member in a type conforming to / implementing an interface
must not be more restrictive than the access modifier of the member in the interface.
That means an implementation may choose to use a more permissive access modifier than the interface.

This might be surprising to developers, as they might assume that the access modifier of the member
in the interface is a _requirement_ / _maximum_, not just a minimum, especially when using
a non-public / non-entitled access modifier.

Removing support for non-public / non-entitled access modifiers for interface members,
as well as requiring access modifiers of members in the implementation to match the access modifiers
of members given in the interface, should avoid confusion and potential footguns.

## User Benefit

Developers will hopefully be no longer confused and won't make assumptions that might lead to security issues.

## Design Proposal

Interface members may not use non-public or non-entitled access modifiers (`access(self)`,
`access(contract)`, `access(account)`).

If an interface member has an access modifier, a composite type that conforms to it / implements
the interface must use exactly the same access modifier.

### Drawbacks

This proposal removes functionality from Cadence that might be needed currently used and needed.

Even though the impact on code is likely low, including this proposal in Cadence 1.0 means
that there is yet another breaking change, fairly late in the process of its release.

### Alternatives Considered

None

### Performance Implications

None

### Dependencies

None

### Engineering Impact

Low.

It should be fairly easy to update the Cadence type checker to:
- Reject non-public / non-entitled access modifiers for interface members
- Enforce access modifiers of members in implementations match those of interfaces

Most effort will be spent on updating existing and adding new test cases.

### Best Practices

This proposal will render existing best practices for interfaces obsolete.

### Tutorials and Examples

For example, the following interface and composite type definitions are currently legal,
but will be no longer valid:

```cadence
access(all)
contract C1 {

    access(all)
    struct interface SI {

        access(contract)
        fun foo()
    }
}

access(all)
contract C2 {

    access(all)
    struct S: C1.SI {

        access(contract)
        fun foo() {}
    }
}
```

### Compatibility

The proposed changes might break existing Cadence programs.

### User Impact

This change is planned to be included in Cadence 1.0,
which already contains many other breaking changes.

## Prior Art

(Most) other languages  (Java, Kotlin, Swift, Rust, etc.) do not support non-public access modifiers
for interface members.
