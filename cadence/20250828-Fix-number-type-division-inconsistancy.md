---
status: proposed 
flip: 343
authors: Supun Setunga (supun.setunga@flowfoundation.org), Dieter Shirley (dete@flowfoundation.com)
sponsor: Dieter Shirley (dete@flowfoundation.com)
updated: 2025-08-28
---

# FLIP 343: Fix numeric type rounding inconsistency

## Objective

Provide a well-defined, consistent behaviour for handling the least-significant digit in arithmetic operations for 
all Cadence numeric types.

## Motivation

Numeric types in Cadence have no documented rounding behaviour when performing division operations, or fractional
multiplication operations, but does truncation in most cases.
For example:

```Cadence
Int8(5) / Int8(2) == Int8(2)
Int8(-5) / Int8(2) == Int8(-2)
Int8(5) / Int8(-2) == Int128(-2)
Int8(-5) / Int8(-2) == Int8(2)

Int64(5) / Int64(2) == Int64(2)
Int64(-5) / Int64(2) == Int64(-2)
Int8(5) / Int8(-2) == Int128(-2)
Int64(-5) / Int64(-2) == Int64(2)

UFix64(0.00000005) / UFix64(2.0) == UFix64(0.00000002)
UFix64(0.00000005) * UFix64(0.5) == UFix64(0.00000002)
```

However, for negative inputs, the types `Int`, `Int128`, `Int256` and `Fix64` provide different results compared to the
previously mentioned types.

```Cadence
Int128(5) / Int128(2) == Int128(2)
Int128(-5) / Int128(2) == Int128(-3)
Int128(5) / Int128(-2) == Int128(-2)
Int128(-5) / Int128(-2) == Int128(3)

Int256(5) / Int256(2) == Int256(2)
Int256(-5) / Int256(2) == Int256(-3)
Int256(5) / Int256(-2) == Int256(-2)
Int256(-5) / Int256(-2) == Int256(3)

Int(5) / Int(2) == Int(2)
Int(-5) / Int(2) == Int(-3)
Int(5) / Int(-2) == Int(-2)
Int(-5) / Int(-2) == Int(3)

Fix64(0.00000005) / Fix64(2.0) == Fix64(0.00000002)
Fix64(-0.00000005) / Fix64(2.0) == Fix64(-0.00000003)
Fix64(0.00000005) / Fix64(-2.0) == Fix64(-0.00000002)
Fix64(-0.00000005) / Fix64(-2.0) == Fix64(0.00000003)
```

This difference in behavior has been due to the use of different implementations for the numeric types.
All integer types that have a length of 64-bit or less (`[U]Int8`, `[U]Int16`, `[U]Int32`, `[U]Int64`) are backed by
Go-lang primitive types, whose division truncates the least-significant digit.

On the other hand, integer types that have a length of 128-bit or more (`[U]Int128`, `[U]Int256`, `[U]Int`),
and fixed-point types (`[U]Fix64`) have been backed by https://pkg.go.dev/math/big standard library, and have been using
[Int.Div(...)](https://pkg.go.dev/math/big#Int.Div) for division, which uses [Euclidean division](https://en.wikipedia.org/wiki/Euclidean_division),
unlike Go-primitives.

This has resulted in the above inconsistency that we are seeing.

## User Benefit

Developers get a consistent behaviour for arithmetic operations for all numeric types.

## Design Proposal

This FLIP proposes to use the truncation for the least-significant digit for all arithmetic operations in Cadence.
As such, while `[U]Int8`, `[U]Int16`, `[U]Int32`, `[U]Int64`, and `UFix64` remains unchanged, `[U]Int128`, `[U]Int256`,
`[U]Int`, and `Fix64` would now behave slightly differently than they used to be, and would give the new results as
follows:

```cadence
Int128(5) / Int128(2) == Int128(2)
Int128(-5) / Int128(2) == Int128(-2)
Int128(5) / Int128(-2) == Int128(-2)
Int128(-5) / Int128(-2) == Int128(2)

Int256(5) / Int256(2) == Int256(2)
Int256(-5) / Int256(2) == Int256(-2)
Int256(5) / Int256(-2) == Int256(-2)
Int256(-5) / Int256(-2) == Int256(2)

Int(5) / Int(2) == Int(2)
Int(-5) / Int(2) == Int(-2)
Int(5) / Int(-2) == Int(-2)
Int(-5) / Int(-2) == Int(2)

Fix64(0.00000005) / Fix64(2.0) == Fix64(0.00000002)
Fix64(-0.00000005) / Fix64(2.0) == Fix64(-0.00000002)
Fix64(0.00000005) / Fix64(-2.0) == Fix64(-0.00000002)
Fix64(-0.00000005) / Fix64(-2.0) == Fix64(0.00000002)
```

Note that this suggested change would only impact the **division (and saturation division) of negative values**
for the above types.
Division of non-negative numbers, as well as all the other arithmetics on both negative and non-negative numbers,
would remain unchanged. 

### Drawbacks

This would impact developers who may have been relying on the old behaviour for the division of negative numbers.

### Performance Implications

None.

### Dependencies

None.

### Engineering Impact

Updating the behaviour is trivial.

### Compatibility

Given this change the behaviour for division for negative numbers, it'll be backward incompatible.

### User Impact

This would impact users who may have been relying on the old behaviour for the division of negative numbers.

## Related Issues

None.

## Implementation
Draft implementation can be found at: https://github.com/onflow/cadence/pull/4184
