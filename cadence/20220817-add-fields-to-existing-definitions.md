---
status: proposed
flip: TBD
authors: Deniz Mert Edincik (deniz@edincik.com), Austin Kline (austin@flowty.io)
sponsor: Bastian Mueller (bastian.mueller@flowfoundation.org)
updated: 2022-08-17
---

# FLIP: Allow adding new fields to existing types

## Objective

This proposed change will allow updating existing types (contracts) with new fields.

## Motivation

One major challenge in smart contract design in Cadence is the inability to add new fields to already deployed types.
The lack of this functionality means that developers have to resort to brittle workarounds.
For example, developers may choose to deploy entirely new contracts,
that are essentially reprints of the original, except with the newly added fields.
Developers may also choose to
- Manually storing fields as dictionaries to accommodate future new fields
- Manually storing new fields in a new, different contract
- Manually storing new fields in storage

Each of these workarounds come with their downsides.
- New contracts lead to complex migrations for applications and sunsetting existing contracts.
- Storing new data manually is brittle, and leads to harder-to-follow code and added complexity.
- Factory patterns increase compute since objects must be built at runtime
  and also lose the benefits of type-checking, since the underlying structure is not truly typed.

## User Benefit

Allowing developers to add new fields to existing types as they need them
will make contract development more focused on the needs of the current version
as opposed to undue complexity to take future plans into account.
It will also allow more maintainable contract code with less complex logic when existing contracts get updated.

## Design Proposal

The [contract updatability checking](https://cadence-lang.org/docs/language/contract-updatability)
is extended to allow new fields to be added to existing types,
such as contracts, structs, resources, and attachments.
These new fields must be optional.

When the added field of an existing stored value is accessed, the result is `nil`.

For example, assume the following struct type is currently deployed as part of a contract:

```cadence
access(all)
struct Message {
    access(all)
    let content: String

    init(content: String) {
        self.content = content
    }
}
```

We can now add a new field to this struct:

```cadence
access(all)
struct Message {
    access(all)
    let content: String

    init(content: String) {
        self.content = content
    }

    // An optional new field. Existing instances of Message will have timestamp set to nil when accessed
    // unless they are set at a later date
    access(all)
    let timestamp: UInt64?
}
```

### Limitations

- This will not allow existing fields to be altered. That is, it is invalid to alter a field's type.
- This will not allow new fields to be derived from existing ones.

### Compatibility

This should be backwards compatible

### User Impact

- Cadence developers will be able to modify their contract more to their needs
  instead of over-designing with the first launch of their contract(s).
- If new unforeseen features or fixes require new fields,
  those additions can be kept in the original contract,
  instead of being silo'd off into their own contract.

## Prior Art

N/A

## Open Questions

- Adding fields can only be allowed if it is not possible to re-add previously removed fields.

  One option could be to allow keep allowing to remove existing fields,
  but require them to be "tomb-stoned",
  similar to how [FLIP 275](https://github.com/onflow/flips/issues/275) proposes
  to allow removing existing type definitions.

- A previous iteration of the proposal also proposed to allow adding non-optional fields,
  by allowing the definition of the added field to provide a *default* value.

  This could be useful for types for which an optional type unnecessarily complicates the type,
  such as for simple types like `Bool`, `String`, `Int`, etc.

  For example, the field added in the example above could be written instead as:

  ```cadence
  let timestamp: UInt64 = 0
  ```

  Support for default values could be restricted to just certain types (e.g. `Bool`, `String`, numbers),
  and default value expressions could be restricted to just literals (e.g. `true`, `""`, `0`).
  Further, value expressions could be extended to allow access to `self`,
  and access expressions on it could be allowed.

  The default value functionality would be similar to default values of
  [resource destruction events](https://github.com/onflow/flips/blob/main/cadence/20230811-destructor-removal.md#destruction-events).
