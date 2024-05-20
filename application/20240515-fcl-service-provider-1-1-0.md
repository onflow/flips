---
status: draft 
flip: 270
authors: Jordan Ribbink (jordan.ribbink@flowfoundation.org)
sponsor: Greg Santos (g@flowfoundation.org)
updated: 2023-05-15 
---

# FLIP 270: FCL `ServiceProvider` v1.1.0

## Objective

The objective of this FLIP is to introduce a new optional metadata field into the FCL `ServiceProvider` data structure to allow for more descriptive wallet provider identification, and ultimately empowering FCL clients to offer more complete user experiences.  In particular, this proposal aims to enahance FCL's integration with WalletConnect-compatible wallets by including identifying information in the `ServiceProvider` data structure.

## Motivation

In recent years, WalletConnect has become an integral part of not only the Flow wallet ecosystem but also the broader Web3 ecosystem. Especially in the context of mobile wallets, WalletConnect significantly reduces the infrastructure requirements for wallet providers to offer a secure and user-friendly experience.

The current integration with WalletConnect in the FCL protocol is less streamlined than others seen throughout Web3 (e.g., Web3Modal). Specifically, `WC/RPC` authentication do not identify which WalletConnect-compatible wallet it is that they correspond to. In practice, this creates an unnecessary hurdle where users must complete WalletConnect's wallet selection process, in addition to already selecting a particular wallet provider in FCL Discovery.

## User Benefit

Users will tangibly benefit from the streamlined UI enabled by this proposal.  WalletConnect-compatible wallets (e.g. Flow Wallet) are at the forefront of Flow's self-custodial ecosystem, and this proposal will allow users to more easily interact with their preferred FCL wallet provider.

## Design Proposal

This proposal adds an optional field to the `ServiceProvider` data structure, adding a new optional field `walletConnectId`.  This should be a string equivalent to the 32-byte hex-identifier used by WalletConnect to identify wallets.

The new `ServiceProvider` data structure will look as follows:

```typescript
interface ServiceProvider {
    f_vsn: "1.1.0"
    f_type: "ServiceProvider"
    address: string
    name?: string
    description?: string
    icon?: string
    website?: string
    supportUrl?: string
    supportEmail?: string
    walletConnectId?: string // NEW FIELD
}
```

### Drawbacks

Minimal drawbacks are expected from this proposal.  The `ServiceProvider` change is non-breaking, so it will not impact existing FCL clients.

Possible hesitations for including this change could be complicating the `ServiceProvider` data structure, or unnecessary coupling with WalletConnect within the FCL protocol. However, given the minimal and optional nature of this change, these drawbacks are expected to be negligible.

### Alternatives Considered

Do not make this change and do not link FCL authentication services to their accompanying WalletConnect identifiers.  This would maintain the status quo, but would not offer the user experience benefits of this proposal.

### Performance Implications

N/A

### Dependencies

This

### Engineering Impact

Negligible impact, optional inclusion for FCL service providers & optional consumption for FCL clients. 

### Best Practices

N/A

### Tutorials and Examples

N/A

### Compatibility

Non-breaking

### User Impact

No action required for end users.  Application developers will need to update to the latest version of FCL-WC to realize these benefits.

## Related Issues

N/A

## Prior Art

[Web3Modal](https://web3modal.com/) - once a user selects a wallet, they a brought to a connection view specific to the wallet they have chosen and do not need to re-select their wallet.

## Questions and Discussion

Discussion can take place on this FLIP's pull request.