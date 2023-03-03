# AuthAccount Capabilities

| Status        | Proposed                                             |
:-------------- |:---------------------------------------------------- |
| **FLIP #**    | [53](https://github.com/onflow/flips/pull/53)        |
| **Author(s)** | Bastian Müller (bastian@dapperlabs.com)              |
| **Sponsor**   | Bastian Müller (bastian@dapperlabs.com)              |
| **Updated**   | 2022-03-02                                           |

## Objective

This FLIP proposes adding a new feature to Cadence: AuthAccount capabilities.
Similar to how storage capabilities allow gaining access to a value stored in an account's storage,
AuthAccount capabilities allow gaining access to an AuthAccount.

## Motivation

Currently, capabilities in Cadence are limited to values stored at the top level on an account.
Users may also want to share access to their account, for example, to perform storage operations.

## User Benefit

User will benefit from this new feature indirectly:
Developers will benefit by having the ability to implement functionality which allows an account to access another,
in their smart contracts, dapps, user agents, etc.

## Design Proposal

This proposal suggest adding a new function to the type `AuthAccount`:

```cadence
fun linkAccount(_ newCapabilityPath: PrivatePath): Capability<&AuthAccount>?
```

This new function behaves like the existing function `link`, but

- It creates a link to the account, instead of creating a link to a value in account storage.
- It only supports private paths (`/private`).

Existing functions that work with links/capabilities, like `getCapability`, `borrow`, `unlink`, etc., can be used with account capabilities,
just like they currently can be used for storage path links.

### Drawbacks

This proposal suggests adding a very powerful feature: An account capability allows full access to the account,
i.e. full access to manage storage, keys, contracts, etc.

### Alternatives Considered

One alternative is to deploy a "proxy" contract to the account which enables such functionality:

```cadence
pub contract AccountProxy {
    access(contract) getAuthAccount(): AuthAccount {
        return self.account
    }

    pub resource ProxyAdmin {
        fun getAuthAccount(): AuthAccount {
            return AccountProxy.getAuthAccount()
        }
    }
}
```

Contracts already have access to the account which they are deployed to through the `account` field.
First, the contract could expose the account object through a function, in a controlled manner,
by leveraging the `access(contract)` access modifier, which only gives code in the same contract access.
Next, the contract could define a resource with a function which in turn calls the contract function which provides access to the account.
Finally, the user could store such a resource in their storage and use the existing storage capability mechanism to share access to the resource,
which in turn can be used to get the account.

However, this would require each account to have a contract deployed to it.

### Performance Implications

None

### Dependencies

None

### Engineering Impact

This change has already implementated and no further engineering impact is expected.

### Compatibility

This change has no impact on compatibility between systems (e.g. SDKs).

### User Impact

The proposed feature is a purely additive.
There is no impact on existing contracts and new transactions.

## Related Issues

None

## Questions and Discussion Topics

None

## Implementation

The proposed feature was implemented in https://github.com/onflow/cadence/pull/2160 and is currently gated behind a feature flag (disabled by default).
