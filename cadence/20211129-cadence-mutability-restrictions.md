---
status: accepted
flip: 703
authors: Daniel Sainati (daniel.sainati@dapperlabs.com)
sponsor: Daniel Sainati (daniel.sainati@dapperlabs.com)
updated: 2021-12-08
---

# FLIP 703: Mutability Restrictions

## Objective

This proposed change would limit the scopes in which the fields of composite types
like contracts, structs, and resources can be mutated. Instead of allowing array
and dictionary fields to be modified in any scope where the field can be read,
Cadence would issue a type error. These fields would instead be only modifiable
in the current declaration scope, as well as inner scopes of that scope.

## Motivation

Accidentally exposing a mutable variable to external consumers of a contract is currently a
large potential footgun standing in the way of a release of a stable, trustless version of
Cadence. Developers may declare a "constant" field on their contract with `pub let`, intending
that the field only be readable to transactions and other contracts, and unintentially allow
other code to add or remove elements from a dictionary or array stored in that field. Consider this code:

```cadence
pub contract Foo {
	pub let x: [Int]

	init() {
	    self.x = []
	}
}

// in some external code importing Foo
pub fun bar() {
	Foo.x.append(1)
}
```

Currently Cadence does not warn against this, or prevent a developer from writing this code, even
though, depending on what is stored in `x`, this could be unsafe.

## User Benefit

This change makes it less likely for a user to expose a value from a contract or other
publicly accessible structure that is not intended to be mutable, but can be modified
by consumers of their contract regardless. This is important for a future stable,
permissionless version of Cadence, where users do not need to have their smart
contracts reviewed in order to deploy them to Flow. By removing one possible way that
users could write unsafe smart contracts, we make it easier for everyone to trust the
correctness of contracts deployed to Flow.

## Design Proposal

### Explanation

