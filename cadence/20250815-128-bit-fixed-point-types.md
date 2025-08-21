---
status: proposed 
flip: 341
authors: Supun Setunga (supun.setunga@flowfoundation.org)
sponsor: Dieter Shirley (dete@flowfoundation.com)
updated: 2025-08-15
---

# FLIP 341: Add 128-bit Fixed-point Types to Cadence

## Objective

The objective is to add a set of 128-bit wide fixed-point types, `Fix128` and `UFix128` to Cadence.

## Motivation

Cadence currently only supports 64-bit wide decimal fixed-point types, `Fix64` and `UFix64`, 
having the ranges `-92233720368.54775808` through `92233720368.54775807` and `0.0` through `184467440737.09551615`
respectively.

However, there could be requirements to have a much higher precision than what `Fix64` and `UFix64` provide.
For example, applications that handle financial data would require higher precision for arithmetic operations.
Though this can be achieved with a custom fixed-point implementation using integers (128-bit length, 256-bit length,
or even unbounded integers), it is not only a lot of work for developers to implement such a type,
but also would be very inefficient.

## User Benefit

Developers can use fixed-point values in Cadence for high-precision arithmetics, efficiently and conveniently.

## Design Proposal

Add `Fix128` and `UFix128` to Cadence.

The proposal is to use a scale of 24, and therefore a scaling factor of 1e-24.

Other blockchain ecosystems (most notably Ethereum) frequently use scale factors (usually called "decimals", per ERC-20)
of 18 (used by ETH itself), 24 (used for internal calculations in MakerDAO), and 27 (used internally by Compound and Aave
for interest calculations).
Although 27 is the most commonly used value, the value of 24 has a precedent in Maker, and leads to a range that is
close to, but slightly larger than the range of `Fix64` and `UFix64`.
This allows all `UFix64` and `Fix64` values to be converted to the equivalent 128-bit types without any loss of precision
or range.
On the other hand, 27 decimals wouldn't have allowed the full range of `Fix64` and `UFix64` to fit in the new types, 
and using 18 decimals has been deemed to be insufficient by a number of defi protocols.

A scaling factor of 24 provides a very high precision for fractional values, while also leaving large enough upper and 
lower bounds sufficient for most real-world use cases (which typically will involve internal calculations that convert
back to the 64-bit types).

The new fixed-point type can accommodate values in the following ranges:
- `Fix128`: `-170141183460469.231731687303715884105728` through `170141183460469.231731687303715884105727`
- `UFix128`: `0.0` through `340282366920938.463463374607431768211455`

### Drawbacks

None.

### Performance Implications

None.

### Dependencies

None.

### Engineering Impact

Implementing a new and efficient fixed-point representation is non-trivial.

### Compatibility

Given this is a new feature addition, there's no impact on backward compatibility.

### User Impact

Given this is a new feature addition, there's no impact for existing users.

## Related Issues

None.

## Implementation
- `Fix128` - https://github.com/onflow/cadence/pull/4131
- `UFix128` - https://github.com/onflow/cadence/pull/4147

## Prior Art

Most programing languages do not have built-in types for 128-bit fixed-point values, but are provided either
as a standard library or are available as third-party libraries. e.g:
- Python: https://docs.python.org/3/library/decimal.html
- Rust: https://docs.rs/fixed/latest/fixed/index.html

