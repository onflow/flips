---
status: implemented
flip: 134
authors: Supun Setunga (supun.setunga@dapperlabs.com)
sponsor: Supun Setunga (supun.setunga@dapperlabs.com)
updated: 2023-10-24
---

# FLIP 134: Relax interface conformance restrictions

## Objective

A previous FLIP (https://github.com/onflow/flips/blob/main/cadence/20230503-improve-conformance.md) improved interface
conformance by allowing conditions defined in other interfaces to coexist with a default function implementation coming
from a different interface.

This FLIP proposes to relax this restriction further, by also allowing empty function declarations defined in other
interfaces to coexist with a default function implementation coming from a different interface.

## Motivation

Assume there are two interfaces, `Receiver` interface declares an empty `isSupportedVaultType` function.
The  second interface `Vault` conforms to the `Receiver` interface and provides a default implementation
to the `isSupportedVaultType` function.

```cadence
access(all) resource interface Receiver {
    access(all) view fun isSupportedVaultType(): {Type: Bool}
}

access(all) resource interface Vault: Receiver {    // Static error

    access(all) view fun isSupportedVaultType(): {Type: Bool} {
        // Implementation...
    }
}

access(all) resource VaultImpl: Vault {}
```

Currently, this reports an error saying `` `isSupportedVaultType` function of `Vault` conflicts with a function with the same name in `Receiver` ``.

However, it is possible to make this work by adding a dummy condition to the function in `Receiver`.
e.g.

```cadence
access(all) resource interface Receiver {
    access(all) view fun isSupportedVaultType(): {Type: Bool} {
        pre { true }
    }
}

access(all) resource interface Vault: Receiver {    // OK

    access(all) view fun isSupportedVaultType(): {Type: Bool} {
        // Implementation...
    }
}

access(all) resource VaultImpl: Vault {}
```

Likewise, restricting empty declarations from coexisting with default functions doesn't really add any value or safety,
as it can be workaround by adding a condition. Rather, it only reduces the composability.

## User Benefit

It can be useful to have interfaces that only declare the function, while another interface defines default functions.
Relaxing the current restriction allows user to use this pattern.

When interface inheritance is supported, the chance of running into similar situations would be more frequent.

## Design Proposal

Here it is proposed to relax the current restriction and allow emtpy function declarations defined in one interface to
coexist with a default function implementation coming from another interface.

```cadence
access(all) resource interface Receiver {
    access(all) view fun isSupportedVaultType(): {Type: Bool}
}

access(all) resource interface Vault: Receiver {     // This would be valid

    access(all) view fun getSupportedVaultTypes(): {Type: Bool} {
        // Implementation...
    }
}

access(all) resource VaultImpl: Vault {}             // This would be valid
```

Note that, above would be equivalent to, and hence is also allowed to have:

```cadence
access(all) resource interface Receiver {
    access(all) view fun isSupportedVaultType(): {Type: Bool}
}

access(all) resource interface Vault {
    access(all) view fun getSupportedVaultTypes(): {Type: Bool} {
        // Implementation...
    }
}

access(all) resource VaultImpl: Vault, Receiver {}    // This would be valid
```

However, to avoid any confusion of overriding of default functions, forbid defining an empty function declaration in the
descendant, if there is already a default function from an inherited interface. i.e:

```cadence
access(all) resource interface Receiver {
    access(all) view fun isSupportedVaultType(): {Type: Bool} {
        // Implementation...
    }
}

access(all) resource interface Vault: Receiver {    // Static error
    access(all) view fun isSupportedVaultType(): {Type: Bool}
}
```

### Drawbacks

Some concerns were discussed in: https://github.com/onflow/cadence/issues/2471

### Alternatives Considered

None

### Performance Implications

None

### Dependencies

None

### Engineering Impact

This change is trivial in the implementation.

### Compatibility

This proposal suggests a relaxation of an existing restriction. Hence, no existing codes would break.

### User Impact

None

## Related Issues

None

## Prior Art

None

## Questions and Discussion Topics

None

