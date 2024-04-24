---
status: approved
flip: 242
authors: Austin Kline (austin@flowty.io)
sponsor: Bastian Müller
updated: 2024-01-23
---

# FLIP 242: Improve Public Capability Acquisition

## Objective

This FLIP proposes some improvements to the way that public capabilities are obtained with the new Capability Controllers API when no capability is published under the requested path.

## Motivation

When Cadence 1.0 goes live, Capability Controllers will be the new way to obtain and manage capabilities. With it, comes a change to what happens if a public capability is requested, using the function `Account.Capabilities.get`, and no capability is published under the requested path.
Whereas before you would get a capability which would simply fail to borrow (unless it was later linked), now a nil object will be returned.

This will present some challenges for existing applications which use capabilities even if they are not borrowable. This includes:

1. [Royalties](https://github.com/onflow/flow-nft/blob/master/contracts/MetadataViews.cdc#L303) - The Royalties Metadata standard assumes you can build a capability, there is no standard based on the kind of token you want to send a user. As such, many
platforms which follow the royalties standard take the return capability's address and reconstruct the correct one afterwards.
2. [Lost and Found](https://github.com/Flowtyio/lost-and-found/blob/main/contracts/LostAndFound.cdc#L720) - Helps users distribute resources to accounts. If a provided capability is not valid
(`check` returns false), it will store the resource for a user to redeem at a later date. If Capabilities now return nil when they aren't present, this utility contract will be much more challenging
to use. Lost and Found is used by a few marketplace platforms to ensure that things like Royalties can be distributed even if the receiving account is not configured, it is a safeguard against malicious
behavior as well for other contracts like Flowty Loans.

## User Benefit

There is value in a Capability not being nil, even if it is not borrowable. 
Nil is not the same as invalid, because we still have the address field which is something that is used often by developers.
Modifying the behavior for obtaining capabilities to instead give an invalid capability allows developers to make use of that address and react to something not being valid. 
It also introduces fewer panic scenarios that are out of general-purpose platforms' control (like if an NFT Collection panics on Royalty creation because of this). 

Even an invalid capability that cannot be borrowed is still valuable, as it provides the address of the account.
## Design Proposal

The return type of the function `Account.Capabilities.get` is changed from the optional type `Capability<T>?` to the non-optional type `Capability<T>`.
When the no capability is published under the requested path, or when the type argument to `get` is not a subtype of the runtime type of the capability published under the requested path, the function returns an *invalid* capability.

This "invalid" capability should:

1. Always return `false` when `check` is called.
2. Always return `nil` when `borrow` is called.
3. Have an ID of 0.
4. Have a runtime type that is the same as the type requested in the type argument of `get`

Capability Controller IDs are stated to be a *non-zero*, which 
means the value of zero can be used to mark capabilities as invalid.

## Alternatives Considered

It was suggested that anything which is expecting a non-nil Capability could instead request their own struct which would be carried along with the Capability itself

```
struct OptCap {
    let address: Address
    let cap: Capability? 
}
```

This might work in cases where the method being called is entirely in control of the caller, but it will not cover cases like Royalties where they are being blindly resolved using the 
Flow Metadata Standard. In cases like that, the caller has no way to protect itself again an optional capability that will panic when being created


## Engineering Effort

The proposed changes require little effort to implement in Cadence, but will require some changes in contracts already staged/updated to Cadence 1.0: Some optional checking code that had to be added will continue to work, while other will need to get removed/reverted.

In particular:
- Most `nil` handling code might still work, e.g. `nil` comparison, force-unwrapping
- Optional chaining and nil-coalescing don't work on non-optional types, so need to be undone
## Questions and Discussion

There is some [context in Discord](https://discord.com/channels/613813861610684416/621847426201944074/1194733333658218647) which led to the creation of this FLIP.