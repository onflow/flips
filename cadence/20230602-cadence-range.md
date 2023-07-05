---
status: accepted
flip: 96
authors: darkdrag00n (darkdrag00n@proton.me)
sponsor: Bastian Müller (bastian@dapperlabs.com)
updated: 2023-06-19
---

# FLIP 96: Range Types

## Objective

This FLIP proposes adding the concept of a range to Cadence. It proposes two new types, `InclusiveRange` and `ExclusiveRange` along with constructor functions to create them. It also proposes their usage inside a `for-in` loop.

## Motivation

Modern langauges often provide a concise way for highly repetitive use cases. One of them is looping between a start & end integer possibly with a step.

Presently, to loop over a start & end integer value, users have to either create an `Array` or use an imperative style `while` loop. A concise syntax would enhance the readability of the language.

## User Benefit

The two `Range` types improves over the usage of arrays. Hence, users will benefit from the concise syntax leading to a cleaner & readable code.

## Design Proposal

New types `InclusiveRange<T: Integer>` & `ExclusiveRange<T: Integer>` will be added to Cadence and defined as follows:

```cadence
struct InclusiveRange<T: Integer> {
   let start: T
   let end: T
   let step: T
}

struct ExclusiveRange<T: Integer> {
   let start: T
   let end: T
   let step: T
}
```

Please note the following points:
1. In `InclusiveRange`, `end` is an element of the resulting sequence if and only if it can be produced from `start` using steps of size `step`.

2. In `ExclusiveRange`, `end` is guaranteed to not be part of the generated sequence. As a result, if `start == end`, the resulting sequence is invalid & we disallow creation of such `ExclusiveRange`.

Both the `Range` types will be usable in a `for-in` loop.

### Constructor Functions
Constructor functions will be defined to instantiate variables of type `InclusiveRange` & `ExclusiveRange`. They are defined below.

- `InclusiveRange(_ start: T, _ end: T, step: T)`: Creates an `InclusiveRange` with the provided start, end and step. `step` will be an optional argument with a default value of step as 1 if `start <= end` or -1 if `start > end`.

- `ExclusiveRange(_ start: T, _ end: T, step: T)`: Creates an `ExclusiveRange` with the provided start, end and step. `step` will be an optional argument with a default value of step as 1 if `start < end` or -1 if `start > end`.

The following constraints are applicable in both the constructor functions:

1. `step` must be non-zero i.e. `abs(step) > 0`.

2. The value of `step` must lead to the sequence moving towards the end.

Violation of the above constraints will lead to runtime errors during instantiation.

### Public Members
Both `InclusiveRange` & `ExclusiveRange` will also provide the following public members:

1. `contains<T: Integer>(value: T): Bool`: Returns if the Range includes the provided `value`.

### Examples

Example usage of `InclusiveRange` and `ExclusiveRange` types:

```cadence
////////////////////
// InclusiveRange //
////////////////////

let inclusiveRangeValue = InclusiveRange(11, 21)
inclusiveRangeValue.contains(20) // True
for i in inclusiveRangeValue {
    // i will be 11, 12, 13, ... 20, 21
}

let inclusiveRangeValueWithStep = InclusiveRange(11, 20, step: 2)
inclusiveRangeValueWithStep.contains(20) // False since 20 cannot be produced from 11 using step of 2.
for i in inclusiveRangeValueWithStep {
    // i will be 11, 13, 15, 17, 19
}

let inclusiveRangeValueBackwards = InclusiveRange(132, 33)
inclusiveRangeValueBackwards.contains(1) // False
for i in inclusiveRangeValue {
    // i will be 132, 131, 130, ... 34, 33
}

let inclusiveRangeValueBackwards = InclusiveRange(132, 33, step: -3)
inclusiveRangeValueBackwards.contains(34) // True
for i in inclusiveRangeValue {
    // i will be 132, 125, 118, ... 41, 34
}

let invalidStep = InclusiveRange(10, 32, step: 0) // Runtime Error
let invalidDirection = InclusiveRange(132, 33, step: 3) // Runtime Error since the sequence moves in increasing direction.

////////////////////
// ExclusiveRange //
////////////////////

let exclusiveRangeValue = ExclusiveRange(11, 21)
exclusiveRangeValue.contains(20) // True
exclusiveRangeValue.contains(21) // False
for i in exclusiveRangeValue {
    // i will be 11, 12, 13, ... 19, 20
}

let exclusiveRangeValueWithStep = ExclusiveRange(11, 19, step: 2)
exclusiveRangeValueWithStep.contains(19) // False
for i in exclusiveRangeValueWithStep {
    // i will be 11, 13, 15, 17
}

let exclusiveRangeValueBackwards = ExclusiveRange(132, 33)
exclusiveRangeValueBackwards.contains(1) // False
for i in exclusiveRangeValue {
    // i will be 132, 131, 130, ... 35, 34
}

let exclusiveRangeValueBackwards = ExclusiveRange(132, 33, step: -3)
exclusiveRangeValueBackwards.contains(35) // False
exclusiveRangeValueBackwards.contains(34) // True
for i in exclusiveRangeValue {
    // i will be 132, 125, 118, ... 41, 34
}

let exclusiveRangeValueEmpty = ExclusiveRange(10, 10, step: 2) // Runtime Error
let invalidStep = ExclusiveRange(10, 32, step: 0) // Runtime Error
let invalidDirection = ExclusiveRange(132, 33, step: 3) // Runtime Error
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

It would require 2-3 weeks of engineering effort to implement, review & test the feature.

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