Cadence controls where and to what extent variables can be read and written using a combination of
access modifiers and declaration kinds, as described [here](https://docs.onflow.org/cadence/language/access-control/).
Of note is that the `let` kind does not allow fields to be written to in any scope, whereas `var` allows them
to be reassigned in the "Current and Inner" scopes; that is, the scope in which the field was declared, and any scopes
contained within that scope.

However, simply writing to a field directly is not the only way in which one can modify a value. Consider the following example:

```cadence
pub struct Foo {
    pub let x: [Int]

    init() {
        self.x = [3]
    }
}

pub fun bar() {
    let foo = Foo()
    foo.x = [0] // reassignment to `x` is not allowed, as variable is `let`
    foo.x[0] = 0 // mutates the array, but does not reassign to `x`, currently allowed
}
```

This FLIP would change Cadence to also restrict the scopes in which an array or dictionary field can be modified (or "mutated").
Examples of mutating operations include an indexed assignment, as in the above example, as well as calls to the `append` or `remove`,
methods of arrays, or the `insert` or `remove` methods of dictionaries. These operations will only be able to be performed on a field
in the current and inner scopes, the same contexts in which the field could be written to if it were a `var`. So the following
would typecheck:

```cadence
pub struct Foo {
    pub let x: [Int]

    init() {
        self.x = [3]
    }

    pub fun addToX(i: Int) {
        self.x.append(i)  // mutation of field `x` in same scope is valid, even though field is `let`
    }
}
```

while the following would not:

```cadence
pub struct Foo {
    pub let y: [Int]

    init() {
        self.y = [3]
    }
}

pub fun main() {
    let foo = Foo()
    foo.y.append(1) // invalid, field `y` is let and cannot be mutated
}
```

This is designed to prevent external code from mutating the values of fields it can read from your contract. Consumers
of your contract may read the values in a `pub let` or `pub var` field, but cannot change them in any way.

If you wish to allow other code to update or modify a field in your contract, you may expose a method
to do so, like in the example above with `addToX`, or you may use the `pub(set)` access mode, which
allows any code to mutate or write to the field it applies to.

### Implementation Details

This change adds a new error, the `ExternalMutationError`, which is raised when a field
is mutated outside of the context in which it was defined. The error message will also
suggest that the user instead use a setter or modifier method for that field.

Specifically, the error is reported whenever a user attempts to perform an
index assignment on a member that is not either declared with the `pub(set)`
access mode, or is defined in the current enclosing scope. This check is the
same one performed for writing to fields, with the difference that mutation
is allowed on both `let` and `var` fields, while only the latter can be written to.

Additionally, array and dictionary methods now track an additional bit of information
indicating whether they mutate their receiver. Mutating methods may not be called on
members that are not declared with the `pub(set)` access mode, or defined in the current
enclosing scope.

The array methods that are considered mutating are `append`, `appendAll`, `remove`,
`insert`, `removeFirst` and `removeLast`. The dictionary methods that are considered
mutating are `insert` and `remove`. Additionally, indexed assignment (e.g. `x[y] = e`) will
be considered mutating for both arrays and dictionaries.

The limitations on mutation are designed to closely mirror the limitations on writes,
so that they can be easily explained to and understood by the user in terms of
language principles with which they are already familiar. Similarly, the suggested
workaround of adding a setter or modifier method to the composite type is designed
to be immediately recognizable to any developer familiar with object-oriented
design principles.

### Drawbacks

This has the potential to break a number of existing contracts by restricting
code that was previously legal. This change would require a migration path for
these contracts in order for developers to update their contracts to satisfy
Cadence's new restrictions.

This also removes a small amount of expressivity from the language. Previously,
the `pub let` declaration was a way to create a field that could be read and
mutated in all contexts, but not written, while `pub(set) var` created
a field that could be read, written and mutated in all contexts. After this change,
it would not be possible to describe a field that can be read and mutated but not set,
as `pub let` would only allow reads in arbitrary contexts.

### Alternatives Considered

One possible approach would be to add mutability modifiers to field or variable declarations,
like the `readonly` or `mut` tags found in languages like Rust, C# and TypeScript. This would make
all fields either mutable or immutable by default, requiring users to supply the appropriate tag
to make their behavior be whichever is not the default.

This approach has the unfortunate side effect of increasing the surface area of the language, while
not also adding immediate benefit; the `readonly` and `mut` distinction may be redundant when
`let` and `var` already exist in the language. To justify this addition, we would need to identify
a use case for allowing field to be written to but not mutated (`readonly var`), or for restricting
the mutation of fields even within their enclosing type.

Another approach would be to make `let` a true constant declaration, and forbid writing or mutating it in any context,
while allowing `var` to be mutated in all the contexts it can be written. This, however, is likely too restrictive;
a complex initialization for a dictionary or array value may benefit from the ability to iteratively mutate its contents,
up to the point at which its value is finalized, and can thus be exported. As such we would like to still allow users
to perform mutations in a limited context.

A third approach would be to ban contract-level public field, with the idea that users cannot accidentally shoot
themselves in the foot by exporting a `pub let` field from their contract and having it be externally mutated
if `pub let` fields cannot exist on contracts in the first place. However, this has a number of downsides. The
obvious one is that all data that one might wish to expose to be read from a contract would need an explicit getter
method. The second, less obvious but also more concerning downside, is that it would be necessary to
ban `pub let` fields on structs and resources as well. Consider the case where a user exports a getter method
to read a struct or resource used by their contract. Any `pub let` fields on this struct or resource
would also be mutable, and thus are subject to the same risks they would be if they were exported
directly from the contract.

Consider:

```cadence
pub contract C {
    pub struct Foo {
        pub let arr: [Int]
        init() {
            self.arr = [3]
        }
    }

    priv let foo: Foo

    init() {
        self.foo = Foo()
    }

    pub fun getFoo(): Foo {
        return self.foo
    }
}

pub fun main() {
    let a = C.getFoo()
    a.arr.append(0) // a.arr is now [3, 0]
}
```

Another approach would be to have fields use a different type depending on the scope in which they are
viewed. Within a struct, a `pub let` array field would have the type `[Int]`, permitting all array
operations on it, including mutating ones. Outside the struct, the same field might have the type
`[Int]{ReadonlyArray<Int>}`, in effect a restricted type that only permits the non-mutating
operations to be performed on that array. This, however, is overly complex, and introduces
a new type to accomplish the same purpose as what this FLIP proposes.

### Performance Implications

* This is a change to the Cadence type checker, so by adding more checks and errors
for it to report, it may have a performance impact. However, this should not add
signficantly more work to typechecking of programs that succeed on typechecks.

### Best Practices

* This changes the way that users will need to expose fields that they wish for
consumers of their code to be able to modify. While in the past, a field declared
with `pub let` could be mutated, now users will need to provide modifier or setter
functions on their contracts or other structures if they wish to allow this behavior.
This will be communicated via changes to the Cadence documentation.

### Examples

Some examples of code that produces a type error as a result of this restriction:

```cadence
pub resource Foo {
    pub let x: {Int: Int}

    init() {
        self.x = {0: 3}
    }
}

pub fun bar() {
    let foo <- create Foo()
    foo.x[0] = 3 // cannot mutate `x`, field was defined in `Foo`
    destroy foo
}
```

```
pub struct Bar {
    pub let foo: Foo
    init() {
        self.foo = Foo()
    }
}

pub struct Foo {
    pub let x: [Int]

    init() {
        self.x = [3]
    }
}

pub fun bar() {
    let bar = Bar()
    bar.foo.x[0] = 3 // cannot mutate `x`, field was defined in `Foo`
}
```

```cadence
pub contract Foo {
    pub let x: S

    pub struct S {
        pub let y: [Int]
        init() {
            self.y = [3]
        }
    }

    init() {
        self.x = S()
    }
}

pub fun bar() {
    Foo.x.y.remove(at: 0) // cannot mutate `y`, field was defined in `S`
}
```

```cadence
pub contract Foo {
    pub let x: S

    pub struct S {
        pub let y: [Int]
        init() {
            self.y = [3]
        }
    }

    init() {
        self.x = S()
        self.x.y.append(2) // cannot mutate `y`, field was defined in `S`
    }
}
```

while the following are allowed:

```cadence
pub struct Foo {
    pub let x: {Int: Int}

    init() {
        self.x = {3: 3}
    }

    pub fun bar() {
        let foo = Foo()
        foo.x[0] = 3 // ok, mutation occurs inside defining struct Foo
    }
}
```

```cadence
pub struct Foo {
    pub(set) var x: {Int: Int}

    init() {
        self.x = {3: 3}
    }
}

pub fun bar() {
    let foo = Foo()
    foo.x.insert(key: 0, 3) // ok, pub(set) access modifier allows mutation
}
```

### Compatibility

This change is not backwards compatible with existing Cadence smart contracts, but
is being made in service of a future version of Cadence wherein no backwards incompatible
changes will be required going forwards. It should not affect any other parts of Flow.

### User Impact

This will change the way users write their smart contracts. Given that this is preventing
users from writing an unsafe pattern, we do not expect this to invalidate a particularly
large amount of existing code. However, any code that is invalidated by this change can be
refactored to expose a setter/modifier method for affected fields.

## Related Issues

Do we wish to handle aliased references? Consider the following example (courtesy of @SupunS):
```cadence
pub fun main() {
  let foo <- create Foo()

  log(foo.immutable)

  foo.mutable.content = "updated" // mutating the 'immutable' via 'mutable'

  log(foo.immutable)

  destroy foo
}

pub resource Foo {
  pub(set) var mutable: &Bar

  pub let immutable: &Bar

  init() {
    self.mutable = &Bar() as &Bar

    self.immutable = self.mutable.   // immutable holds a direct/indirect reference to a mutable
  }
}

pub struct Bar {
  pub(set) var content: String
  init() {
    self.content = "original"
  }
}
```

This is one potential way to get around mutability restrictions on `pub let` fields, by aliasing such
a field with a `pub(set) var`. Is preventing this kind of issue within the scope of this change? How
would we begin to approach such a restriction?

## Prior Art

In Swift (from where Cadence's access modifiers already draw their inspiration), one can define a field
with a `public private(set) var` access modifier. A field defined this way can be read in a public context,
but can only be set privately. In effect, the change described in this FLIP is changing the default behavior
of fields to have what Swift would consider a private setter.

## Questions and Discussion Topics

