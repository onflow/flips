# Change the syntax for function types.

| Status        | Accepted                                             |
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
let bar = fun (x: Int): String {...}

// a variable declaration with a function value
// note the lack of the `fun` keyword when referring to its type
let baz: ((Int): String) = foo
```

This convention for representing functions gets even more confusing when higher-order functions are brought into the mix:

```cadence
// cadence type: ((Int): ((Int): ((Int): Int)))
// pretty type: Int -> Int -> Int -> Int

fun add3(_ x: Int): ((Int): ((Int): Int)) {
    return fun (y: Int): ((Int): Int) {
        return fun (z: Int): Int {
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
    fun apply(f: ((AnyStruct): Void), x: AnyStruct) {...}

    // after
    fun apply(f: fun (AnyStruct): Void, x: AnyStruct) {...}
    ```

This could be implemented as a non-breaking change by still allowing the old syntax of omitting `fun`, but preferring the keyword in new contracts and transactions. The stringification of function types should be updated to use the new syntax.

For type signatures ascribed to variables, we currently rely on the colon `:` token in order to parse the type as a function. This requires the type to be wrapped up in parentheses to avoid parsing ambiguity. Reusing a previous example:

```cadence
let bar = fun (x: Int): String {...}

// a variable declaration with a function value
let baz: ((Int): String) = foo
```

If we continued to elide the `fun` keyword in function types, parentheses could still be eliminated by making `:` a right-associative operator with a higher precedence than `=`. 

Currently, a function type signature is defined by the following grammar:

```ebnf
functionType
    : '(' 
         '(' ( typeAnnotation ( ',' typeAnnotation )* )? ')'
         ':' typeAnnotation
      ')'

(* example: ((Int, @AnyResource): @AnyResource) *)
```

Where the surrounding parentheses are required. By instead treating `:` as an operator, the rule is simplified to:

```ebnf
functionType
    : '(' ( typeAnnotation ( ',' typeAnnotation )* )? ')'
      ':' typeAnnotation
```

At the same time however, how do we express a function that returns `Void` (`()`) with this syntax? We currently allow authors to omit a return type in function declarations, which causes the type to default to `Void`. Typing the function still requires one to explicitly name `Void`:

```cadence
fun noop() {} // return type is inferred to be `Void`

let noop_: ((): Void) = noop // return type is required because we use `:` as a marker token
```

Using the `fun` keyword would simplify parsing rules and also allow for an author to omit the return type when writing signatures for procedures, as we already permit when declaring them.

```cadence
let noop_: fun() = noop // unambiguous, since `fun` is our marker now instead of `:`

fun apply2(f: fun (Int), g: fun (Int), x: Int) {...}

let baz: fun (Int): String = bar
```

The adjusted grammar would only affect `functionType`:

```ebnf
functionType:
    'fun' '(' ( typeAnnotation ( ',' typeAnnotation )* )? ')' 
         ( ':' typeAnnotation )?
```

### Drawbacks

To the best of my knowledge, this change should not introduce any ambiguities into the language grammar. There is currently an ambiguous case for function expressions that return restricted types:

```cadence
interface Foo {...}

let f = fun (): AnyStruct{Foo}
```

In this case, it is unclear to the parser whether `f` is a function that returns a restricted `AnyStruct{Foo}` without a body, or if it returns an unrestricted `AnyStruct` and whose body is the single expression `Foo`. We currently step around this issue by requiring restricted types' curly brackets to follow immediately after the parent type without any whitespace:

```cadence
fun (): AnyStruct{Foo} // return type is a restricted type, no body
fun (): AnyStruct {Foo} // return type is AnyStruct, body is Foo
```

This ambiguity does not extend to types however, and is outside the scope of this FLIP. Using the proposed syntax, we could still write

```cadence 
let f: fun (): AnyStruct{Foo} = fun (): AnyStruct{Foo} {...}
```

### Alternatives Considered

An alternative to using `fun` to denote function types is to introduce another keyword, such as `=>` or `->`. This would be more familiar to users coming from languages such as JavaScript, Swift, and Kotlin, while remaining as a non-breaking change due to these operators being unreserved currently. For example, 

```cadence
// before
fun apply(f: ((AnyStruct): Void), x: AnyStruct): Void {...}
let _apply: (((AnyStruct): Void, AnyStruct): Void) = apply

// proposed syntax with `fun`
fun apply(f: fun (AnyStruct): Void, x: AnyStruct): Void {...}
let _apply: fun (fun (AnyStruct): Void, AnyStruct): Void = apply

// alternative syntax with '->'
fun apply(f: AnyStruct -> Void, x: AnyStruct): Void {...}
let _apply: (AnyStruct -> Void, AnyStruct) -> Void = apply
```

The main benefit of this alternative syntax would be the unambiguous parsing and lower visual noise compared to using `fun`. This would still introduce another inconsistency between how functions are declared and how they're typed, but it's more intuitive at a glance to unfamiliar readers. Introducing function arrows in this way also opens the door to an alternative syntax for denoting closures that doesn't require an annotated return type or `return` statement, which could be useful for simple one-liner functions.

Introducing a separate operator for return types still does not eliminate the previously-mentioned ambiguity in parsing function expressions, however:

```cadence
// still ambiguous, but looks cleaner because ':' in declarations 
// now only indicates a type assignment to a variable
let f: fun () -> AnyStruct{Foo} = fun () -> AnyStruct{Foo} 
```

### Performance Implications

Function types written with either syntax will create identical parse trees, so the performance impact is expected to be limited to another conditional branch in the parser.

### Engineering Impact

Given that first-class functions are a very small part of onchain contracts and the standard library in its current form, no significant impacts should occur. Any required changes are expected to be unit-testable and relatively encapsulated. 

### Best Practices

If we decide to still allow the old syntactic forms, then the new syntax should be reocommended for any new code.

### Tutorials and Examples

Tutorials and documentation should be updated to use the new syntax.

### Compatibility

This can be implemented as a non-breaking change, requiring no intervention from developers to update existing code.

### User Impact

Function types will be printed using the new syntax in the REPL, error messages, and language server. If we still continue to allow the old syntax, then no migrations or inverventions are needed and this change can be rolled out independently of Stable Cadence.
## Related Issues

If we use a syntax other than `fun` for denoting function types, such as an arrow `->`, the new keyword could be used to implement a shorthand for declaring closures.

## Prior Art

Most languages with first-class functions use similar language for referring to those functions. For instance,

```scala
// scala
val f: Int => Int = {n => n * 2}
```

```haskell
-- haskell
let f :: Int -> Int 
    f = \n -> n * 2
```

```swift
// swift
// note that using the arrow allows for the `func` keyword 
// to be omitted from function types
let f: (Int) -> Int = func(_ n: Int) -> Int {
    return n * 2
}
```

```typescript
// typescript
let f: (n: number) => number = n => n * 2;
```

```go
// go
var f func(int) int = func(int) int {
    return 2
}
```

In the case of omitting a return type in the signatures of procedures, the main distinction between languages that provide this feature and languages that don't, is whether they denote function types with an infix arrow (`->`) or a prefixed keyword (`fun`, `func`, `fn`, etc). Some languages allow the return type to be omitted in a function's declaration, but still require it to be annotation in its corresponding type signature.

For instance:

```scala
// scala
def noop() = {}
// with return type:
def noop(): Unit = {}
// type written out
val noop_: () => Unit = noop
```

```haskell
-- haskell
noop :: () -> ()
noop _ = ()
```

```rust
// rust
fn noop() {}
// with return type
fn noop() -> () {}
// type written out
let noop_: fn() = noop
// the return type can be explicitly supplied
let noop_: fn() -> () = noop
```

```swift
// swift
func noop() {}
// with return type:
func noop() -> Void {}
// type written out
let noop_: () -> Void = noop
```

```go
// go
func noop() {}
// the only way to denote procedures in golang is to omit a return type
var noop_ func() = noop
```

## Questions and Discussion Topics

Is the existing syntax fine, i.e. do people actually care?

Is the issue large enough to warrant adding another keyword to the language?

Will adding the `fun` keyword lead to intractable parse ambiguities or break existing contracts?

Will changing the syntax to use an operator improve readability, or just introduce another source of inconsistency to the language?