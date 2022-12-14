---
status: draft 
flip: NNN (do not set)
authors: Daniel Sainati (daniel.sainati@dapperlabs.com)
sponsor: Daniel Sainati (daniel.sainati@dapperlabs.com)
updated: 2022-12-14
---

# `auth` model changes

## Objective

This FLIP proposes two major changes to the language:
1) The addition of a new access modifier on contracts, structs and resources: `auth`
2) A change to the behavior of reference types to add granular authority restrictions to specific interfaces, and allow safe downcasting

## Motivation

Currently in Cadence, when users are in possession of a reference value, unless that reference is declared
with an `auth` type, users are unable to downcast that reference to a more specific type. This restriction
exists for security reasons; it should not be possible for someone with a `&{Balance}` reference to downcast that
to a `&Vault`. Cadence's current access control model relies on exposing limited interfaces to grant others capabilities
to resources they own. 

However, this can often result in restrictions that are unnecessary and onerous to users. In the above example, a user
with a `&{Balance}` reference would not be able to cast that reference to a `&{Receiver}` reference, despite the fact
that there would be no safety concerns in doing so; it should be possible to call `deposit` regardless of what kind of 
reference a user possesses. 

The proposed change is designed to overcome this limitation by allowing developers to directly mark which fields and functions
are dangerous and should be access-limited (using the new `auth` keyword), and which should be generally available to anyone. 
 
## User Benefit

How will users (or other contributors) benefit from this work? What would be the
headline in the release notes or blog post?

## Design Proposal

### `auth` fields

The first portion of this FLIP proposes to add a new access control modifier to field and function declarations in composite types:
`access(auth)`, which allows access to either the immediate owner of the resource (i.e. anybody who has the actual resource value),
or someone with an `auth` reference to the type on which the member is defined. 

Like `access(contract)` and `access(account)`, this new modifier sits exactly between `pub` and `priv` (or equivalently `access(self)`) 
in permissiveness; it allows less access than `pub`, but strictly more than `priv`, as an `auth` field or function can be used anywhere 
in the implementation of the composite. To see why, consider that `self` is necessarily of the same type as the composite, meaning 
that the access rules defined above allow any `auth` members to be accessed on it. 

As such, the following would be prohibited statically:

```cadence
pub resource R {
    access(auth) fun foo() { ... }
}
let r: &R = // ...
r.foo()
```

while all of these would be permitted:

```cadence
pub resource R {
    access(auth) fun foo() { ... }
    pub fun bar() {
        self.foo()
    }
}
let r: @R = // ...
r.foo()
let ref: auth &R = &r
ref.foo()
```

### Safely Downcastable References

The second, more complex part of this proposal, is a change to the behavior of references to allow the `auth` modifier not to 
refer to the entire referenced value, but to the specific set of interfaces to which the referenced value has `auth` permissions. 
To express this, the `auth` keyword can now be used with similar syntax to that of restricted types: `auth{T1, T2, ...}`, where the `T`s
in the curly braces denote the interface types to which that reference has `auth` acccess. This permits these references
to access `auth` members on the interfaces to which they have `auth` access. So, for example, given three interface definitions and 
a composite definition:

```cadecnce
pub resource interface A { 
    access(auth) fun foo()
}
pub resource interface B { 
    access(auth) fun bar()
}
pub resource interface C { 
    pub fun baz()
}
pub resource R: A, B, C {
    // ... 
}
```

a value of type `auth{A} &R` would be able to access `foo`, because the reference has `auth` access to `A`, but not `bar`, because
it does not have `auth` access to `B`. However, `baz` would be accessible on this reference, since it has `pub` access. 

The interfaces over which a reference can be authorized must be a supertypes of the underlying referenced type: it is nonsensical,
for example, to express a type like `auth{A} &{B}`, since `A` and `B` are disjoint interfaces that do not share a heirarchy, and thus
the `auth` access to `A` granted by this reference would have no effect. As such, given a list of `I`s in a reference `auth{I1, I2} &T`, 
we require that `I` appear in the conformance heirarchy of `T` if it is a composite type, or that `I` exists in the restriction set of `T` 
if it is a restricted type. 

The other part of this change is to remove the limitations on resource downcasting that used to exist. Prior to this change, 
non-`auth` references could not be downcast at all, since the sole purpose of the `auth` keyword was to indicate that references could be
downcast. With the proposed change, all reference types can be downcast or upcast the same way any other type would be. So, for example
`&{A} as! &R` would be valid, as would `&AnyResource as? &{B, C}` or `&{A} as? &{B}`, using the hierarchy defined above. 

However, the `auth`-ness (and the set of interfaces for which the reference is `auth`) would not change on downcasting, nor would that set
be expandable via casting. The subtyping rules for `auth` references is that `auth {U1, U2, ... } &X <: auth {T1, T2, ... } &X` whenever `{U1, U2, ... }`
is a superset of `{T1, T2, ... }`, or equivalently `∀T ∈ {T1, T2, ... }, ∃U ∈ {U1, U2, ... }, T = U`.

