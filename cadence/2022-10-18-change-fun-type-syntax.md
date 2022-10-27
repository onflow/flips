# Change the syntax for function types.

| Status        | (Proposed / Rejected / Accepted / Implemented)       |
:-------------- |:---------------------------------------------------- |
| **FLIP #**    | [43](https://github.com/onflow/flips/pull/43)        |
| **Author(s)** | Naomi Liu (naomi.liu@dapperlabs.com)                 |
| **Updated**   | 2022-10-27                                           |

## Objective

Modify/add syntax for representing function types.

## Motivation

Currently, functions are declared and referred to with differing syntax. For instance,

```cadence
// function declaration
fun foo(_ x: Int): String {...}
// function expression bound to a name
let bar = fun(x: Int): String {...}

// a reference to a function
let baz: ((Int): String) = foo
```

This convention for representing functions gets even more confusing when currying or higher-order functions are brought into the mix:

```cadence
// cadence type: ((Int): ((Int): ((Int): Int)))
// pretty type: Int -> Int -> Int -> Int

fun add3(_ x: Int): ((Int): ((Int): Int)) {
    return fun(y: Int): ((Int): Int) {
        return fun(z: Int): Int {
            return x + y + z
        }
    }
}

// cadence type: (([Int], ((Int): String)): [String])
// pretty type: [Int] -> (Int -> String) -> [String]
fun mapIntString(list: [Int], f: ((Int): String)): [String] {...}
```

Another extreme example is the type `(({Int: String}): {Int: Int})`. To a reader who isn't familiar with Cadence, it's not immediately obvious that this type means "a function taking a dictionary of `Int`s to `String`s, returning a dictionary of `Int`s to `Int`s".


## User Benefit

The existing syntax is inconsistent and unfamiliar, ultimately harming readability of Cadence code. Users would benefit from type signatures with clearer intent.

## Design Proposal

Rename function types to require the `fun` keyword.

Given that `fun` is already a reserved keyword for declaring functions, requiring the keyword in the type signature of a function is more consistent with the existing syntax. For instance, 

    ```cadence
    // (AnyStruct -> Void) -> AnyStruct -> Void
    fun call(f: ((AnyStruct): Void), x: AnyStruct) {...}

    // after
    fun call(f: (fun(AnyStruct): Void), x: AnyStruct) {...}
    ```

This could be implemented as a non-breaking change by still allowing the old syntax of omitting `fun`, but preferring the keyword in new contracts and transactions. The stringification of function types should be updated to use the new syntax.

### Drawbacks

As @turbolent mentioned in an internal discussion, there is some ambiguity in the language grammar regarding the distinction between functions and restricted types. This is what motivated the choice to use the existing syntax without the `fun` keyword.

For example,

```cadence
fun(): T{}
```

Is this a function that returns a `T` but has an empty body? Or does it return a restricted type `T{}` and is missing its body?

One potential route is to give functions a higher priority in the parser than restricted types, so the above expression is parsed as a function returning  `T` with an empty body. Let's take a more complex example:

```cadence
interface Foo {...}

let f = fun(): AnyStruct{Foo}
```

In this case, `f` can be parsed either as a function returning `AnyStruct` whose body only contains the expression `Foo`, or as a function returning a restricted type `AnyStruct{Foo}` but is missing a body. Assume that we prefer to follow the first parsing rule. Since the "body" of the function is missing an explicit `return`, its return type is inferred to be `Void`. However, the stray `Foo` exists in the type namespace instead of the value namespace, so it will result in an unknown name error unless a variable with the same name exists in scope. A lint could be added that ensures statements are either function calls, assignments, declarations, or moves. For instance,

```
var x = 0 // declaration, ok
fun f(): Int {log("hi"); return 42} // declaration, ok
x = f() // assignment, ok
f() // function call with unused return, ok
x = x + 1 // ok
x // noop, should throw a lint error
```

