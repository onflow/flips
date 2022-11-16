# Interface inheritance in Cadence

| Status         | Proposed                                     |
|:---------------|:---------------------------------------------|
| **FLIP #**     |                                              |
| **Author(s)**  | Supun Setunga (supun.setunga@dapperlabs.com) |
| **Sponsor**    | Supun Setunga (supun.setunga@dapperlabs.com) |
| **Updated**    | 2022-10-24                                   |

## Objective

This FLIP proposes to allow interfaces to inherit other interfaces of the same kind.
This makes it easier for developers to structure their conformances and reduces a lot of redundant code.

## Motivation

Currently, concrete type implementations can conform to interfaces, but the interfaces cannot conform to other interfaces.
What this means is, an author of an interface has no way to mandate the implementations to conform to another interface.
The only way to enforce this at the moment is to duplicate all the fields, methods, and type requirements from
that other interface to the new interface.

e.g: Suppose there are two resource interfaces `Receiver` and `Vault`.

```cadence
pub resource interface Receiver {
    pub fun deposit(_ something: @AnyResource)
}

pub resource interface Vault {
    pub fun withdraw(_ amount: Int): @Vault
}
```

Suppose all implementations of the `Valut` would also need to conform to the interface `Receiver`.
Currently, there is no way to enforce this.

Anyone who implements the `Vault` would have to explicitly specify that their concrete type also implements the `Receiver`.
But it's not always guaranteed that all implementations would follow this informal agreement.

```cadence
pub resource MyVault: Vault, Reciever {

    pub fun deposit(_ something: @AnyResource) {}
    
    pub fun withdraw(_ amount: Int): @Vault {}
}
```

## User Benefit

Authors of interfaces would have the ability to mandate the implementations of their interfaces
to also conform to other interfaces.
They would no longer need to copy over methods, fields, or type requirements of other interfaces to their interface.
This also increases the re-usability of interfaces.

## Design Proposal

The solution proposed is to allow specifying interface conformance to interfaces. e.g:

```cadence
pub resource interface Receiver {
    pub fun deposit(_ something: @AnyResource)
}

pub resource interface Vault: Receiver {
    pub fun withdraw(_ amount: Int): @Vault
}
```

In the above example, the `Vault` interface conforms to the `Receiver` interface.
Anyone implementing the `Vault` interface would also have to implement the `Receiver` interface as well.

```cadence
pub resource MyVault: Vault {

    pub fun withdraw(_ amount: Int): @Vault {}

    pub fun deposit(_ something: @AnyResource) {}
}
```

The concrete type `MyVault` would not need to explicitly specify the `Receiver` interface conformance.
By conforming to the `Vault` interface, it is implicitly required to also conform to the `Receiver` interface.

```cadence
pub resource MyVault: Vault {

    pub fun withdraw(_ amount: Int): @Vault {}

    // Error: Missing implementation for `deposit(_ something: @AnyResource)` function.
}
```

However, having a chain of interface conformances would introduce some complexities.
Therefore, below restrictions apply for any interface inheritance.

### Duplicate interface members

When an interface implements another interface, it is possible for the two interfaces to have members
(functions, fields, and type requirements) with the same name.

#### Fields

If two fields with identical-names have identical types, then it will be valid.

```cadence
pub resource interface Receiver {
    pub var id: UInt64
}

pub resource interface Vault: Receiver {
    // `id` field has same type as the `Receiver.id`. Hence this is valid.
    pub var id: UInt64
}
```

Otherwise, interface conformance is not valid.

```cadence
pub resource interface Receiver {
    pub var id: Int
}

pub resource interface Vault: Receiver {
    // `id` field has a different type than the `Receiver.id`. Hence this is invalid.
    pub var id: UInt64
}
```

#### Functions

If two functions with identical names also have identical signatures, then it will be valid.

```cadence
pub resource interface Receiver {
    pub fun deposit(_ something: @AnyResource)
}

pub resource interface Vault: Receiver {

    // `deposit` function has the same signature as the `Receiver.deposit`. 
    // Also none of them have any default implementations. 
    // Hence this is valid.
    pub fun deposit(_ something: @AnyResource)
}
```

If the signatures of the two functions are different, then the interface conformance is not valid.

```cadence
pub resource interface Receiver {
    pub fun deposit(_ something: @AnyResource)
}

pub resource interface Vault: Receiver {
    // Error: `deposit` function has a different signature compared to the `Receiver.deposit`.
    // So these two cannot co-exist.
    pub fun deposit()
}
```

