---
status: draft
flip: 357
authors: Bastian Müller (bastian.mueller@flowfoundation.org)
updated: 2026-02-09
---

# FLIP 357: Add minOf and maxOf Functions to Cadence

## Objective

Add `minOf` and `maxOf` functions to the Cadence standard library,
providing a convenient way to find the minimum or maximum of two comparable values.

## Motivation

Finding the minimum or maximum of two values is a common operation in smart contract development.
Currently, developers must implement this logic manually using conditional expressions:

```cadence
// Current approach: manual comparison
let smaller = a < b ? a : b
let larger = a > b ? a : b
```

While this works, it has several drawbacks:
- **Verbose**: Requires writing the comparison logic repeatedly
- **Error-prone**: Easy to make mistakes with the comparison operators or the ternary expression
- **Less readable**: The intent isn't immediately clear, especially in complex expressions

Other programming languages provide built-in or standard library functions for this common operation.
Having standard functions improves code readability and reduces the likelihood of errors.

## User Benefit

The `minOf` and `maxOf` functions provide several benefits:

**Improved Readability**: The intent is immediately clear from the function name:
```cadence
let price = minOf(bidPrice, maxPrice)  // Clearer than: bidPrice < maxPrice ? bidPrice : maxPrice
```

**Reduced Errors**: Eliminates the risk of swapping comparison operators or ternary branches,
or accidentally comparing the wrong variables.

**Type Safety**: The functions work with any comparable type and ensure both arguments have the same type,
catching type mismatches at compile time.

**Consistency**: Provides a standard way to perform these operations across all Cadence codebases.

## Design Proposal

Add two generic functions to the Cadence standard library that work with any comparable type.

### Function Signatures

```cadence
/// Returns the minimum of two values
///
/// The arguments must be of the same comparable type.
///
/// Examples:
///   minOf(5, 10) == 5
///   minOf(10, 5) == 5
///   minOf(1.5, 2.5) == 1.5
///   minOf("apple", "banana") == "apple"
///
access(all) fun minOf<T>(_ a: T, _ b: T): T

/// Returns the maximum of two values
///
/// The arguments must be of the same comparable type.
///
/// Examples:
///   maxOf(5, 10) == 10
///   maxOf(10, 5) == 10
///   maxOf(1.5, 2.5) == 2.5
///   maxOf("apple", "banana") == "banana"
///
access(all) fun maxOf<T>(_ a: T, _ b: T): T
```

### Type Requirements

The type parameter `T` must be a **comparable type**.
In Cadence, comparable types are those that support comparison operators (`<`, `>`, `<=`, `>=`),
e.g. all concrete number types, strings, characters, booleans, and addresses.

### Naming Rationale

The functions are named `minOf` and `maxOf` (rather than `min` and `max`) to avoid naming conflicts:
- Number types already have `min` and `max` fields,
  which represent the minimum and maximum values of that type, e.g. `Int8.min` is the minimum value of `Int8`
- Existing contracts may have defined `min` and `max` variables or functions,
  so using `minOf` and `maxOf` avoids potential conflicts.
  As of 2026-02-09, there are no global functions named `minOf` or `maxOf`, so this naming is safe.
- Kotlin uses the names `minOf` and `maxOf`, see e.g. https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.comparisons/min-of.html

## Drawbacks

**Limited to Two Arguments**: The functions only accept two arguments. Finding the minimum or maximum
of more values requires chaining:

```cadence
let min = minOf(minOf(a, b), c)
```

A future enhancement could add variadic versions that accept more than two arguments.

## Alternatives Considered

### Alternative 1: Methods

Add `min(_ other: T)` and `max(_ other: T)` methods to comparable types `T`:

```cadence
a.min(b)  // Returns the minimum of a and b
a.max(b)  // Returns the maximum of a and b
```

**Pros:**
- Intuitive method syntax
- No naming conflict

**Cons:**
- Requires many separate implementations (one per comparable type)

### Alternative 2: Static Methods

Add `minimum()` and `maximum()` methods to each comparable type:

```cadence
Int8.minimum(5, 10)
UFix64.maximum(1.5, 2.5)
```

**Pros:**
- More discoverable through type-specific documentation
- No naming conflict

**Cons:**
- More verbose
- Requires many separate implementations (one per comparable type)
- Users must remember type-specific methods instead of a single global function

## Performance Implications

The functions are implemented as native functions,
so they will have minimal overhead compared to manual comparisons.

## Compatibility

This is a purely additive feature.

However, to ensure no breaking changes to existing contracts,
the proposal currently assumes that existing contracts do not declare variables or functions named
`minOf` or `maxOf`, which is true as of 2026-02-09.
If a conflict is detected, we may need to consider alternative names,
or an alternative design such as methods on comparable types.

## Prior Art

Nearly all major programming languages provide minimum/maximum functions:

**Python**:

Global functions that work with any comparable type:

```python
min(a, b)
max(a, b)
```

**JavaScript**:

Functions that only work with numbers, not with strings or other comparable types:

```javascript
Math.min(a, b)
Math.max(a, b)
```

**Rust**:

Methods on comparable types:

```rust
a.min(b)
a.max(b)
```

**Swift**:

Global functions that work with any comparable type:

```swift
min(a, b)
max(a, b)
```

**Kotlin**:

Global functions that work with any comparable type:

```kotlin
minOf(a, b)
maxOf(a, b)
```

Cadence's design follows the Kotlin naming convention, which also distinguishes comparison functions
from value range constants.

## Implementation

An implementation is available at: https://github.com/onflow/cadence/pull/4430

## Related Issues

None

## Questions and Discussion Topics

1. Should we rather use method syntax instead of global functions, or static methods on each comparable type?
2. Should we add variants that accept more than two arguments in the future?
3. Should we add `clamp(value, min, max)` as a related utility function?
