---
status: draft
flip: 357
authors: Bastian Müller (bastian.mueller@flowfoundation.org)
updated: 2026-02-17
---

# FLIP 357: Add Comparison Functions to Cadence

## Objective

Add `min`, `max`, and `clamp` functions to a new `Comparison` contract in the Cadence standard library,
providing a convenient way to find the minimum or maximum of two comparable values,
or to clamp a value within a range.

## Motivation

Finding the minimum or maximum of two values, or clamping a value to a range,
are common operations in smart contract development.
Currently, developers must implement this logic manually using conditional expressions:

```cadence
// Current approach: manual comparison
let smaller = a < b ? a : b
let larger = a > b ? a : b
let clamped = value < low ? low : (value > high ? high : value)
```

While this works, it has several drawbacks:
- **Verbose**: Requires writing the comparison logic repeatedly
- **Error-prone**: Easy to make mistakes with the comparison operators or the ternary expression
- **Less readable**: The intent isn't immediately clear, especially in complex expressions

Other programming languages provide built-in or standard library functions for these common operations.
Having standard functions improves code readability and reduces the likelihood of errors.

## User Benefit

The `min`, `max`, and `clamp` functions provide several benefits:

**Improved Readability**: The intent is immediately clear from the function name:
```cadence
import Comparison

let price = min(bidPrice, maxPrice)  // Clearer than: bidPrice < maxPrice ? bidPrice : maxPrice
let fee = clamp(computedFee, min: minFee, max: maxFee)  // Clearer than: computedFee < minFee ? minFee : (computedFee > maxFee ? maxFee : computedFee)
```

**Reduced Errors**: Eliminates the risk of swapping comparison operators or ternary branches,
or accidentally comparing the wrong variables.

**Type Safety**: The functions work with any comparable type and ensure all arguments have the same type,
catching type mismatches at compile time.

**Consistency**: Provides a standard way to perform these operations across all Cadence codebases.

## Design Proposal

Add three generic functions to a new `Comparison` contract in the Cadence standard library
that work with any comparable type.

### Usage

```cadence
import Comparison

let smaller = min(a, b)
let larger = max(a, b)
let bounded = clamp(value, min: low, max: high)
```

### Function Signatures

```cadence
/// Returns the minimum of two values.
///
/// The arguments must be of the same comparable type.
///
/// Examples:
///   min(5, 10) == 5
///   min(10, 5) == 5
///   min(1.5, 2.5) == 1.5
///   min("apple", "banana") == "apple"
///
access(all) fun min<T>(_ a: T, _ b: T): T

/// Returns the maximum of two values.
///
/// The arguments must be of the same comparable type.
///
/// Examples:
///   max(5, 10) == 10
///   max(10, 5) == 10
///   max(1.5, 2.5) == 2.5
///   max("apple", "banana") == "banana"
///
access(all) fun max<T>(_ a: T, _ b: T): T

/// Returns the value clamped to the inclusive range [min, max].
///
/// If the value is less than min, min is returned.
/// If the value is greater than max, max is returned.
/// Otherwise, the value itself is returned.
/// The arguments must be of the same comparable type.
///
/// Examples:
///   clamp(7, min: 1, max: 10) == 7
///   clamp(0, min: 1, max: 10) == 1
///   clamp(20, min: 1, max: 10) == 10
///
access(all) fun clamp<T>(_ value: T, min: T, max: T): T
```

### Type Requirements

The type parameter `T` must be a **comparable type**.
In Cadence, comparable types are those that support comparison operators (`<`, `>`, `<=`, `>=`),
e.g. all concrete number types, strings, characters, booleans, and addresses.

### Naming and Location Rationale

The functions are named `min` and `max` and placed in a `Comparison` contract
(rather than being global functions) to avoid naming conflicts:
- Number types already have `min` and `max` fields,
  which represent the minimum and maximum values of that type,
  e.g. `Int8.min` is the minimum value of `Int8`
- This provides better compatibility with existing code:
  Existing contracts may have defined `min` and `max` variables.
  Placing these functions inside the `Comparison` contract avoids conflicts,
  since users only bring them into scope by explicitly importing the contract.

## Drawbacks

**Requires Import**: Unlike some other standard library functions,
`min`, `max`, and `clamp` require an explicit `import Comparison` statement.
This is a minor inconvenience but necessary to avoid naming conflicts with existing code.

**Limited to Two Arguments for min/max**: The `min` and `max` functions only accept two arguments.
Finding the minimum or maximum of more values requires chaining:

```cadence
import Comparison

let minimum = min(min(a, b), c)
```

A future enhancement could add variadic versions that accept more than two arguments.

## Alternatives Considered

### Alternative 1: Global Functions Named `minOf`/`maxOf`

Use `minOf` and `maxOf` as global standard library functions (no import required):

```cadence
let smaller = minOf(a, b)
let larger = maxOf(a, b)
```

**Pros:**
- No import required
- Kotlin uses these names, see e.g. https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.comparisons/min-of.html

**Cons:**
- Less familiar naming than `min`/`max`
- Still adds global names that could shadow existing declarations

This was the initial design but was superseded in favor of `min`/`max` in the `Comparison` contract.

### Alternative 2: Methods

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

### Alternative 3: Static Methods

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
The functions are scoped to the `Comparison` contract,
so they only affect code that explicitly imports it,
ensuring no breaking changes to existing contracts.

## Prior Art

Nearly all major programming languages provide minimum/maximum/clamp functions:

**Python**:

Global `min`/`max` functions that work with any comparable type.
No built-in `clamp`, but `max(low, min(value, high))` is the idiomatic form.

```python
min(a, b)
max(a, b)
```

**JavaScript**:

`min`/`max` functions that only work with numbers.
No built-in `clamp`.

```javascript
Math.min(a, b)
Math.max(a, b)
Math.max(low, Math.min(value, high))  // clamp idiom
```

**Rust**:

Methods on comparable types, including `clamp`:

```rust
a.min(b)
a.max(b)
a.clamp(min, max)
```

**Swift**:

Global `min`/`max` functions that work with any comparable type.
`clamp` is available on `Comparable` types as `clamped(to:)`:

```swift
min(a, b)
max(a, b)
value.clamped(to: low...high)
```

**Kotlin**:

Global functions that work with any comparable type, including `coerceIn` for clamping:

```kotlin
minOf(a, b)
maxOf(a, b)
value.coerceIn(low, high)
```

Cadence's design uses the familiar `min`/`max`/`clamp` naming (closest to Rust),
while scoping the functions to a `Comparison` contract to avoid naming conflicts.

## Implementation

An implementation is available at: https://github.com/onflow/cadence/pull/4430

## Related Issues

None

## Questions and Discussion Topics

1. Should we rather use method syntax instead of global functions, or static methods on each comparable type?
2. Should we add variadic variants of `min` and `max` that accept more than two arguments in the future?
