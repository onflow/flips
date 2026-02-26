---
status: draft
flip: 359
authors: Bastian Müller (bastian.mueller@flowfoundation.org)
updated: 2026-02-25
---

# FLIP 359: For-in Iteration Over Dictionary Keys

## Objective

This FLIP proposes adding support for iterating over the keys of a dictionary using a `for-in` loop,
extending the existing iteration support from arrays to dictionaries.

## Motivation

Cadence already supports `for-in` loops over arrays, strings, and inclusive ranges.
Dictionaries are a fundamental and commonly used collection type in Cadence,
but currently lack direct `for-in` loop support.

To iterate over a dictionary today, developers must obtain the keys as an array first
and then iterate over that:

```cadence
let dict = {"a": 1, "b": 2, "c": 3}
for key in dict.keys {
    // use key and dict[key]
}
```

While this works, it requires an intermediate allocation of the keys array.

## User Benefit

Developers can iterate over dictionary keys directly and concisely using the `for-in` loop syntax
they are already familiar with, without needing to extract keys into a temporary array first:

```cadence
let dict = {"a": 1, "b": 2, "c": 3}
for key in dict {
    let value = dict[key]!
    // ...
}
```

This also avoids allocating an intermediate array of keys.

## Design Proposal

The `for-in` loop is extended to support dictionary types.
When iterating over a dictionary, the loop variable is bound to each key of the dictionary in turn.
The type of the loop variable is the key type of the dictionary.

### Syntax

No new syntax is introduced. The existing `for-in` loop syntax is reused:

```cadence
for <binding> in <dictionary-expression> {
    <body>
}
```

### Type Checking

When the expression following `in` has a dictionary type `{K: V}`, the loop variable has type `K`.

The index binding form (`for index, value in collection`) is **not** (yet) supported for dictionaries.
Attempting to use it is a static type error.
A future FLIP could explore adding support for iterating over key-value pairs.

### Iteration Order

The order of iteration over dictionary keys is determined by the underlying storage order of the dictionary.
The order is not guaranteed to be sorted or insertion-ordered.
This matches the behaviour of `dict.keys` and `dict.forEachKey`.

### Drawbacks

None. This is a purely additive change that does not affect existing code.

### Alternatives Considered

None

### Performance Implications

Direct dictionary iteration avoids allocating an intermediate `[K]` array of keys
(as `dict.keys` does), which may be beneficial for large dictionaries.

### Dependencies

None. This change is self-contained within the Cadence language and runtime.

### Engineering Impact

The addition requires minimal code changes.

### Best Practices

Developers iterating over dictionary keys should prefer `for key in dict` over
`for key in dict.keys` when they do not need a snapshot of the keys as an array.

A linter rule could be added in the future to encourage this form,
as it is more efficient and concise.

### Compatibility

This is a purely additive language change. Existing contracts and transactions are unaffected.

### User Impact

New feature, no impact on existing code.

## Related Issues

Adding iteration over key-value pairs (e.g. `for key, value in dict`) could be explored in a future FLIP.

## Prior Art

Many languages support direct dictionary/map iteration:

- **Python**: `for key in d:` iterates over keys of a dictionary.
- **Go**: `for k := range m` iterates over keys of a map.

## Implementation

An implementation is available at https://github.com/onflow/cadence/pull/4436
