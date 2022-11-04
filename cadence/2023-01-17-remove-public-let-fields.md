**# Improve Mutability Restrictions

| Status         | Proposed                                     |
|:---------------|:---------------------------------------------|
| **FLIP #**     |                                              |
| **Author(s)**  | Supun Setunga (supun.setunga@dapperlabs.com) |
| **Sponsor**    | Supun Setunga (supun.setunga@dapperlabs.com) |
| **Updated**    | 2023-01-17                                   |

## Objective

This proposal suggest to limit the scope of `let` fields to `priv`, so that potential foot-guns from mutability
of such fields are eliminated.

## Motivation

A previous version of cadence (secure cadence release) restricted the potential foot-gun of mutating container-typed
`let` fields/variables via the
[Cadence mutability restrictions FLIP](https://github.com/onflow/flips/blob/main/flips/20211129-cadence-mutability-restrictions.md).

However, there are still ways to mutate such fields by:
- Directly mutating a nested composite typed field.
- Mutating a field by calling a mutating function on the field.
- Mutating the field via a reference.

### Mutating nested composite typed fields

The existing nested field mutation restrictions only apply to array-typed and dictionary-typed fields/variables.
Developers may still run into a foot-gun where a composite-typed field/variable declared with `let` keyword gets mutated
by mutating the nested fields of the composite value.

```cadence
struct Foo {
    pub let bar: Bar
}

struct Bar {
    pub var id: Int
}

fun test() {
    var foo = Foo()

    // Even though the `bar` field of `foo` is declared with `let` keyword,
    // the nested `id` field can be updated, resulting `bar` field being updated.
    foo.bar.id = 4
}
```

### Mutating through a function

Methods defined in composite types can also modify the receiver value.

```cadence
pub struct Foo {
    pub let bar: Bar
}

pub struct Bar {
    pub let array: [Int]

    pub fun mutateBar() {
        self.array.append(0)
    }
}

pub fun test() {
    let foo = Foo()

    // Error: Directly mutating the nested field is restricted.
    foo.bar.array.append(0)

    // However, mutating the nested `let` field via a function is allowed.
    foo.bar.mutateBar()
}
```

Like in the above example, it is currently allowed to mutate a nested composite value through a mutating function,
even if the field `bar` was defined using the `let` keyword.

### Mutating via a reference

It is also possible to mutate a container typed field, by taking a reference to it and then mutating the field
through that reference.

```cadence
pub struct Foo {
    pub let array: [Int]
}

pub fun test() {
    let foo = Foo()

    // Error: Directly mutating the field is restricted.
    foo.array.append(0)

    // However, mutating the field via a reference to the field is not restricted.
    let arrayRef = &foo.array as &[Int]
    arrayRef.append(0)
}
```

## User Benefit

Suggested changes of this proposal eliminate the potential of users accidentally exposing mutable fields,
thinking they are constants.

## Design Proposal

This proposal suggest to remove the support for public `let` fields, and only support private (`priv`) `let` fields.
This also eliminates the requirement for the restriction introduced in the previous [Cadence mutability restrictions FLIP](https://github.com/onflow/flips/blob/main/flips/20211129-cadence-mutability-restrictions.md).

Assuming only `priv let` field are supported, let's revisit the problem cases.

### Mutating nested composite typed fields

Nested composite fields are no longer accessible via a `let` field, as they are now private.

```cadence
struct Foo {
    priv let bar: Bar
}

struct Bar {
    pub var id: Int
}

fun test() {
    var foo = Foo()

    // Static error: `bar` is not accessible from here.
    foo.bar.id = 4
}
```

### Mutating through a function

Similar to directly updating `let` fields from outside, calling functions on `priv let` fields are restricted to outside.
Hence, potential for accidentally exposing mutable functions to outsiders is now eliminated.

```cadence
pub struct Foo {
    priv let bar: Bar
    
    pub fun mutateFoo() {
        self.bar.mutateBar()
    }
}

pub struct Bar {
    priv let array: [Int]

    pub fun mutateBar() {
        self.array.append(0)
    }
}

pub fun test() {
    let foo = Foo()

    // Static error: `bar` is not accessible from here.
    foo.bar.array.append(0)

    // Static error: `bar` is not accessible from here.
    foo.bar.mutateBar()
    
    // However, it is possible to mutate through delegation.
    foo.mutateFoo()
}
```

However, as in the above example, author of the type can opt in to exposing the mutable functions to outsiders,
by adding a delegation function.

### Mutating through references

Similar to above, references cannot be taken to `let` fields from outside scopes anymore.
Author of the type can still provide mutable references to outsiders via functions, but that would be considered safe, 
since it is done by the author intentionally, and is no longer 'accidental'.

```cadence

pub struct Foo {
    priv let array: [Int]
    
    pub fun getArrayRef() &[Int] {
        return &self.array as &[Int]
    }
}

pub fun test() {
    let foo = Foo()

    // Static error: `array` is not accessible from here.
    let arrayRef = &foo.array as &[Int]

    // Valid: Reference is given with the author's consensus. So it is not 'accidental'.
    let arrayRef = foo.getArrayRef()
    arrayRef.append(0)
}
```

### Drawbacks

- Developers can no longer expose `let` fields to outsiders. If they want outsiders to access `let` fields, then that 
  has to be done via delegation functions (i.e: getters/setters)
  However, having `let` fields expose to outside scopes is the root cause of the problem.
  Hence, this is a reasonable compromise for safety. 

### Alternatives Considered

- An alternative solution is proposed in [Improve Cadence external mutability restrictions FLIP](https://github.com/onflow/flips/pull/58).
  However, the restrictions suggested in that FLIP could affect the usability of Cadence severely.

### Performance Implications

This proposes an additional static check to the type checker, which has no performance impact.

### Dependencies

None

### Engineering Impact

This change is relatively easier to implement.

### Compatibility

This change introduces some additional restrictions. Hence, it is backward incompatible.

### User Impact

Some existing contracts may become invalid with the newly added restrictions.
Developers will have to update their smart contracts to match the new type-checking rules.

## Prior Art

This has the same prior art mentioned in https://github.com/onflow/flips/blob/main/flips/20211129-cadence-mutability-restrictions.md

## Questions and Discussion Topics

None
