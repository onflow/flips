---
status: draft 
flip: 293
authors: Raymond Zhang (raymond.zhang@flowfoundation.org)
sponsor: Supun Setunga (supun.setunga@flowfoundation.org) 
updated: 2024-10-21
---

# FLIP 293: Stringer Interface

## Objective

This FLIP proposes the addition of a struct interface `StructStringer` to Cadence, which all string-convertible structs will implement. One goal of this FLIP is to simplify the process for representing `AnyStruct` as a `String`. Secondly, a customizable `toString` will be useful for future string formatting such as string interpolation with this interface.

## Motivation

Currently if a developer wants to accept a value as `AnyStruct` and convert it to a string they have to identify which subtype the value belongs to and call the associated `toString` function. This leads to code such as the following
```cadence
access(all) fun toString(_ value: AnyStruct): String? {
    if let stringValue = value as? String {
        return stringValue
    } else if let boolValue = value as? Bool {
        return boolValue ? "true" : "false"
    } else if let characterValue = value as? Character {
        return characterValue.toString()
    } else if let addressValue = value as? Address {
        return addressValue.toString()
    } else if let pathValue = value as? Path {
        return pathValue.toString()
    } else if let intValue = value as? Int {
        return intValue.toString()
    }
    ...
}
```
With the proposed addition, the code could be simplified to 
```cadence
access(all) fun toString(_ value: AnyStruct): String? {
    if let stringerValue = value as? {StructStringer} {
        return stringerValue.toString()
    }
}
```
Additionally a conforming struct can be converted to a `String` in the same function which was not previously possible. This can be useful for various string formatting functions.

## User Benefit

This will significantly streamline the process of converting `AnyStruct` values to `String` which is useful for developers, especially for debugging and providing user readable descriptions. It also allows for more powerful string formatting functions.

## Design Proposal

The proposed interface is as follows

```cadence
access(all) 
struct interface StructStringer {
    access(all)
    view fun toString(): String
}
```

### Drawbacks

As with all user-defined types there are risks associated with calling someone else's `toString` function such as panic or gas usage concerns that developers need to be aware of.

### Alternatives Considered

An alternative is to have `AnyStruct` itself implement a `toString` function. As a native function there is no longer any risk of malicious code and it still solves the first goal. Additionally, it still allows for more powerful string formatting. The downside of this approach is in limiting customizability for developers.

### Performance Implications

None.

### Dependencies

None.

### Engineering Impact

This change should be simple to implement.

### Best Practices

Reduce code size in converting `AnyStruct` to `String`.

### Compatibility

Proposed changes are backwards compatible.

### User Impact

Feature addition, no impact.

## Related Issues

[string formatting](https://github.com/onflow/cadence/issues/3579)

## Prior Art

Many languages have an interface for formatting types as `String` such as

- [`Display` trait](https://doc.rust-lang.org/std/fmt/trait.Display.html) in Rust
- [`Stringer` interface](https://pkg.go.dev/fmt#Stringer) in Go
- [`CustomStringConvertible`](https://developer.apple.com/documentation/swift/customstringconvertible)  in Swift

