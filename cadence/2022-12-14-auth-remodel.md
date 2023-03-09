---
status: draft 
flip: NNN (do not set)
authors: Daniel Sainati (daniel.sainati@dapperlabs.com)
sponsor: Daniel Sainati (daniel.sainati@dapperlabs.com)
updated: 2023-03-09
---

# Entitlements

## Objective

This FLIP proposes two major changes to the language:
1) The addition of new, bespoke access modifiers on fields and functions in composites and interfaces in the form of entitlements
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
should be access-limited using these new bespoke modifiers, and which should be generally available to anyone. 
 
## User Benefit

This feature would make interacting with references significantly easier, as users would no longer be restricted from
accessing methods like `deposit` that are safe for anyone to use, just because they do not possess a reference of the specific type, 
or because they are trying to write a generic method that works, say, on any `NFT` type. 

It would also increase safety, as we would be able to do additional checking on the values and types to which users create capabilities. 
For example, it is currently possible for a user to (presumably accidentally) create a public Capability directly to a `Vault` object, 
rather than a Capability to a `&{Balance, Receiver}` restricted type as is intended. With these changes, such a Capability still would 
not be subject to having its funds `withdraw`n, as this `Vault` Capability would not be `auth(Provider)`. In order to have access to an `auth`
method like `withdraw`, the user would need to explicitly create the Capability with an `auth(Provider)` reference, and we could enforce statically
(or warn) that `auth` reference capabilies should not be stored publicly. 

## Design Proposal

### Entitlements

The first part of this FLIP proposes to add a new declaration type to Cadence: `entitlement`s. `entitlement`s are declared similarly to interfaces, but
with a few key differences. They use the following syntax: `pub? entitlement <Identifier> { ... }`, where the body of the `entitlement` contains a list of 
functions or fields like in an interfaces. However, it is important to note three key distinctions: 

1) `entitlement`s are not kinded the way that interfaces are; they are neither resources nor structs, and as such cannot appear in any type position where a
kinded type is expected. In practice, this means entitlement annotations can only be used inside of the `auth` portion of references (described below).

2) `entitlement` functions cannot contain default implementations, pre-conditions, or post-conditions. `entitlement`s are used purely for access control, and
contain no polymorphism functionality like interfaces do. 

3) `entitlement` members are not declared with an access modifier the way that interface members are; this is because the `entitlement` is used to define
the access of these members on composites and interfaces, and thus it would be nonsensical for the entitlement definition itself to contain an access modifier. 

As such, the following is a valid entitlement definition:

```cadence
pub entitlement E {
    fun foo(a: Int)
    let x: String
}
```

while the following are not:

```cadence
pub entitlement A {
    pub fun foo(a: Int) // cannot have an access modifier
}
pub resource entitlement A { // entitlement is not kinded
    fun foo(a: Int) 
}
pub entitlement A {
    fun foo(a: Int) {} // cannot have a body
}
```

### Entitlement-access fields

To go with these `entitlement` declarations, this FLIP proposes to add a new access control modifier to field and function declarations in composite types:
`access(X)`, which allows access to either the immediate owner of the resource (i.e. anybody who has the actual resource value),
or someone with an `auth(X)` reference to the type on which the member is defined. The `X` here can be the qualified name of any entitlement, but
the function or field definition that uses that access modifier must match its definition in the `X` entitlement. So, this would be allowed:

```cadence
entitlement E {
    fun foo(a: Int)
}
resource R {
    access(E) fun foo(a: Int) { }
}
```

while this would not:

```cadence
entitlement E {
    fun foo(a: Int)
}
resource R {
    access(E) fun foo(a: String) { } // definition does not match `foo`'s definition in `E`. 
}
```

A single member definition can include multiple entitlements, using either a `|` or a `,` separator when defining the list.

An entitlement list defined using a `|` functions like a disjunction (or an "or"); it is accessible to any `auth` reference with any of those entitlements. 
An entitlement list defined using a `,` functions like a conjection set (or an "and"); it is accessible only to an `auth` reference with all of those entitlements. 

So, for example, in 

```cadence
entitlement E {
   fun foo() {}
   fun bar() {}
}
entitlement F {
   fun foo() {}
   fun bar() {}
}
resource R {
   access(E, F) foo() {}
   access(E | F) bar() {}
}
```

