---
status: proposed 
flip: NNN (set to PR number)
authors: Daniel Sainati (daniel.sainati@dapperlabs.com)
sponsor: Daniel Sainati (daniel.sainati@dapperlabs.com)
updated: 2023-08-14 
---

# FLIP NNN: Removal of Custom Destructors

## Objective

This FLIP proposes to remove support for destructors from Cadence. 
In particular, resources will neither be required nor able to declare a special `destroy` method. 
Instead, any nested resources will automatically be destroyed when the resource is destroyed. 

## Motivation

The primary motivation for this is to resolve the "trolling attack" that can be used to 
exploit the attachments feature, whereby an attachment with a `panic`ing destructor is added to a resource 
without the owner's knowledge, and thus unremovably takes up space in an account. By removing the ability
to declare custom destructors, malicious actors would not be able to prevent the destruction or removal 
of an attachment. 

## User Benefit

The primary benefit of this change would be the ability to enable the attachments feature without any
security concerns. 

However, a secondary benefit is that the language would change to reflect the practical realities of Cadence on-chain more closely. 
Although the language guarantees that the `destroy` method of a resource is invoked whenever that resource is destroyed with the `destroy`
statement, the reality is more complicated. There are ways to take a resource (e.g. an NFT or a token) "out of circulation" without actually
destroying it; an easy example would be sending that resource to a locked account where it is irretrievable. Such resources are destroyed in
the sense that they cannot be interacted with further, and as such any logic that relies on destructors to guarantee an accurate count of how many 
of a resource exist (e.g. `totalSupply` in the Fungible Token contract), to prevent the destruction of resources that do not satisfy certain conditions, 
or to emit an event when a resource is destroyed, is in practice impossible. 
As such, removing support for custom destructors would prevent developers being mislead about the possible guarantees that can be made about their resources. 

## Design Proposal

The design is simple; `destroy` will no longer be a declarable special function. Instead, 
when a resource is destroyed, any nested resources in that resource will be iteratively destroyed as well. Effectively,
the behavior will be the same as if the author of the `destroy` method had simply done the minimal implementation of that method, 
destroying sub-resources and nothing else. 

So, e.g. a resource `R` defined like this:

```cadence

resource SubResource {
    //...
}

resource R {
    let subResource: @Sub
    let subArray: @[Sub]

    // ...
}
```

would automatically destroy the `subResource` and `subArray` fields when it itself is destroyed. Users would not be able to 
rely on any specific order for the execution of these destructors, but because nothing can happen in destructors except for destruction 
of sub-resource, it would not be possible for the order to matter. 

### Drawbacks

This will break a large amount of existing code, and further increase the Stable Cadence migration burden. 

### Compatibility

This will not be backwards compatible. 

### Alternatives Considered

There has been some other discussion of potential alternatives to this in the past for solving the "trolling attack". 
The attack relies on a user defining an aborting destructor to make it impossible to destroy an attachment, and as such potential solutions
have been proposed to specifically ban panicking or aborting operations in destructors, or to have some kind of try-catch mechanism to continue
destruction after an abort. 

The issue with both of these solutions is their technical complexity; we estimate that either would take between 6 months to a year to design and
implement, and as they are both breaking changes, we would need to delay the release of Stable Cadence until the feature was complete. The hope
with this proposal is that the simpler change would both solve the problem and allow a faster release of Stable Cadence, with all its attendant benefits.

### User Impact

To evaluate the impact of this change, existing contracts on Mainnet were surveyed. 
A [custom linter](https://github.com/onflow/cadence-tools/pull/194) was defined to identify contracts containing complex destructors,
where a complex destructor is defined as one that performs and logic beyond recursively destroying sub-resources.

1711 total complex destructors exist currently on Mainnet.

Of these, 6 distinct categories of operations were identified (note that some destructors may fall into more than 1 category):

* Event emission: this destructor emits an event to signal that the resource was destroyed. 1328 of the complex destructors are of this kind
* Total Supply Decrementing: this destructor decrements a `totalSupply` variable. 492 of the complex destructors are of this kind
* Conditions/Assertions: this destructor asserts a condition, either with a pre/post condition or an assert statement. 29 of the complex destructors are of this kind
* Logging: this destructor logs some data. 17 of the complex destructors are of this kind
* Panic: this destructor panics under some circumstances. 10 of the complex destructors are of this kind
* Other complex operations: this destructor performs some logic that is not included in these categories (e.g. function call, resource movement). 211 of the complex destructors are of this kind. 

Based on this data, we can see that the two largest use cases for complex destructors currently is emitting an event signalling destruction, and decrementing the total supply for a token of some kind. 
While developers cannot actually rely on this logic to actually execute (events will not be emitted nor supply decremented when a resource is sent to a burner account), these use cases would be most
negatively impacted by this change. 

The Condition/Assertion and Panic categories are uncommon, and also anti-patterns. 
Destructors being able to abort execution is the source of the attachments-related "trolling" attack in the first place, 
and any solution we come up with for this would necessarily involve circumventing these operations.

## Prior Art

Move is an example of a resource-based smart contract language that does not allow the definition of custom destructors. 

## Questions and Discussion Topics

* Because this proposes to remove support for `destroy` entirely, it would be possible to re-add support back later once
we can make the method safer with respect to the trolling problem (e.g. by adding support for try-catch).

* It is theoretically plausible for us to special-case support for emitting events and decrementing total supply in destructors in order to maintain 
support for these use cases. However, there is a large amount of complexity involved, and support would be very restricted. 
In order to guarantee that the destructor cannot abort execution, we would have to prevent any operations that might possible abort at runtime. In particular, 
all function calls, array/dictionary accesses and numerical operations would not be permitted. As such, any events that take arguments would need to 
likely take those arguments as literals or fields on the resource; it would not be possible to do any computation as part of the event emission other than passing arguments. 
The total supply decrement support would be even more complex; in general subtracting two unsigned integers is not guaranteed not to abort because underflow is possible. 
We would need to somehow be able to identify programmatically what is and is not a "total supply" variable and allow it to be decremented in some way that cannot abort.
Is it worth the additional effort to support either of these use cases? 