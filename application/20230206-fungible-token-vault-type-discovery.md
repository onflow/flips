---
status: implemented
flip: 69
authors: Satyam Agrawal (satyam.agrawal@dapperlabs.com)
updated: 2023-02-06
---

# FLIP 69: Fungible Tokens Vault Type Discovery

## Objective

The purpose of this proposal is to enable the Fungible Token (FT) `Receiver`s to return a list of vault types that they can accept.
This would enhance the discoverability of FT vault types and reduce the likelihood of unintended failures in the deposit method because the deposited vault is not supported.
This is because Cadence code trying to perform a deposit would now be able to query the acceptable vault types before making a deposit.

## Motivation

The [Fungible Token Standard](https://github.com/onflow/flow-ft) mandates that a single Vault can only receive one type of token,
meaning that a Flow Vault can only receive Flow tokens and a FUSD Vault can only receive FUSD tokens.
The `FungibleToken.Receiver` interface, on the other hand, is designed to be able to accept any Vault type in its `deposit` method.
Currently, there is no programmatic way to determine the type of token that a `Receiver` is able receive, leading to the possibility of a failed deposit due to an incorrect assumption about the Vault that the `Receiver` is linked to.
To address this issue, a new proxy receiver, the [`FungibleTokenSwitchboard`](https://github.com/onflow/flow-ft/blob/master/contracts/FungibleTokenSwitchboard.cdc), was created to allow for the receipt of multiple Vault types through a single `FungibleToken.Receiver` capability.
The [`SwitchboardPublic`](https://github.com/onflow/flow-ft/blob/4416bbe585629671d00d3acfa6fd8052104dd861/contracts/FungibleTokenSwitchboard.cdc#L37) interface also includes the `getVaultTypes` function, which solves the problem of knowing what vaults the receiver accepts, but only for the switchboard and only for code that is aware of the switchboard.
This solution is not complete. A `{FungibleToken.Receiver}` capability can represent any type of receiver such the `FungibleTokenSwitchboard`, a standard receiver, or any other custom `Receiver`.
As the Flow ecosystem expands, the number and complexity of proxy receivers developed to solve various problems is likely to increase.

The current `FungibleToken.Receiver` interface only contains the `deposit` method. It does not offer a clear-cut method for knowing the `Vault` types that a particular receiver is able to receive.
This issue is compounded by the fact that proxy receivers, which implement the `FungibleToken.Receiver` interface, can also return `Capability<&{FungibleToken.Receiver}>`.
As a result, the true nature of the receiver, whether it be derived from the standard Vault or a proxy receiver, remains ambiguous.

## User Benefit

Adding type information to `Receiver` capabilities has the potential to streamline the FT receiving process and minimize transaction failures.
This is because developers now have the ability to programmatically verify the compatibility between the given receiver capability and the intended Vault type,
allowing them to take proactive measures accordingly such as depositing the tokens elsewhere instead of the transaction just failing.

## Design Proposal

The essence of this proposal lies in modifying the `FungibleToken.Receiver` interface to include a method to return the expected `Vault` types that it can receive and include a default implementation so that the upgrade will not be breaking for existing fungible tokens.

So new `Receiver` interface would look like this -
```cadence
    /// The interface that enforces the requirements for depositing
    /// tokens into the implementing type.
    ///
    /// We do not include a condition that checks the balance because
    /// we want to give users the ability to make custom receivers that
    /// can do custom things with the tokens, like split them up and
    /// send them to different places.
    ///
    pub resource interface Receiver {

        /// Takes a Vault and deposits it into the implementing resource type
        ///
        /// @param from: The Vault resource containing the funds that will be deposited
        ///
        pub fun deposit(from: @Vault)

        /// Returns the type of implementing resource i.e If `FlowToken.Vault` implements
        /// this then it would return `{Type<@FlowToken.Vault>(): true}` and if any custom receiver
        /// uses the default implementation then it would return empty dictionary as its parent
        /// resource doesn't conform with the `FungibleToken.Vault` resource.
        ///
        /// @return Dictionary of supported vault types by the implemented resource.
        ///
        pub fun getSupportedVaultTypes() :{Type: Bool} {
            // Below check is implemented to make sure that run-time type would
            // only get returned when the parent resource conforms with `FungibleToken.Vault`.
            if self.getType().isSubtype(of: Type<@FungibleToken.Vault>()) {
                return {self.getType(): true}
            } else {
                // Return empty dictionary as the default value for the resource who don't
                // implements `FungibleToken.Vault`, such as `FungibleTokenSwitchboard`, `TokenForwarder` etc.
                return {}
            }
        }
    }
```

while the existing interface can be accessed [here](https://github.com/onflow/flow-ft/blob/4416bbe585629671d00d3acfa6fd8052104dd861/contracts/FungibleToken.cdc#L105).

The proposed interface features a default implementation of the `getSupportedVaultTypes` function, ensuring that existing FTs will not encounter any breakage and will have a ready-made implementation available. This default implementation also eliminates the need for upgrading existing FTs.

### Examples
Implementation of proposed `Receiver` type for `FungibleTokenSwitchboard` contract.

```cadence
    /// The resource that stores the multiple fungible token receiver
    /// capabilities, allowing the owner to add and remove them and anyone to
    /// deposit any fungible token among the available types.
    ///
    pub resource Switchboard: FungibleToken.Receiver, SwitchboardPublic {

        /// Dictionary holding the fungible token receiver capabilities,
        /// indexed by the fungible token vault type.
        ///
        access(contract) var receiverCapabilities: {Type: Capability<&{FungibleToken.Receiver}>}

        // ...
        // snip
        // ...

        /// A getter function to know which tokens a certain switchboard
        /// resource is prepared to receive.
        ///
        /// @return The keys from the dictionary of stored
        /// `{FungibleToken.Receiver}` capabilities that can be effectively
        /// borrowed.
        ///
        pub fun getSupportedVaultTypes(): {Type: Bool} {
            let supportedTypes: {Type: Bool} = []
            for vaultType in self.receiverCapabilities.keys {
                if self.receiverCapabilities[vaultType]!.check() {
                    supportedTypes.insert(key: vaultType, true)
                }
            }
            return supportedTypes
        }
    }
```

### Alternatives Considered

There are no equivalent alternatives to the proposed solution. While some options such as implementing the [`PaymentHandler`](https://github.com/onflow/Offers/blob/offers-implementation/contracts/PaymentHandler.cdc) exist, they are overly complex and do not offer the same level of flexibility, comprehensiveness, and ease of comprehension as the proposed solution.

### Performance Implications

There is no real performance implications on proposed solution.

### Dependencies

This proposal builds on the [existing FT interface](https://github.com/onflow/flow-ft).

### Engineering Impact

Adhering to this standard will require contract authors to implement a new methods as part of their FT. However, default implementation covers the need of all FTs.

### Best Practices

Once the proposed solution is adopted then it is recommended to validate vault type before transferring the FTs from one account to another.

### Compatibility

All the custom receiver contracts owners will need to update them to add implementations for the new methods.

### User Impact

This feature can be rolled out with no fear of changes to the user.

## Prior Art

<TBD>

## Questions and Discussion Topics

<TBD>