`foo` is only calleable on a reference to `R` that is `auth` for both `E` and `F`, while `bar` is calleable on any `auth` reference that is `auth` for either `E` or `F` (or both).

In either case, a member's definition must match all of its definitions in each of the `entitlement` declarations being used in the access modifier for that member, so in the below case:

```cadence
entitlement E {
   fun bar(a: A) {}
}
entitlement F {
   fun bar() {}
}
resource R {
   access(E | F) bar(a : A) {}
}
```

This would fail to type check because `bar`'s type does not match its definition in `F`. 

Additionally, because we only check that each member using an `entitlement` matches that entitlement's declaration of the member, it is not
necessary for a composite or interface to use every single member on an `entitlement` if they do not want or need to. So, for example, this code is valid:

```cadence
entitlement E {
   fun foo(a: A) {}
   fun bar(b: B) {}
}
resource R {
   access(E) foo(a: A) {}
}
```

because `foo`'s definition matches its definition in `E`. There is no check that `R` implmements all the members in `E`, however. This does not
pose an issue for type safety because `entitlement`s are never used for polymorphism, only access control. 

Like `access(contract)` and `access(account)`, this new modifier sits exactly between `pub` and `priv` (or equivalently `access(self)`) 
in permissiveness; it allows less access than `pub`, but strictly more than `priv`, as an `access(X)` field or function can be used anywhere 
in the implementation of the composite. To see why, consider that `self` is necessarily of the same type as the composite, meaning 
that the access rules defined above allow any `access(I)` members to be accessed on it. 

As such, the following would be prohibited statically:

```cadence
pub entitlement E {} 
pub resource R {
    access(E) fun foo() { ... }
}
let r: &R = // ...
r.foo()
```

while all of these would be permitted:

```cadence
pub entitlement E {} 
pub resource R {
    access(E) fun foo() { ... }
    pub fun bar() {
        self.foo()
    }
}
let r: @R = // ...
r.foo()
let ref: auth(E) &R = &r
ref.foo()
```

Note also that while the normal interface implementation subtyping rules would allow a composite to implement an `access(X)` interface member with a `pub` 
composite member, as this is less restrictive, in order to prevent users from accidentally surrendering authority and security this way, we prevent this statically. 
As such, the below code would not typecheck.

```cadence
pub entitlement E {
    fun foo()
}

pub resource interface I {
  access(E) fun foo() 
}

pub resource R: I {
  pub fun foo() {} // must also be access(E)
}
```

If users would like to expose an access-limited function to `pub` users, they can do so by wrapping the `access(X)` function in a `pub` function. 

As with normal subtyping, this would also be statically rejected:

```cadence
pub entitlement E {
    fun foo()
}

pub resource interface I {
  pub fun foo() 
}

pub resource R: I {
  access(E) fun foo() {}
}
```

since if this were to typecheck, anybody with a `&R` reference could upcast it to `&{I}` and thus gain the ability to call `foo`. 

When multiple interfaces declare the same function with different entitlements, a composite implementing both interfaces must use the `|` union of the function's entitlement 
sets as the access modifier for that function. E.g.

```cadence
pub entitlement E {
    fun foo()
}
pub resource interface I {
    access(E) fun foo() 
}

pub entitlement F {
    fun foo()
}
pub resource interface G {
    access(F) fun foo() 
}

pub resource R: I, G {
  access(E | F) fun foo() {} // valid, access(E | F) permits both access(E) and access(F)
}
pub resource S: I, G {
  access(E) fun foo() {} // invalid, does not match definition in G
}
pub resource T: I, G {
  access(F) fun foo() {} // invalid, does not match definition in I
}
```

### Safely Downcastable References

