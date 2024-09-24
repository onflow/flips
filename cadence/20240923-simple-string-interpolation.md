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

Developers can avoid repeatedly using `concat` to generate strings which will simplify code and readability. Repeated concatenation is inefficient, as each concatenation results in an intermediate/temporary result, leading to an allocation and associated metering (computation and memory usage).

In comparison, string interpolation can be implemented efficiently, and will thus also will lead to the user benefit of being less costly in terms of computation and memory usage.

## Design Proposal

The proposed syntax for the string-literal with interpolation looks like follows:

```swift
"Variable = \(someVar)"
```

The main constraint is backward compatibility, existing Cadence 1.0 code cannot be affected. This change is backwards compatible because `\(` is not currently a valid escape character. 

For this initial proposal there will be several limitations on `someVar`:
- `someVar` will only be variable references with ability to be extended to expressions in the future
- `someVar` must support the built-in function `toString()` meaning it must be either a `String`, `Number`, `Address`, `Character`, `Bool` or `Path`.  

This is still useful for the first iteration since there are easy workarounds for these limitations such as extracting expressions into local variables.

### Drawbacks

None.

### Alternatives Considered

String interpolation is preferred over a formatting function (such as `printf`) for two main reasons (1) string formatting is error prone and (2) there are many languages which moved from traditional formatting to string interpolation (e.g. Python).

To talk more about (1):
- The type of the argument passed to the formatting facility may not match the format specifier (e.g. value of type `String` is passed to format specifier for type Integer like `%d`)
- Number of arguments passed to the formatter might not match the number of placeholders in the format specifier (e.g. too many or too few arguments)
- When there are many placeholders in the format string, the order of arguments can get mixed up easily
In most implementations of string formatting facilities, like when it is implemented as a normal function, these errors can not be detected statically, but only at run-time.

However, some languages have formatting facilities that avoid some of the problems noted above, by statically checking the format specifier and the argument list. For example, Rust's `format!` macro does that.

There are also many other implementations for string interpolation itself. For example:
- Python has "f-strings" where an `f` prefix of a string-literal indicates interpolation with curly brace placeholders (e.g. `f"{someVar}"`)
- JavaScript has backticks to indiciate string interpolation with curly brace placeholders (e.g. `` `{someVar}` ``)
- Kotlin has string interpolation in normal string literals with optional curly brace placeholders with a leading `$` (e.g. `"$someVar and ${someVar2}"`)

There exists no clear "standard". The proposed design leverages the fact that escaping is already supported `\` and parentheses naturally surround values in expressions. Therefore, no new syntax is introduced (e.g. no literal prefix, no placeholder indicator, no new expression surround characters).


### Performance Implications

None, non-interpolated strings should not be affected. The proposed design of string interpolation has much better performance compared to repeated string concatenation.

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
Existing instances in other languages
- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/#String-Interpolation
- https://docs.python.org/3/tutorial/inputoutput.html#formatted-string-literals
Some rationale and useful comparisons of string interpolation versus string formatting
- https://peps.python.org/pep-0498/

## Questions and Discussion

Feel free to discuss on the PR.

## Implementation

A POC is available at https://github.com/onflow/cadence/pull/3585.
