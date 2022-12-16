---
status: draft 
flip: NNN (do not set)
authors: Daniel Sainati (daniel.sainati@dapperlabs.com)
sponsor: Daniel Sainati (daniel.sainati@dapperlabs.com)
updated: 2022-12-16
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

This feature would make interacting with references significantly easier, as users would no longer be restricted from
accessing methods like `deposit` that are safe for anyone to use, just because they do not possess a reference of the specific type, 
or because they are trying to write a generic method that works, say, on any `NFT` type. 

It would also increase safety, as we would be able to do additional checking on the values and types to which users create capabilities. 
For example, it is currently possible for a user to (presumably accidentally) create a public Capability directly to a `Vault` object, 
rather than a Capability to a `&{Balance, Receiver}` restricted type as is intended. With these changes, such a Capability still would 
not be subject to having its funds `withdraw`n, as this `Vault` Capability would not be `auth`. In order to have access to an `auth`
method like `withdraw`, the user would need to explicitly create the Capability with an `auth` reference, and we could enforce statically
(or warn) that `auth` reference capabilies should not be stored publicly. 

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

Note also that the interface implementation rules allow a composite to implement an `access(auth)` interface member with a `pub` 
composite member, as this is less restrictive, but not the other way around, as this would allow the interface type to gain more access 
to the composite than should be possible. So this would be acceptable:

```cadence
pub resource interface I {
  access(auth) fun foo() 
}

pub resource R: I {
  pub fun foo() {}
}
```

since upcasting a value of type `&R` to type `&{I}` would not allow the reference to call any more functions. There is no similar concern 
with downcasting a `&{I}` to a `&R` and gaining the ability to call `foo`, because `foo` is declared as `pub` on `R`, and we treat the
access control declarations on the concrete composite as the "source of truth" for the value's access control at runtime. 

while this is not:

```cadence
pub resource interface I {
  pub fun foo() 
}

pub resource R: I {
  access(auth) fun foo() {}
}
```