The second, more complex part of this proposal, is a change to the behavior of references to allow the `auth` modifier not to 
refer to the entire referenced value, but to the specific set of interfaces to which the referenced value has `auth` permissions (the reference's entitlements). 
To express this, the `auth` keyword can now be used with similar syntax to that of restricted types: `auth(E1, E2, ...)`, where the `E`s
in the parentheses denote the entitlements. This permits these references to access `access(E)` members for any `E` entitlement they possess. 
So, for example, given two entitlement definitions and a composite definition:

```cadence
pub entitlement A { 
    fun foo()
}
pub entitlement B { 
    fun bar()
}
pub resource R {
    access(A) fun foo() { ... }
    access(B) fun bar() { ... }
    pub fun baz() { ... }
}
```

a value of type `auth(A) &R` would be able to access `foo`, because the reference has the `A` entitlement, but not `bar`, because
it does not have an entitlement for `B`. However, `baz` would be accessible on this reference, since it has `pub` access. 

A reference type's entitlements must be valid entitlements of the referenced type: it is nonsensical, given a set of definitions like: 

```cadence
pub entitlement A { 
    fun foo()
}
pub entitlement B { 
    fun bar()
}
pub resource R {
    access(A) fun foo() { ... }
}
```

to create a reference like like `auth(B) &R`, since `R` has no functions with `B` entitlements. Thus 

```cadence
let r <- create R()
let ref = &r as auth(B) &R // cannot take a reference to `r` with a `B` entitlement
```

would fail statically. It is important to note, however, that because the type on the right-hand side of the `&` may be an interface, the dynamic type of a reference may
permit more entitlements than the static type. Consider the following code:

```cadence
pub entitlement A { 
    fun foo()
}
pub entitlement B { 
    fun bar()
}
pub resource interface I {
    access(A) fun foo()
}
pub resource R: I {
    access(A) fun foo() { ... }
    access(B) fun bar() { ... }
}
let r <- create R()
let ref1 = &r as auth(A, B) &R // valid
let ref2 = ref as auth(A, B) &{I} // invalid cast
let ref3 = ref as auth(A) &{I} // valid cast
```

Here, `auth(A, B) &R` is a valid type because both `A` and `B` are valid entitlements for `R`. However, if we wished to upcast this to a reference of 
type `&{I}`, this would not permit a `B` entitlement, so in order to upcast this we would need to drop the `B` entitlement to get `auth(A) &{I}`. The `B`
entitlement would remain present on the dynamic (run-time) type of this reference, however, and could be recovered with a runtime downcasting operation (described below).

The other part of this change is to remove the limitations on resource downcasting that used to exist. Prior to this change, 
non-`auth` references could not be downcast at all, since the sole purpose of the `auth` keyword was to indicate that references could be
downcast. With the proposed change, all reference types can be downcast or upcast the same way any other type would be. So, for example
`&{I} as! &R` would be valid, as would `&AnyResource as? &{I}` or `&{I} as? &{J}`, using the hierarchy defined above (for any `J`).

However, the `auth`-ness (and the reference's set of entitlements) would not change on downcasting, nor would that set
be expandable via casting. The subtyping rules for `auth` references is that `auth (U1, U2, ... ) &X <: auth (T1, T2, ... ) &X` whenever `{U1, U2, ...}`
is a superset of `{T1, T2, ...}`, or equivalently `∀T ∈ {T1, T2, ...}, ∃U ∈ {U1, U2, ...}, T = U`. Of course, all `auth` reference types
would remain subtypes of all non-`auth` reference types as before. 

As such, while `auth(A, B) &R` would be statically upcastable to `auth(A) &R`, since this decreases the permissions on the 
reference, it would require a runtime cast to go from `auth(A) &R` to `auth(A, B) &R`, as this cast would only succeed if the 
runtime type of the reference was entitled to both `A` and `B`. So in the code below:

```cadence
fun foo(ref: &R): Bool {
    let authRef = ref as? auth(A, B) &R
    return authRef != nil 
}
let r <- create R()
let ref1 = &r as auth(A) &R
let ref2 = &r as auth(A, B, C) &R
```

`foo` would return `true` when called with `ref2`, because the runtime type of `ref` in the failable cast is a subtype of `auth(A, B) &R`, since
`{A, B, C}` is a superset of `{A, B}`, but would return `false` when called with `ref1`, since `{A}` is not a superset of `{A, B}`.

In addition to the `,`-separated list of entitlements (which defines a conjunction/"and" set for `auth` modifiers similarly to its behavior for `access` modifiers), it is also possible, 
although very rarely necessary, to define `|`-separated entitlement lists in `auth` modifers for references, like so: `auth(E1 | E2 | ...) &T`. In this case, the type denotes that the
reference to `T` is authorized for **at least one** of the entitled specified, not that it is `auth` for all of them. This means, for example, that an `auth(A | B) &R` reference
would not be permitted to call an `access(A)` function on `R`, because we only know that the reference is authorized for one of `A` **or** `B`, and cannot guarantee that it is permitted to 
call an `A`-entitled function. However, an `auth(A | B) &R` reference could call an `access(A | B)` function on `R`, because the function requires one of `A` or `B`, and we know 
that our reference has at least one of these entitlements. 

```cadence
entitlement E {
    fun foo()
    fun bar()
    fun baz()
}
entitlement F {
    fun foo()
    fun bar()
    fun qux()
}
resource R {
    access(E | F) fun foo() { ... }
    access(E,  F) fun bar() { ... }
    access(E) fun baz() { ... }
    access(F) fun qux() { ... }
}
fun test(ref: auth(E | F) &R) {
    ref.foo() // allowed because `foo` requires either `E` or `F`
    ref.bar() // not allowed because the reference may not have `E` and `F`
    ref.baz() // not allowed because the reference may not have `E`
    ref.qux() // not allowed because the reference may not have `F`
    (ref as? auth(E) &R)?.baz() // allowed statically, will succeed at runtime if `ref` has an `E` entitlement
    (ref as? auth(F) &R)?.baz() // allowed statically, will succeed at runtime if `ref` has an `F` entitlement
    (ref as? auth(E, F) &R)?.qux() // allowed statically, will succeed at runtime if `ref` has both an `E` and an `F` entitlement
}
```

The subtyping rules for `|`-separated entitlement lists allow lists to expand on supertyping. I.e., `auth(A | B) &R <: auth(A | B | C) &R`, as this decreases the information we have about
the reference's entitlements and thus permits fewer operations. In general, `auth (U1 | U2 | ... ) &X <: auth (T1 | T2 | ... ) &X` whenever `{U1, U2, ...}`
is a subset of `{T1, T2, ...}`, or equivalently `∀U ∈ {U1, U2, ...}, ∃T ∈ {T1, T2, ...}, T = U`. To shrink the `|`-separated entitlement list, downcasting is required (as shown in the example above). 

`,`-separated entitlement lists subtype `|`-separated ones as long as the two sets are not disjoint; that is, as long as there is an entitlement in the subtype set that is also in the supertype set. 
This is because we are casting from a type that is known to possess all of the listed entitlements to a type that is only guaranted to possess one. More specifically, 
`auth (U1, U2, ... ) &X <: auth (T1 | T2 | ... ) &X` whenever `{U1, U2, ...}` is not disjoint from `{T1, T2, ...}`, or equivalently `∃U ∈ {U1, U2, ...}, ∃T ∈ {T1, T2, ...}, T = U`.

In practice `|`-separated entitlement lists never subtype `,`-separated ones except in the trivial case. This is because we are attempting to upcast (change the type without gaining any new
specificity) from a reference where we only know we possess at least one of the listed entitlement to one where we know we possess all of them. This is only possible when all of the references
in both lists are equal. More specifically, `auth (U1, U2, ... ) &X <: auth (T1 | T2 | ... ) &X` whenever `∀U ∈ {U1, U2, ...}, ∀T ∈ {T1, T2, ...}, T = U`. 
As one can see, this is only possible when every `U` and `T` are the same entitlement, or when the two entitlement lists each only have a single equivalent element. 

#### Attachments and Entitlements

Attachments would interact with entitlements and access-limited members in a nuanced but intuitive manner, where the attachment's entitlements
are implicitly parameterized over the entitlements of its `base` value. Attachments would remain `pub` accessible, but would permit the declaration of `access`-limited members. 
The `base` value, which currently is simply a `&R` reference for an attachment `attachment A for R` in any of `A`'s member functions, would now have its entitlements
depend on the entitlements of the member function in question. In a member declaration `access(X | Y) fun foo()` in `A`, the `base` variable would have
type `auth(X | Y) &R`, while in a member declaration in `A` `pub fun bar()`, `base` would just be an `&R`. This would effectively mean that the `access(X)`
members of the `base` would only be available to the attachment author in `access(X)` members on the attachment. Similarly, in an `access(X, Y)` 
member, the `self` reference of the attachment `A` would be `auth(X, Y) &A`, while in a `pub`-access member it would just be `&A`.

One important point to note here is that the previously mentioned rules about `auth(...) &T` reference types (namely that the set of entitlements on the `auth` modifier must all be valid for the referenced type `T`) require that attachments for any `base` type only use entitlements that are already present on that `base`. 
To see why, consider an attachment `A` declared `attachment A for R` with a member `access(X) fun foo()`. Within the body of `foo`, using the rules described above, the 
`base` reference would have type `auth(X) &R`. This type is only reasonable if `X` is a valid entitlement for `R`. 

This would then be combined with a change to the attachment access rules: rather than `v[A]` always returning an `&A?` value, the type of the returned
attachment reference would depend on the type of `v`. If `v` is not a reference, then any access `v[A]` would be fully authorized with type `(auth(owner) &A)?` (or similar
super-entitlement keyword). If `v` is a reference, then the access `v[A]` would return a reference to `A` with the same set of entitlements as `v`. This would prevent 
the attachment from accessing `auth` members on its `base` unless the specific instance of that base to which it is attached has the proper entitlement.

So, for example, given the following declaration:
```cadence
entitlement Withdraw {
    fun withdraw(_ amount: UFix64): @Vault 
}
interface Provider {
    access(Withdraw) fun withdraw(_ amount: UFix64): @Vault
}

attachment CurrencyConverter for Provider {
    pub fun convert(_ amount: UFix64): UFix64 {
        // ...
    }

    pub fun convertVault(_ vault: @Vault): @Vault {
        vault.balance = self.convert(vault.balance)
        return <-vault
    }

    access(Withdraw) fun withdraw(_ amount: UFix64): @Vault {
        let convertedAmount = self.convert(amount) 
        // this is permitted because this function has an entitlement to `Withdraw`
        // on the attachment, and thus `base` has type `auth(Withdraw) &{Provider}`
        return <-base.withdraw(amount: amount) 
    }

    pub fun deposit (from: @Vault) {
        // cast is permissable under the new reference casting rules
        let baseReceiverOptional = base as? &{Receiver}
        if let baseReceiver = baseReceiverOptional {
            let convertedVault <- self.convertVault(<-from) 
            // this is ok because `deposit` has `pub` access
            baseReceiver.deposit(from: <-convertedVault)
        }
    }

    pub fun maliciousStealingFunctionA(_ amount: UFix64): @Vault {
        // This fails statically, as `self` here is just an `&A` 
        // because `maliciousStealingFunctionA`'s access is `pub`,
        // and therefore `self` does not have access to `withdraw`
        return <-self.withdraw(amount)
    }

    pub fun maliciousStealingFunctionB(_ amount: UFix64): @Vault {
        // This fails statically, as `base` here is just an `&{Provider}` 
        // because `maliciousStealingFunctionB`'s access is `pub`,
        // and therefore `base` does not have access to `withdraw`
        return <-base.withdraw(amount: amount) 
    }
}

let vault <- attach CurrencyConverter() to <-create Vault(balance: /*...*/)
let authVaultReference = &vault as auth(Withdraw) &Vault
let converterRef = authVaultReference[CurrencyConverter]! // has type auth(Withdraw) &CurrencyConverter, can call `withdraw`

let otherVaultReference = &vault as &Vault
let otheConverterRef = otherVaultReference[CurrencyConverter]! // has type &CurrencyConverter, cannot call `withdraw`
```

### Drawbacks

This dramatically increases the complexity of references and capabilities by requiring users to think not only about which types they 
are creating the references with, but also which entitlements they are providing on each reference they create. It also increases 
the implementation complexity of the type system significantly by adding an additional dimension to the reference type's subtype heirarchy. 

### Best Practices

This would encourage users to use the new `auth` access modifier to restrict access on their contracts and resources; and use different
entitlement sets to control access. 

### Tutorials and Examples

With this new design, the `Vault` heirarchy might be written like so:

```cadence
pub entitlement Withdraw {
    fun withdraw(amount: UFix64): @Vault
}

pub resource interface Provider {
    access(Withdraw) fun withdraw(amount: UFix64): @Vault {
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
    access(Withdraw) fun withdraw(amount: UFix64): @Vault {
        // ...
    }
    pub fun deposit(from: @Vault) {
       // ...
    }
    pub var balance: UFix64
}
```

Then, someone with a `&{Balance}` reference would be able to cast this to a `&{Receiver}` and call `deposit` on it. 
They would also be able to cast this to a `&Vault` reference, but because this reference is not entitled to `Withdraw`,
they would be unable to call the `withdraw` function. However, if a user possessed a `auth(Withdraw) &{Balance}` reference, 
they would be able to cast this to `auth(Withdraw) &Vault` and call `withdraw`.

### Compatibility

This would not be backwards compatible with existing code, and thus would need to be part of the Stable Cadence release. 
Most contracts would need to be audited and rewritten to use the new access modifiers, as they would immediately become
vulnerable to having `pub` fields accessed from now-downcastable references. 

### User Impact

There is a huge user impact to rolling out this change; every single contract would become invalid, and would need to be manually audited
by the authors in order to be used. This is because all code previously written using the old model of access control (wherein giving someone
a `&{Balance}` would prevent them from calling `pub` functions on `Provider` like `withdraw`), would become invalidated, allowing everyone 
to call `pub` members like `withdraw` unless those methods were updated to be `access(Withdraw)`.  

### Rollout

There are two cases that need handling to migrate to this new paradigm: contracts and data. 

Existing references in storage (i.e. `Capability` values) would need to be migrated, as they would no longer be semantically valid under the new system. 
The simplest way to do this would be to ensure existing capabilities stay usable by converting existing capabilities and links' reference types to `auth` references, 
e.g. `Capability<&Vault{Withdraw}>` -> `Capability<auth(Withdraw) &Vault>`. Specifically, all existing references would become `auth` with regard to their entire borrow type,
as this does not add any additional power to these `Capability` values that did not previously exist. For example, `&Vault{Provider}` previously had the ability to call `withdraw`,
which was `pub` on `Provider`. After this change, it will still have the ability to call `withdraw`, as it is now `access(Withdraw)` on `Provider`, but the reference now has `auth(Withdraw)`
access to that type.

However, this does not handle the previously mentioned problem wherein existing contracts become vulnerable to exploitation, as all their `pub` functions would become
accessible to anybody with any kind of reference to a contract or resource. 

To handle this case we woukld "freeze" all existing contracts, preventing calling any code defined in them or interacting with their data until they are updated
at least once after the release of this feature (it's also possible that given the large number of breaking changes being released with Cadence 1.0, this restriction would happen
automatically and would not need special handling). Developers would be encouraged to give their contracts and resources the proper access modifiers. 

## Alternatives Considered

A previous version of this idea involved having the implementors of concrete types like `Vault` specify in their interface conformance list
which interfaces were `auth` for that concrete type. So, for example, one would write something like (using strawman syntax):

```cadence
pub resource interface A {
    pub fun foo()
}
pub resource interface B {
    pub fun bar()
}
pub resource R: auth A, B {
    pub fun foo() {}
    pub fun bar() {}
}
```

Then, given a reference of type `&R`, only the functions and fields defined on `B` would be accessible, since the reference is non-`auth`. In 
order to access the fields and functions on `A`, one would need an `auth` reference to `R`, since `A` was declared as `auth` in `R`'s conformances. 

The upside of this proposal was that there was a single decision point, in that the implementor of the concrete type needed to decide which interfaces
would be `auth` and which would be not, and the creators of the interfaces and the users of the concrete types would have no decisions to make other than 
whether or not the reference they are creating should be `auth` or not.

The drawback of this compared to the proposal in this FLIP is that it lacks granularity; an entire interface must be made `auth` or not, whereas in the 
FLIP's proposal individual functions can be declared with `auth` access or `pub` access. This allows the creator of the interface (and then later, the
creator of the reference type) to exert more fine-grained control over what is accessible. By contrast, the implementor of the concrete type has little
to no say in the access-control on their value beyond selecting which interfaces they wish to conform to. 

## Prior Art

`entitlement`s function similarly to facets in the Object Capabilities model. 

## Questions and Discussion Topics

* One potential change this unlocks would be to restrict the creation of Capabilities based on the entitlements
they contain. We could restrict the `public` domain to be only for non-entitled Capabilities, while the `private` domain would be only
for `auth`-references, which would prevent accidentally allowing anybody to get access to your `withdraw` function, for example.

* We've discussed the value of being able to declare entitlements in terms of existing entitlements. I.e. if I have
some entitlement `A` and `B`, it would be useful to be able to declare `C` as the intersection of `A` and `B` or the union. Additionally
it would be valuable to be able to declare `C` as a subset of `A` or `B`'s functions to reduce code repetition. However, this is not a
necessary part of the initial implementation of this FLIP, as it can be added later in a non-breaking way. As such this feature is
out of scope for this FLIP. 