#### Functions with conditions

If the two functions with identical names and signatures have pre/post conditions, then it will still be valid.
However, the pre/post conditions would be linearized (TBD - refer [questions and discussion topics section](#questions-and-discussion-topics)) 
to determine the order of the execution of the conditions.
Given the pre/post conditions are `view` only, the order of execution would not have an impact on the conditions.

```cadence
pub resource interface Receiver {
    pub fun deposit(_ something: @AnyResource) {
        pre{ self.balance > 100 }
    }
}

pub resource interface Vault: Receiver {

    // `deposit` function has the same signature as the `Receiver.deposit`. 
    // Having pre/post condition is valid.
    // Both conditions would be executed, in a pre-determined order.
    pub fun deposit(_ something: @AnyResource) {
        pre{ self.balance > 50 }
    }
}
```

#### Default functions

It is also possible for two functions with identical signatures to have default implementations.
The priority will be given to the default implementation at the bottom of the inheritance chain.
i.e: Inherited default methods will be shadowed.

<div style="text-align: left;">
<img height="300" src="2022-11-16-interface-inheritance/img1.jpg"/>
</div>

```cadence
pub resource interface Receiver {
    pub fun log(_ message: String) {
        log(message.append("from Receiver"))
    }
}

pub resource interface Vault: Receiver {

    // This will shadow the implementation coming from the `Receiver` interface.
    pub fun log(_ message: String) {
        log(message.append("from Vault"))
    }
}

pub resource MyVault: Vault {}

pub fun test() {
    let myVault = MyVault()
    
    // Below will print "Hello from Vault"
    myVault.log("Hello")
}
```

In the above example, invoking the `log` method of `MyVault` will call the default implementation provided by
`Vault`, and the implementation provided by `Receiver` will be shadowed.

However, it is not allowed to have two or more inherited default implementations for an interface.

<div style="text-align: left;">
<img height="200" src="2022-11-16-interface-inheritance/img2.jpg"/>
</div>

```cadence
pub resource interface Receiver {
    pub fun log(_ message: String) {
        log(message.append("from Receiver"))
    }
}

pub resource interface Provider {
    pub fun log(_ message: String) {
        log(message.append("from Provider"))
    }
}

// Invalid: Two default functions from two interfaces.
pub resource interface Vault: Receiver, Provider {}
```

To resolve this ambiguity, the `Vault` interface would need to add a default implementation for the `log` function.

Having said that, there can be situations where the same default function can be available via different
inheritance paths.

<div style="text-align: left;">
<img height="300" src="2022-11-16-interface-inheritance/img3.jpg"/>
</div>

```cadence
pub resource interface Logger {
    pub fun log(_ message: String) {
        log(message.append("from Logger"))
    }
}

pub resource interface Receiver: Logger {}

pub resource interface Provider: Logger {}

// Valid: `Logger.log()` default function is visible to the `Vault` interface
// via both `Receiver` and `Provider`.
pub resource interface Vault: Receiver, Provider {}
```

In the above example, `Logger.log()` default function is visible to the `Vault` interface via both `Receiver`
and `Provider`. Even though it is available from two different interfaces, they are both referring to the same
default implementation. Therefore, the above code is valid.

However, if at least one of the interfaces in the middle of the chain also adds a default implementation to the `log`
function, then the code becomes invalid, as there are multiple implementations present now, which leads to ambiguity.

<div style="text-align: left;">
<img height="300" src="2022-11-16-interface-inheritance/img4.jpg"/>
</div>

```cadence
pub resource interface Logger {
    pub fun log(_ message: String) {
        log(message.append("from Logger"))
    }
}

pub resource interface Receiver: Logger {
    pub fun log(_ message: String) {
        log(message.append("from Receiver"))
    }
}

pub resource interface Provider: Logger {}

// Invalid: The default implementation of the `log` function provided by the `Logger`
// interface is visible to the `Vault` via the `Provider` interface.
// Another default implementation of `log` function is visible to the `Vault` via the `Receiver` interface.
// This creates ambiguity.
pub resource interface Vault: Receiver, Provider {}
```

Like in the previous scenario, to resolve this ambiguity, the `Vault` interface would need to add a default
implementation for the `log` function.

#### Conditions with Default functions

A more complex situation is where a default function is available via one inheritance path and a pre/post condition
is available via another inheritance path.

```cadence
pub resource interface Receiver {
    pub fun log(_ message: String) {
        log(message.append("from Receiver"))
    }
}

pub resource interface Provider {
    pub fun log(_ message: String) {
        pre{ message != "" }
    }
}

// Valid: Both the default function and the condition would be available.
pub resource interface Vault: Receiver, Provider {}
```

In such situations, all rules applicable for default functions inheritance as well as condition inheritance
would be applied.
Thus, the default function from coming from the `Receiver` interface, and the condition comes from the `Provider`
interface would be made available for the inherited interface.

#### Types and event definitions

Type and event definitions would also behave similarly to the default functions.
Inherited interfaces can override type definitions and event definitions.

```cadence
pub contract interface Token {
    pub struct Foo {}
}

pub contract interface NonFungibleToken: Token {
    pub struct Foo {}
}

pub contract MyToken: NonFungibleToken {
    pub fun test() {
        let foo: Foo  // This will refer to the `NonFungibleToken.Foo`
    }
}
```

If a user needed to access the `Foo` struct coming from the super interface `Token`, then they can
access it using the fully qualified name. e.g: `let foo: Token.Foo`.

However, it is not allowed to have two or more inherited type/events definitions with identical names for an interface.

```cadence
pub contract interface Token {
    pub struct Foo {}
}

pub contract interface Collectible {
    pub struct Foo {}
}

// Invalid: Two type definitions with the same name from two interfaces.
pub contract NonFungibleToken: Token, Collectible {
}
```
Similar to default functions, there can be situations where the same type/event definition can be available
via different inheritance paths.

```cadence
pub contract interface Logger {
    pub struct Foo {}
}

pub contract interface Token: Logger {}

pub contract interface Collectible: Logger {}

// Valid: `Logger.Foo` struct is visible to the `NonFungibleToken` interface via both `Token` and `Collectible`.
pub contract interface NonFungibleToken: Token, Collectible {}
```

In the above example, `Logger.Foo` type definition is visible to the `NonFungibleToken` interface via both `Token`
and `Collectible`. Even though it is available from two different interfaces, they are both referring to the same
type definition. Therefore, the above code is valid.

However, if at least one of the interfaces in the middle of the chain also overrides the type definition `Foo`,
then the code becomes invalid, as there are multiple implementations present now, which leads to ambiguity.

```cadence
pub contract interface Logger {
    pub struct Foo {}
}

pub contract interface Token: Logger {
    pub struct Foo {}
}

pub contract interface Collectible: Logger {}

// Invalid: The default implementation of the `Foo` struct by the `Logger`
// interface is visible to the `NonFungibleToken` via the `Collectible` interface.
// Another implementation of `Foo` struct is visible to the `NonFungibleToken` via the `Token` interface.
// This creates ambiguity.
pub resource interface NonFungibleToken: Token, Provider {}
```

Many other languages that support multiple-inheritance also follow similar approaches, where inheritances that
cause ambiguities are rejected by the compiler (see: https://en.wikipedia.org/wiki/Multiple_inheritance).

### Drawbacks

One drawback is, as mentioned in the above section, when there are complex inheritance chains with default functions
being available from multiple paths, overriding a default function by an interface in the middle of the chain
can lead to breaking the downstream codes that depend on this interface.

Disallowing multiple default implementations is done to avoid ambiguity and any unintuitive behaviors for the developers.
But it can also be argued that this can defeat the purpose of default functions: which is to avoid breaking downstream
dependencies.

### Alternatives Considered

None

### Performance Implications

None

### Dependencies

None

### Engineering Impact

This change has moderate implementation complexity.
Interface conformance for composite types (structs, resources, and contracts) is already implemented.
Interface inheritance can also follow the same approach, but may need additional work to handle complexities
in multiple inheritance, etc.

### Compatibility

Proposed changes are backward compatible.

### User Impact

This is a new feature addition, hence has no impact on the existing contracts.

## Related Issues

None

## Prior Art

- Many of the existing object-oriented languages such as C++, Go, Java, Kotlin, etc. supports interface inheritance.
- These languages also already deal with the complexities that come with multiple inheritance:
https://en.wikipedia.org/wiki/Multiple_inheritance 

## Questions and Discussion Topics

As mentioned in the [functions with conditions](#functions-with-conditions) section, it would be required to linearize
the function conditions, to determine the order of execution of pre- and post-conditions.
A possible option is to use the [C3 linearization](https://en.wikipedia.org/wiki/C3_linearization) algorithm.
