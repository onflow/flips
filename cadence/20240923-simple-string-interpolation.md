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

There are many possible implementations two of which are highlighted below. The main constraint is backward compatibility, existing Cadence 1.0 code cannot be affected.

### Python f-string syntax
Support the following:
```python
f"value is {value}"
```
This change is backwards compatible as it introduces a new class of strings.

### Swift \\()
Support the following:
```swift
"value is \(value)"
```
This change is backwards compatible because `\(` is not currently a valid escape character. 

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

Extension to support expressions as opposed to just identifiers. Support custom `toString()` functions as well.

## Prior Art

- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/#String-Interpolation
- https://docs.python.org/3/tutorial/inputoutput.html#formatted-string-literals

## Questions and Discussion

Feel free to discuss on the PR.

Questions:
- Preferences or concerns on proposed syntax
- It may be less work to allow expressions versus enforcing identifier only, how much extra testing would be involved?