since if this were to typecheck, anybody with a `&R` reference could upcast it to `&{I}` and thus gain the ability to call `foo`. 

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
```

`foo` would return `true` when called with `ref2`, because the runtime type of `ref` in the failable cast is a subtype of `auth{A, B} &R`, since
`{A, B, C}` is a superset of `{A, B}`, but would return `false` when called with `ref1`, since `{A}` is not a superset of `{A, B}`.

Because only interface types can appear in the curly braces after the `auth` keyword, the `auth` keyword without a restriction set will denote
full (unrestricted) `auth` access to the reference type. E.g. `auth &R` would be a reference with full `auth` access to `R`. 

### Drawbacks

This dramatically increases the complexity of references and capabilities by requiring users to think not only about which types they 
are creating the references with, but also which `auth` accesses they are providing on each reference they create. It also increases 
the implementation complexity of the type system significantly by adding an additional dimension to the reference type's subtype heirarchy. 

### Best Practices

This would encourage users to use the new `auth` access modifier to restrict access on their contracts and resources; and use different
`auth` restriction sets to control access. 

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

There is a huge user impact to rolling out this change; every single contract would become invalid, and would need to be manually audited
by the authors in order to be used. This is because all code previously written using the old model of access control (wherein giving someone
a `&{Balance}` would prevent them from calling `pub` functions on `Provider` like `withdraw`), would become invalidated, allowing everyone 
to call `pub` members like `withdraw` unless those methods were updated to be `auth`.  

## Prior Art

* This section needs filling out; would love to know if there are any languages out there doing something similar to this. 

## Questions and Discussion Topics

* The upcoming proposed changes to permit interfaces to conform to other interfaces will necessarily change the subtyping rules
for `auth` refrences, as it would no longer be sufficient to compare the sets for the two `auth` references just based on membership. 
The membership check would still exist, but it would function as a width subtyping rule, in addition to a depth rule based on
interface chains. I.e. it would be the case that for two auth references, `auth{U} &X <: auth{T1} &X`, if `U <: T`. 

    We can combine this depth rule with the already existing width rule to yield a combined subtyping rule: `auth{U1, U2, ... } &X <: auth{T1, T2, ... } &X`, 
whenever `∀T ∈ {T1, T2, ... }, ∃U ∈ {U1, U2, ... }, U <: T`. 

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
    It would also be the case that `auth{D, C} &R <: auth{A} &R` as well, because `C <: A`, and that `auth{E} &R <: auth{B, C} &R`, 
    beacuse `E <: B` and `E <: C`. 

* It is unclear how this should interact with the new attachments feature. Prior to this proposal a functional mental model 
for attachments was to treat them as implicit type-disambiguated `pub` fields on the resource or struct to which they were attached.
The existing behavior preventing downcasting of non-`auth` references would prevent someone from accessing a `Vault`-attachment (which 
would have access to functions like `withdraw` via the `base` reference) if they only possessed a `&{Balance}`, since only `Balance`-attachments
would be visible with such a reference. 

    One simple approach would be to allow attachments to be accessed only on values that have `auth` access to the attachment's `base` type. 
    I.e. given an `attachment A for I`, `v[A]` would be permissable when `v` has type `auth{I} &R` but not when `v` is just a regular `&R`, 
    or even an `&{I}`. Functionally this would mean that attachments would become `auth` fields in the mental model described above. 

    This approach is simple but very restrictive. In the motivating `KittyHat` use case for example, only users with an `auth`-reference 
    to the `Kitty` resource would be able to use the `Hat`. This would either make the `KittyHat` almost unusable without giving out
    an unreasonable amount of authority to users, or require the author of the `Kitty` to write a specific interface describing the set of 
    features they would want an attachment to be able to use. This latter case would completely defeat the point of attachments in the first place,
    since they are designed to be usable with no prior planning from the base value's author. 

    An alternative would be to implicitly parameterize the attachment's `auth` access over the `auth` access of its base value. In this model, 
    attachments would remain `pub` accessible, but would themselves permit the declaration of `auth` members. The `base` value, 
    which currently is simply a `&R` reference for an attachment `attachment A for R` in any of `A`'s member functions, would now have it's `auth`-ness
    depend on the `auth`-ness of the member function in question. In a member declaration `access(auth) fun foo()` in `A`, the `base` variable would have
    type `auth &R`, while in a member declaration in `A` `pub fun bar()`, `base` would just be an `&R`. This would effectively mean that the `auth`-access
    members of the `base` would only be available to the attachment author in `auth`-access members on the attachment. Similarly, in an `auth`-access 
    member, the `self` reference of the attachment `A` would be `auth &A`, while in a `pub`-access member it would just be `&A`.

    This would then be combined with a change to the attachment access rules: rather than `v[A]` always returning an `&A?` value, the type of the returned
    attachment reference would depend on the type of `v`. If `v` is not a reference, or a reference with `auth` access to `A`'s `base` type, then `v[A]` would return
    an `(auth &A)?` type, while if the reference did not have access to `A`'s `base`, then the access would be a regular `&A?` type. This would prevent 
    the attachment from accessing `auth` members on its `base` unless the specific instance of that base to which it is attached has the proper `auth` access.

    So, for example, given the following declaration:
    ```cadence
    attachment CurrencyConverter for Provider {
        pub fun convert(_ amount: UFix64): UFix64 {
            // ...
        }

        pub fun convertVault(_ vault: @Vault): @Vault {
            vault.balance = self.convert(vault.balance)
            return <-vault
        }

        access(auth) fun withdraw(_ amount: UFix64): @Vault {
            let convertedAmount = self.convert(amount) 
            // this is permitted because this function has `auth` access
            // on the attachment, and thus `base` has type `auth{Provider} &{Provider}`
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
    let authVaultReference = &vault as auth{Provider} &Vault
    let converterRef = authVaultReference[CurrencyConverter]! // has type auth &CurrencyConverter, can call `withdraw`

    let otherVaultReference = &vault as auth{Balance} &Vault
    let otheConverterRef = otherVaultReference[CurrencyConverter]! // has type &CurrencyConverter, cannot call `withdraw`
    ```

* One potential change this unlocks would be to restrict the creation of Capabilities based on the `auth`-ness of the reference
they contain. We could restrict the `public` domain to be only for non-`auth` Capabilities, while the `private` domain would be only
for `auth`-references, which would prevent accidentally allowing anybody to get access to your `withdraw` function, for example.