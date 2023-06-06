---
status: draft
flip: 96
authors: darkdrag00n (darkdrag00n@proton.me)
sponsor: Bastian Müller (bastian@dapperlabs.com)
updated: 2023-06-02
---

# FLIP 96: Range Type

## Objective

This FLIP proposes adding the concept of a `Range` to Cadence. It proposes two new types, `InclusiveRange` and `ExclusiveRange` along with constructor functions to create them. It also proposes their usage inside a `for-in` loop.

## Motivation

Modern langauges often provide a concise way for highly repetitive use cases. One of them is looping between a start & end integer possibly with a step.

Presently, to loop over a start & end integer value, users have to either create an `Array` or use an imperative style `while` loop. A concise syntax would enhance the readability of the language.

## User Benefit

The two `Range` types improves over the usage of arrays. Hence, users will benefit from the concise syntax leading to a cleaner & readable code.

## Design Proposal

New types `InclusiveRange<T: Integer>` & `ExclusiveRange<T: Integer>` will be added to Cadence and defined as follows:

```go
struct InclusiveRange<T: Integer> {
   let start: T
   let endInclusive: T
   let step: T
}

struct ExclusiveRange<T: Integer> {
   let start: T
   let endExclusive: T
   let step: T
}
```

Please note the following points:
1. In `InclusiveRange`, `endInclusive` is an element of the resulting sequence if and only if it can be produced from `start` using steps of size `step`.

2. In `ExclusiveRange`, `endExclusive` is guaranteed to not be part of the generated sequence. As a result, if `start == endExclusive`, the resulting sequence is empty.

Both the `Range` types will be usable in a `for-in` loop.

### Constructor Functions
Constructor functions will be defined to instantiate variables of type `InclusiveRange` & `ExclusiveRange`. They are defined below.

- `InclusiveRange(start: T, endInclusive: T)`: Creates an `InclusiveRange` with the provided start and end with a default value of step as 1 if `start <= endInclusive` or -1 if `start > endInclusive`.

- `InclusiveRangeWithStep(start: T, endInclusive: T, step: T)`: Creates an `InclusiveRange` with the provided start, end and step.

- `ExclusiveRange(start: T, endExclusive: T)`: Creates an `ExclusiveRange` with the provided start and end with a default value of step as 1 if `start <= endInclusive` or -1 if `start > endInclusive`.

- `ExclusiveRangeWithStep(start: T, endExclusive: T, step: T)`: Creates an `ExclusiveRange` with the provided start, end and step.

The following constraints are applicable in all the four constructor functions:

1. `step` must be non-zero i.e. `abs(step) > 0`.

2. The value of `step` must lead to the sequence moving towards the end.

Violation of the above constraints will lead to runtime errors during instantiation.

### Public Members
`Range` will also provide the following public member variables and functions:

1. `length`: Returns the count of integers included in the `Range`.
2. `contains<T: Integer>(value: T): Bool`: Returns if the `Range` includes the provided `value`.

### Examples

Example usage of `InclusiveRange` and `ExclusiveRange` types:

```cadence
////////////////////
// InclusiveRange //
////////////////////

let inclusiveRangeValue = InclusiveRange(11, 21)
inclusiveRangeValue.length // 11
inclusiveRangeValue.contains(20) // True
for i in inclusiveRangeValue {
    // i will be 11, 12, 13, ... 20, 21
}

let inclusiveRangeValueWithStep = InclusiveRangeWithStep(11, 20, 2)
inclusiveRangeValueWithStep.length // 5
inclusiveRangeValueWithStep.contains(20) // False since 20 cannot be produced from 11 using step of 2.
for i in inclusiveRangeValueWithStep {
    // i will be 11, 13, 15, 17, 19
}

let inclusiveRangeValueBackwards = InclusiveRange(132, 33)
inclusiveRangeValueBackwards.length // 100
inclusiveRangeValueBackwards.contains(1) // False
for i in inclusiveRangeValue {
    // i will be 132, 131, 130, ... 34, 33
}

let inclusiveRangeValueBackwards = InclusiveRangeWithStep(132, 33, -3)
inclusiveRangeValueBackwards.length // 15
inclusiveRangeValueBackwards.contains(34) // True
for i in inclusiveRangeValue {
    // i will be 132, 125, 118, ... 41, 34
}

let invalidStep = InclusiveRangeWithStep(10, 32, 0) // Runtime Error
let invalidDirection = InclusiveRangeWithStep(132, 33, 3) // Runtime Error since the sequence moves in increasing direction.

////////////////////
// ExclusiveRange //
////////////////////

let exclusiveRangeValue = ExclusiveRange(11, 21)
exclusiveRangeValue.length // 10
exclusiveRangeValue.contains(20) // True
exclusiveRangeValue.contains(21) // False
for i in exclusiveRangeValue {
    // i will be 11, 12, 13, ... 19, 20
}

let exclusiveRangeValueWithStep = ExclusiveRangeWithStep(11, 19, 2)
exclusiveRangeValueWithStep.length // 4
exclusiveRangeValueWithStep.contains(19) // False
for i in exclusiveRangeValueWithStep {
    // i will be 11, 13, 15, 17
}

let exclusiveRangeValueBackwards = ExclusiveRange(132, 33)
exclusiveRangeValueBackwards.length // 99
exclusiveRangeValueBackwards.contains(1) // False
for i in exclusiveRangeValue {
    // i will be 132, 131, 130, ... 35, 34
}

let exclusiveRangeValueBackwards = ExclusiveRangeWithStep(132, 33, -3)
exclusiveRangeValueBackwards.length // 15
exclusiveRangeValueBackwards.contains(34) // True
for i in exclusiveRangeValue {
    // i will be 132, 125, 118, ... 41, 34
}

let exclusiveRangeValueEmpty = ExclusiveRangeWithStep(10, 10, 2) // Empty since start == endExclusive

let invalidStep = ExclusiveRangeWithStep(10, 32, 0) // Runtime Error
let invalidDirection = ExclusiveRangeWithStep(132, 33, 3) // Runtime Error

```

### Drawbacks

None

### Alternatives Considered
Initial proposal considered defining operators such as `..` or `downTo` for defining `Range`. It also proposed adding another type named `Progression` for allowing non-default values of `step`. 

Due to the readability concerns associated with the inclusive vs exclusive behavior of the operators, it was considered better to have types and constructor functions with self-explanatory names to avoid ambiguity.

It was also proposed that since `Progression` is essentially a `Range` with non-default value of `step`, the two types can be merged into one.

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

