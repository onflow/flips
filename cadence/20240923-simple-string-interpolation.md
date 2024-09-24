---
status: draft 
flip: 288
authors: Raymond Zhang (raymond.zhang@flowfoundation.org)
sponsor: Supun Setunga (supun.setunga@flowfoundation.org) 
updated: 2024-09-23
---

# FLIP 288: Simple String Interpolation

## Objective

This FLIP proposes adding support for simple string interpolation limited to identifiers only (no support for full expressions).

## Motivation

Currently Cadence has no support for string interpolation. It is convenient for developers to be able to inline variables in strings as opposed to the current solution of applying `concat` repeatedly. 

In general many languages support string interpolation for readability and ease-of-use. 

## User Benefit

Developers can avoid repeatedly using `concat` to generate strings which will simplify code and readability.

## Design Proposal

The main constraint is backward compatibility, existing Cadence 1.0 code cannot be affected. The proposed solution follows the syntax of swift as below:

```swift
"Variable = \(someVar)"
```

This change is backwards compatible because `\(` is not currently a valid escape character. 

For this initial proposal there will be several limitations on `someVar`:
- `someVar` will only be variable references with ability to be extended to expressions in the future
- `someVar` must support the built-in function `toString()` meaning it must be either a String, Number, Address or Character.  

This is still useful for the first iteration since there are easy workarounds for these limitations such as extracting expressions into local variables.

### Drawbacks

None.

### Alternatives Considered

String interpolation is preferred over a formatting function (such as `printf`) for the following reasons:

- Formatting is error prone (e.g. passing a `String` to `%d`)
- There are many languages which moved from traditional formatting to string interpolation (e.g. Python)

### Performance Implications

None, non-interpolated strings should not be affected.

### Dependencies

None.

### Engineering Impact

This change should be simple to implement. 

### Best Practices

It may be preferred to use string interpolation over concat once implemented.

### Compatibility

Proposed changes are backwards compatible. 

### User Impact

This is a feature addition, no impact. 

## Related Issues

https://github.com/onflow/cadence/issues/3579

## Prior Art

- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/#String-Interpolation
- https://docs.python.org/3/tutorial/inputoutput.html#formatted-string-literals

## Questions and Discussion

Feel free to discuss on the PR.

## Implementation

A POC is available at https://github.com/onflow/cadence/pull/3585.
