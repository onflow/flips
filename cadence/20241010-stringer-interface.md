---
status: draft 
flip: NNN (set to the issue number)
authors: Raymond Zhang (raymond.zhang@flowfoundation.org)
sponsor: AN Expert (core-contributor@example.org) 
updated: 2024-10-10
---

# FLIP NNN: Stringer Interface

## Objective

This flip proposes the addition of a struct interface `StructStringer` which all string-convertible structs will implement. One goal of this FLIP is to simplify the code for converting `AnyStruct` to `String`. Secondly, a customizable `toString` will be useful for future string formatting such as string interpolation with this interface.

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
    if let stringerValue = value as ? {StructStringer} {
        return stringerValue.toString()
    }
}
```
The same code above additionally allows for a conforming `Struct` to be converted to a `String` as well which was not previously generalizable. This can be useful for various string formatting functions.

## User Benefit

This will significantly streamline the process of converting `AnyStruct` values to `String` which is useful for developers. It also allows for much cleaner and simpler descriptions of a `Struct` for both developers and end users through customizability. It also allows for more powerful string formatting functions.

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

As with all user-defined types there are risks associated with calling someone else's `toString` function such as panic or gas usage concerns that developers need to be aware of. This means that unless the value in question is a primitive type you will have to perform the necessary defensive checks. 

### Alternatives Considered

An alternative is to have `AnyStruct` itself implement a `toString` function. As a native function there is no longer any risk of malicious code and it still solves the first goal. Additionally, it still allows for more powerful string formatting. The downside of this approach is in limiting customizability for developers. 

### Performance Implications

None.

### Dependencies

None.

### Engineering Impact

This change should be simple to implement.

### Best Practices

This should become the preferred way of converting `AnyStruct` to `String`.

### Compatibility

Proposed changes are backwards compatible.

### User Impact

Feature addition, no impact.

## Related Issues

string formatting

## Prior Art

Many languages have an interface for formatting types as `String` such as

- [Display trait](https://doc.rust-lang.org/std/fmt/trait.Display.html) in Rust
- `Stringer` in Go

