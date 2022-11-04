**# Improve Mutability Restrictions

| Status         | Proposed                                     |
|:---------------|:---------------------------------------------|
| **FLIP #**     |                                              |
| **Author(s)**  | Supun Setunga (supun.setunga@dapperlabs.com) |
| **Sponsor**    | Supun Setunga (supun.setunga@dapperlabs.com) |
| **Updated**    | 2022-11-09                                   |

## Objective

This proposal tightens and improves the mutability restrictions introduced in
[Cadence mutability restrictions FLIP](https://github.com/onflow/flips/blob/main/flips/20211129-cadence-mutability-restrictions.md).
It extends the existing restrictions of array-typed and dictionary-typed `let` fields to also apply to
composite-typed `let` fields. With this change, Cadence will issue an error on an attempt to modify a composite-typed
`let` field by updating a nested field.

## Motivation

A previous version of cadence (secure cadence release) restricted the potential foot-gun of mutating container-typed
`let` fields/variables via the
[Cadence mutability restrictions FLIP](https://github.com/onflow/flips/blob/main/flips/20211129-cadence-mutability-restrictions.md).

However, these existing restrictions only apply to array-typed and dictionary-typed fields/variables.
Developers may still run into a foot-gun where a composite-typed field/variable declared with `let` keyword may still
get mutated, by mutating the nested fields.

```cadence
struct Foo {
    let bar: Bar
}

struct Bar {
    var id: Int
}

fun test() {
    var foo = Foo()

    // Even though the `bar` field of `foo` is declared with `let` keyword,
    // the nested `id` field can be updated, resulting `bar` field being updated.
    foo.bar.id = 4
}
```

## User Benefit

Suggested changes of this proposal reduce the potential of users accidentally exposing mutable fields,
thinking they are constants.

## Design Proposal

This proposal suggests making `let` variables behave more similarly to constants, by making the
`let` keyword to mark **both** the **variable** and the **value** as immutable.

```cadence
struct Foo {
    let bar: Bar
}

struct Bar {
    var a: Int
    let b: Int
}

fun test() {
    var foo = Foo()

    // All fields of `foo.bar` would be visible as if they are constants.
    // i.e: accessing a mutable field through a non-mutable variable/value would be prohibited.

    // This will be a static error because even though `a` is not defined with `let`,
    // it is accessed via `bar` field which is defined with `let`.
    foo.bar.a = 4

    // This is already a static error because `b` is defined with `let`.
    foo.bar.b = 5
}
```

This can be achieved by restricting the mutation of a mutable member that is accessed via an immutable member.
For eg: in a member access expression `v.x.y.z` or `v[x][y][z]`, even if member `z` is mutable,
if at least one of the members in the chain prior to `z` (i.e: `x` or `y`) is immutable,
then treat `z` also as immutable.

The only exception is the mutation of an array/dictionary-typed immutable member which is owned by the enclosing type.
This is already supported - so there won't be any change to the existing behavior.

e.g:
```cadence
struct Foo {
    pub let bar: [Int]

    fun mutate() {
        self.bar[0] = 4    // Valid, because `bar` belongs to the enclosing type. i.e: same scope
    }
}
```

### Mutating functions

#### Built-in functions

Built-in container functions are already annotated/marked as mutating or not.
Using these annotations and the modifier of fields, the type checker can determine whether calling a particular
function is allowed on a given field.

#### User functions

Currently, user functions have no mechanism to be marked as mutating or not.
Hence, functions defined in composite types that mutate the receiver are allowed to be called on a field value,
regardless of whether the field is defined with `let` or `var`.
This proposal doesn't suggest any changes to this behavior.

However, it may be required to change this to further tighten up and improve the mutability restrictions.
Refer to the [Questions and Discussion Topics](#questions-and-discussion-topics) section for more details.

### Dynamic checks

The restrictions suggested in the above sections can be enforced by the type checker.
For the specific problem addressed in this proposal, it would not require to have any dynamic checks at the interpreter.
This is because:
  - Concrete-type implementations cannot change the field type or the access modifier of the conforming interface.
  - Even though it is possible for fields to dynamically dispatch, the modifier is always known statically.
  - So it is always possible to statically determine if a mutable field is modified via an immutable field.

However, as described initially, marking the 'value' as immutable leaves room for adding dynamic
checks in the future if required. Whether such checks are required or not is out of the scope of this proposal.

### Drawbacks

It is important to note that, this proposed solution would not prevent all possible types of mutations
(refer [Questions and Discussion Topics](#questions-and-discussion-topics) section for more details),
but rather only solves a potion of it.

### Alternatives Considered

None

### Performance Implications

This proposes an additional static check to the type checker.
The performance implication would be negligible.

### Dependencies

None

### Engineering Impact

This change is relatively simple in the implementation.

### Compatibility

This change introduces some additional restrictions. Hence, it is backward incompatible.

### User Impact

Some existing contracts may become invalid with the newly added restrictions.
Developers will have to update their smart contracts to match the new type checking rules.

## Prior Art

This has the same prior art mentioned in https://github.com/onflow/flips/blob/main/flips/20211129-cadence-mutability-restrictions.md

## Questions and Discussion Topics

As mentioned in the [User functions](#user-functions), methods defined in composite types can also modify the
receiver value.

```cadence
struct Foo {
    let bar: Bar
}

struct Bar {
    var a: Int

    pub fun incrementA() {
        self.a = self.a + 1
    }
}

fun test() {
    var foo = Foo()

    // Static error
    foo.bar.a = 4

    // Not an error
    foo.bar.incrementA()
}
```

Like in the above example, it is currently allowed to mutate a nested composite value through a mutating function,
even if the field `bar` was defined using the `let` keyword.

Restricting such mutations would require a more complicated solution that would severely impact existing behavior,
and would need further discussions.