As such, `auth{A, B} &R` would be statically upcastable to `auth{A} &R`, since this decreases the permissions on the 
reference, it would require a runtime cast to go from `auth{A} &R` to `auth{A, B} &R`, as this cast would only succeed if the underlying 
runtime value was `auth` for both `A` and `B`. So in the code below:

```cadence
fun foo(ref: &R): Bool {
    let authRef = ref as? auth{A, B} &R
    return authRef != nil 
}
let r <- create R()
let ref1 = &r as auth{A} &R
let ref2 = &r as auth{A, B, C} &R
``

`foo` would return `true` when called with `ref2`, because the runtime type of `ref` in the failable cast is a subtype of `auth{A, B} &R`, since
`{A, B, C}` is a superset of `{A, B}`, but would return `false` when called with `ref1`, since `{A}` is not a superset of `{A, B}`.

### Drawbacks

Why should this *not* be done? What negative impact does it have? 

### Alternatives Considered

* Make sure to discuss the relative merits of alternatives to your proposal.

### Performance Implications

* Do you expect any (speed / memory)? How will you confirm?
* There should be microbenchmarks. Are there?
* There should be end-to-end tests and benchmarks. If there are not 
(since this is still a design), how will you track that these will be created?

### Dependencies

* Dependencies: does this proposal add any new dependencies to Flow?
* Dependent projects: are there other areas of Flow or things that use Flow 
(Access API, Wallets, SDKs, etc.) that this affects? 
How have you identified these dependencies and are you sure they are complete? 
If there are dependencies, how are you managing those changes?

### Engineering Impact

* Do you expect changes to binary size / build time / test times?
* Who will maintain this code? Is this code in its own buildable unit? 
Can this code be tested in its own? 
Is visibility suitably restricted to only a small API surface for others to use?

### Best Practices

* Does this proposal change best practices for some aspect of using/developing Flow? 
How will these changes be communicated/enforced?

### Tutorials and Examples

With this new design, the `Vault` heirarchy might be written like so:

```cadence
pub resource interface Provider {
    access(auth) fun withdraw(amount: UFix64): @Vault {
        // ...
    }
}

pub resource interface Receiver {
    pub fun deposit(from: @Vault) {
       // ...
    }
}

pub resource interface Balance {
    pub var balance: UFix64
}

pub resource Vault: Provider, Receiver, Balance {
    access(auth) fun withdraw(amount: UFix64): @Vault {
        // ...
    }
    pub fun deposit(from: @Vault) {
       // ...
    }
    pub var balance: UFix64
}
```

Then, someone with a `&{Balance}` reference would be able to cast this to a `&{Receiver}` and call `deposit` on it. 
They would also be able to cast this to a `&Vault` refrence, but because this reference is non-`auth` for `Provider`,
they would be unable to call the `withdraw` function. However, if a user possessed a `auth{Provider} &{Balance}` reference, 
they would be able to cast this to `auth{Provider} &{Provider}` and call `withdraw`.

### Compatibility

This would not be backwards compatible with existing code, and thus would need to be part of the Stable Cadence release. 
Most contracts would need to be audited and rewritten to use the new `auth` access modifier, as they would immediately become
vulnerable to having `pub` fields accessed from now-downcastable references. 

### User Impact

* What are the user-facing changes? How will this feature be rolled out?

## Related Issues

What related issues do you consider out of scope for this proposal, 
but could be addressed independently in the future?

## Prior Art

Does the proposed idea/feature exist in other systems and 
what experience has their community had?

This section is intended to encourage you as an author to think about the 
lessons learned from other projects and provide readers of the proposal 
with a fuller picture.

It's fine if there is no prior art; your ideas are interesting regardless of 
whether or not they are based on existing work.

## Questions and Discussion Topics

* The upcoming proposed changes to permit interfaces to conform to other interfaces will necessarily change the subtyping rules
for `auth` refrences, as it would no longer be sufficient to compare the sets for the two `auth` references just based on membership. 
The membership check would still exist, but it would function as a width subtyping rule, in addition to a depth rule based on
interface chains. I.e. it would be the case that for two auth references, `auth{U} &X <: auth{T1} &X`, if `U <: T`. 

We can combine this depth rule with the already existing width rule to yield a combined subtyping rule: `auth{U1, U2, ... } &X <: auth{T1, T2, ... } &X`, 
whenever `∀T ∈ {T1, T2, ... }, ∃U ∈ {U1, U2, ... }, T <: U`. 

So, for example, given an interface heirarchy: 

```cadence
pub resource interface A {}
pub resource interface B {}
pub resource interface C: A {}
pub resource interface D: B {}
pub resource interface E: C, D {}
pub resource R: E {}
```

we would have `auth{E} &R <: auth{C} &R <: auth{A} &R <: &R` and `auth{E} &R <: auth{D} &R <: auth{B} &R <: &R`.
It would also be the case that `auth{D, C} &R <: auth{A} &R` as well, and that `auth{E} &R <: auth{B, C} &R`.