Additionally, if the user intends to write a restricted type with multiple interfaces like `AnyStruct{Foo, Bar}`, the parser will throw an error at the use of bare commas outside of a function call or composite literal. It's akin to writing an expression `Foo, Bar`. It seems that the only cases where this ambiguity doesn't throw an error is when the unrestricted return type is parsed as a supertype of `Void`, and the restricting interface is shadowed by a value. 

### Alternatives Considered

An alternative to using `fun` to denote function types is to introduce another keyword, such as `=>` or `->`. This would be more familiar to users coming from languages such as Javascript or Haskell, while remaining as a non-breaking change due to these operators being unreserved currently. For example, 

```cadence
// before
fun call(f: ((AnyStruct): Void), x: AnyStruct) {...}
let _call: (((AnyStruct): Void, AnyStruct): Void) = call

// after
fun call(f: AnyStruct -> Void, x: AnyStruct): Void {...}
let _call: (AnyStruct -> Void, AnyStruct) -> Void = call
```

The main benefit of this alternative syntax would be the unambiguous parsing and lower visual noise compared to using `fun`. This would still introduce another inconsistency between how functions are declared and how they're typed, but it's more intuitive at a glance to unfamiliar readers. Introducing function arrows in this way also opens the door to an alternative syntax for denoting closures that doesn't require an annotated return type or `return` statement, which could be useful for simple one-liner functions.

### Performance Implications

Function types written with either syntax will create identical parse trees, so the performance impact is expected to be limited to another conditional branch in the parser.

### Engineering Impact

* Do you expect changes to binary size / build time / test times?
* Who will maintain this code? Is this code in its own buildable unit?
Can this code be tested in its own?
Is visibility suitably restricted to only a small API surface for others to use?

Given that first-class functions are a very small part of onchain contracts and the standard library in its current form, no significant impacts should occur. Any required changes are expected to be unit-testable and relatively encapsulated. 

### Best Practices

* Does this proposal change best practices for some aspect of using/developing Flow?
How will these changes be communicated/enforced?

If we decide to still allow the old syntactic forms, then the new syntax should be reocommended for any new code.

### Tutorials and Examples

* If design changes existing API or creates new ones, the design owner should create
end-to-end examples (ideally, a tutorial) which reflects how new feature will be used.

Tutorials and documentation should be updated to use the new syntax.

### Compatibility

* Does the design conform to the backwards & forwards compatibility [requirements](../docs/compatibility.md)?
* How will this proposal interact with other parts of the Flow Ecosystem?
    - How will it work with FCL?
    - How will it work with the Emulator?
    - How will it work with existing Flow SDKs?

This can be implemented as a non-breaking change, requiring no intervention from developers to update existing code.

### User Impact

* What are the user-facing changes? How will this feature be rolled out?

Function types will be printed using the new syntax in the REPL, error messages, and language server. No migrations or inverventions are needed, and this change can be rolled out with any spork before, during, or after Stable Cadence's release.

## Related Issues

What related issues do you consider out of scope for this proposal,
but could be addressed independently in the future?

If we use a syntax other than `fun` for denoting function types, such as an arrow `->`, the new keyword could be used to implement a shorthand for declaring closures.

## Prior Art

Most languages with first-class functions use similar language for referring to those functions. For instance,

```scala
// scala
val f: Int => Int = {n => n * 2}
```

```haskell
-- haskell
let f: Int -> Int = \n -> n * 2
```

```swift
// swift
// cadence's existing syntax is similar,
// but we use the colon (:) in other places leading to confusion
let f: (Int) -> Int = func(_ n: Int) -> Int {
    return n * 2
}
```

```typescript
// typescript
let f: (n: number) => number = n => n * 2;
```

## Questions and Discussion Topics

Is the existing syntax fine, i.e. do people actually care?

Is the issue large enough to warrant adding another keyword to the language?

Will adding the `fun` keyword lead to intractable parse ambiguities or break existing contracts?

Will changing the syntax to use an operator improve readability, or just introduce another source of inconsistency to the language?