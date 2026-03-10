---
status: accepted
flip: 355
authors: Bastian Müller (bastian.mueller@flowfoundation.org)
updated: 2026-03-10
---

# FLIP 355: Cadence Guard Statement

## Objective

Add guard statements to Cadence,
enabling early exit validation patterns that reduce nesting and eliminate force unwrapping.

## Motivation

Smart contracts require extensive input validation.
Currently, developers must choose between deeply nested if statements or manual nil checks followed by force unwrapping:

```cadence
// Current approach: Manual nil check + force unwrap
fun processUser(name: String?, age: Int?): User? {
    if name == nil { return nil }
    if age == nil { return nil }

    // Must force unwrap - type checker doesn't track the validation
    return User(name: name!, age: age!)
}
```

The nil check pattern requires force unwrapping because the type checker doesn't track the validation.
This is unsafe and not idiomatic.

The alternative is nested if-let statements:

```cadence
// Alternative: Nested if-let
fun processUser(name: String?, age: Int?): User? {
    if let n = name {
        if let a = age {
            return User(name: n, age: a)
        }
    }
    return nil
}
```

The if-let approach is safe but leads to deep nesting, making the code harder to read.

Guard statements solve both problems:

```cadence
// With guard: clean and safe
fun processUser(name: String?, age: Int?): User? {
    guard let n = name else { return nil }
    guard let a = age else { return nil }

    return User(name: n, age: a)
}
```

Guard eliminates force unwrapping, reduces nesting, and makes the happy path prominent by handling error cases first.

## User Benefit

Guard statements provide three key benefits:

**Eliminate Force Unwrapping**:
Optional binding with guard unwraps values safely.
The type checker tracks that validation has occurred, eliminating the need for unsafe force unwrap operators.

**Improve Readability**: Early exits for error cases keep the main logic unindented and linear:
First validate preconditions, then execute the happy path.

**Better Resource Safety**: Guard integrates with Cadence's resource tracking system.
The type checker verifies resources are handled correctly in all code paths,
including failable casts and optional resources.

## Design Proposal

The design for guard statements is similar to if statements, with three forms:
boolean guards, optional binding guards, and failable casting guards for resources.

### Syntax

```cadence
// Boolean guard
guard <boolean-expression> else {
    // must exit: return, panic, break, or continue
}

// Optional binding guard
guard let <identifier> = <optional-expression> else {
    // must exit: return, panic, break, or continue
}

// Failable casting guard for resources
guard let <identifier> <- <expression> as? @<Type> else {
    // must exit: return, panic, break, or continue
}
```

### Semantics

The guard statement evaluates its condition.
If the condition is false, or the value is nil, or a failable cast fails, the else block executes.
The else block must always exit the current scope.

For optional binding and failable casts,
the unwrapped or downcasted variable is available in the current scope after the guard,
not in a nested scope like if-let.

For failable casts with resources,
if the cast succeeds, the original resource is invalidated (moved into the new binding).
If the cast fails, the original resource remains available in the else block for cleanup.

### Key Design Decisions

**Mandatory else block exit**:
The type checker verifies the else block always exits via return, panic, break, or continue.
This ensures the unwrapped variable is only used when valid.

**Current scope for variables**:
Unlike if-let, guard-let declares variables in the current scope.

### Examples

**Basic validation:**
```cadence
access(all) fun withdraw(amount: UFix64) {
    guard amount > 0.0 else {
        panic("Amount must be positive")
    }
    guard amount <= self.balance else {
        panic("Insufficient balance")
    }
    self.balance = self.balance - amount
}
```

**Optional unwrapping:**
```cadence
access(all) fun getUser(id: UInt64): User? {
    guard let user = self.users[id] else {
        return nil
    }
    // user available as User (not User?)
    return user
}
```

**Resource handling:**
```cadence
access(all) fun processVault(vault: @Vault?): UFix64 {
    guard let v <- vault else {
        return 0.0
    }
    let balance = v.balance
    destroy v
    return balance
}
```

**Failable casts:**
```cadence
access(all) fun extractNFT(resource: @AnyResource): UInt64 {
    guard let nft <- resource as? @NFT else {
        destroy resource  // original resource available in else
        return 0
    }
    let id = nft.id
    destroy nft
    return id
}
```


## Drawbacks

None identified. This is a purely additive feature.

## Alternatives Considered

None. Potentially different syntax, e.g. `let ... else` in Rust, but `guard` is more explicit.

## Compatibility

This is a purely additive language feature with no impact on existing code.
The `guard` keyword is already reserved as a hard keyword and cannot be used as an identifier in existing code.

## User Impact

Existing contracts are unaffected. New code can immediately use guard statements. No migration needed.

## Prior Art

Swift introduced [guard statements](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/statements#Guard-Statement)
in Swift 2.0 (2015). They are widely used in production code. Cadence already shares several design principles with Swift.

Rust added a [similar feature (let-else)](https://doc.rust-lang.org/book/ch06-03-if-let.html#staying-on-the-happy-path-with-letelse)
in [version 1.65 (2022)](https://blog.rust-lang.org/2022/11/03/Rust-1.65.0/#let-else-statements),
validating the pattern's utility.

## Implementation

An implementation is available at https://github.com/onflow/cadence/pull/4432
