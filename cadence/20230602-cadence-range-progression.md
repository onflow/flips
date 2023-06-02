---
status: draft
flip: 96
authors: darkdrag00n (darkdrag00n@proton.me)
sponsor: Bastian Müller (bastian@dapperlabs.com)
updated: 2023-06-02
---

# FLIP 96: Range & Progression Type

## Objective

This FLIP proposes adding the `Range` & `Progression` types to Cadence.
Both of these types can then used inside a `for-in` loop.

## Motivation

Modern langauges often provide a concise way for highly repetitive use cases. One of them is looping between a start & end integer possibly with a step.

Presently, to loop over a start & end integer value, users have to either create an `Array` or use an imperative style `while` loop. A concise syntax for e.g. `1 .. 10` would enhance the readability of the language.

## User Benefit

Users will benefit from the concise syntax leading to a cleaner & readable code.

## Design Proposal

This proposal suggest adding two new types to Cadence:
1. `Range<T: Integer`
2. `Progression<T: Integer>`

### Range

A `Range` will be defined as follows:

```go
struct Range<T: Integer> {
   let start: T
   let endInclusive: T
}
```

It can be instantiated using the `..` & `downTo` binary operators & can be used in a `for-in` loop as follows:

```cadence
// i will be start, start + 1, .. endInclusive
for i in start .. endInclusive {
    // do something with i
}

// i will be start, start - 1, .. endInclusive
for i in start downTo endInclusive {
    // do something with i
}
```

When `..` is used, we must have `start <= endInclusive` while if `downTo` is used, then we must have `start >= endInclusive`.

`Range` will also provide the following public member variables and functions:

1. `length`: Returns the count of integers included in the `Range`.
2. `contains<T: Integer>(value: T): Bool`: Returns if the `Range` includes the provided `value`.

#### Examples

Example usage of a `Range` type:

```cadence
let rangeValue = 11 .. 45

let len1 = rangeValue.length // 35
let isPresent1 = rangeValue.contains(24) // True

for i in rangeValue {
    // logic using i
}

let rangeValueBackwards = 132 .. 33

let len2 = rangeValueBackwards.length // 100
let isPresent2 = rangeValueBackwards.contains(1) // False
```

### Progression

A `Progression` type will essentially be a `Range` with an additional `step` which determines the step of the progression. In other words, `Range` will be a `Progression` with `step` set to 1 or -1.

It will be defined as follows:

```go
struct Progression<T: Integer> {
   let start: T
   let endInclusive: T
   let step: T
}
```

It can be instantiated with the usage of `..` & `downTo` along with a `step` operator & can be used in a `for-in` loop as follows:

```cadence
// i will be start, start + 2, step + 4, .. endInclusive
for i in start .. endInclusive step 2 {
    // do something with i
}

// i will be start, start - 3, start - 6, .. endInclusive
for i in start downTo endInclusive step 3 {
    // do something with i
}
```

When `..` is used, we must have `start <= endInclusive` while if `downTo` is used, then we must have `start >= endInclusive`.

`Progression` will also provide the following public member variables and functions:

1. `length`: Returns the count of integers included in the `Progression`.
2. `step`: Returns the step size of the `Progression`.
3. `contains<T: Integer>(value: T): Bool`: Returns if the `Progression` includes the provided `value`.

#### Examples

Example usage of a `Progression` type:

```cadence
let progressionValue = 11 .. 45 step 2

let len1 = progressionValue.length // 18
let isPresent1 = progressionValue.contains(24) // False

for i in progressionValue {
    // logic using i
}

let progressionValueBackwards = 132 .. 33 step 2

let len2 = progressionValueBackwards.length // 50
let isPresent2 = progressionValueBackwards.contains(34) // True
```

### Drawbacks

None

### Alternatives Considered

No. Relatively straightforward. Other modern languages such as Kotlin, Swift & Rust have similar syntax & API for `Range`.

### Performance Implications

None

### Dependencies

None

### Engineering Impact

No engineering impact is expected.

### Compatibility

This change has no impact on compatibility between systems (e.g. SDKs).

### User Impact

The proposed feature is a purely additive.
There is no impact on existing contracts and new transactions.

## Related Issues

None

## Questions and Discussion Topics

None

## Implementation
Will be done as part of https://github.com/onflow/cadence/issues/2482.

