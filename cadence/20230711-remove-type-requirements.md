---
status: draft 
flip: NNN (do not set)
authors: Daniel Sainati (daniel.sainati@dapperlabs.com)
sponsor: Daniel Sainati (daniel.sainati@dapperlabs.com)
updated: 2023-07-24
---

# Remove Type Requirements from Cadence

## Objective

Currently, it is possible for a contract interface to declare a nested concrete type within itself. 
When this occurs, it requires any contracts that implement this interface to also provide an implementation 
of that nested concrete type. So, for example, given the following contract definition:

```cadence
access(all) contract interface Outer {
    access(all) resource Nested {
        access(all) field: Int
    }
}

access(all) contract ExampleOuter: Outer {
    access(all) resource Nested {
        access(all) var field: Int
        init(field: Int) {
            self.field = field
        }
    }
}
```

The definition of `Outer` includes a definition of a concrete type `Nested`. 
However, note that this concrete type is actually defined like an interface, and includes no implementation. 
Instead, this definiton of `Nested` requires any contracts implementing `Outer` to provide a definition of `Nested`
that conforms to the specification provided in `Outer`. This is called a nested type requirement. 

## Motivation

Type requirements are a very complex feature, both in their use and their implementation. 

In their usage, they behave differently than any other concrete type definitions, 
and as such are a source of confusion for users. 
Additionally they result in a large amount of boilerplate code for any contracts that implement
these interfaces, as they need to implement all of the type requirements, even when they are not
required for the specific functionality of the implementing contract. 

From the implementation side, 
they require a great deal of special-cased behavior that complicates the Cadence codebase. 

Removing this feature would simplify the language both for the users and the implementors, 
and with the addition of the ability to emit events direclty from interfaces, 
the primary use-case for this feature (requiring contracts to define certain events) has been removed. 

## User Benefit

This will simplify the language for users.  As an example, a contract previously written with a type requirement like so:

```
contract interface Interface {

    struct NestedType {}

    fun foo(v: NestedType) {}
}
```

would instead be written as 

```
contract interface Interface {

    struct interface NestedInterface {}

    fun foo(v: {NestedInterface}) {}
}
```

## Design Proposal

This will require two related changes to the codebase:

### Defining Concrete Events in Interfaces

First, the semantics of declaring an event in an interface will need to be changed. 
Currently, a definition such as 

```cadence 
access(all) contract interface Interface {
    access(all) event Foo()
}
```

does not declare a concrete event type `Foo`, but rather specifies a type requirement enforcing
that all concrete contracts implementing `Interface` also specify a `Foo` event type.
This is unnecessary boilerplate, since implementing `Interface` gives the implementer no choice
in how they implement `Foo`; they must simply copy the definition of `Foo` from the interface
to the concrete contract. 

We can remove this unnecessary code duplication by changing the semantics of this code to instead
define a concrete event type `Foo`, which can be referenced as such within `Interface` (i.e. if it is 
emitted by a condition or default implementation of one of `Interface`'s functions). Note that 
the semantics of `emit` require that the emitted event is defined in the same scope as the `emit` statement, 
so `Foo` would not be emittable outside of `Interface`. This is in order to ensure that the author
of `Interface` and `Foo` can be certain that all emitted `Foo` events originate from `Interface` code, and 
cannot originate from elsewhere. 

### Ban Non-Event Concrete Type Declarations in Interfaces

Coupled with this change to events would be a ban on definitions of all other concrete types inside interfaces. 
Specifically, resources, structs, enums, and attachments would no longer be declarable inside of interfaces. 
Attempting to declare a concrete type in an interface would result in a static error. 

Removing the ability to define a non-event concrete type in an interface, 
coupled with the change to the semantics of event definitions, would mean that the necessary first step of 
creating a type requirement (a concrete type definition in a contract interface) would become impossible, 
thus allowing us to remove all references and uses of this feature from the Cadence codebase. 

### Drawbacks

This will break any code that currently relies on nested type requirements. 

### Alternatives Considered

It is possible to leave this feature present in the language without blocking anything else. 
However, the release of Stable Cadence is our only chance to remove this complex legacy feature, 
and if we don't do it now we will be forced to support it indefinitely. 

It is also possible to change all concrete type declarations in interfaces to instead declare
a qualified concrete type, instead of just having events behave this way and banning the definitions
for other types. This would extend the changes to the semantics for event definitions to all concrete types. 

### Tutorials and Examples

Existing tutorials or examples that make use of type requirements will need to be updated. 

### User Impact

This will break a large amount of existing user code. 
Existing contracts that provide implementations for type requirements will not be updatable (since nested
type definitions cannot be removed), however these themselves will not break. Rather, the upstream
contract interfaces that provide the type requirements will instead break. These will be removable 
as they are not actually nested type definitions. 

## Questions and Discussion Topics
