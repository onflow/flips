---
status: draft
flip: [233](https://github.com/onflow/flips/pull/233)
authors: Giovanni Sanchez (giovanni.sanchez@dapperlabs.com)
sponsor: Jerome Pimmel (jerome.pimmel@dapperlabs.com)
updated: 2023-12-22
---

# FLIP 233: Flow VM Bridge. A contract protocol enabling arbitrary token bridging between Flow and EVM on Flow VMs

## Objective

This proposal outlines a contract-based protocol enabling the automated bridging of arbitrary Fungible (FT) and 
Non-Fungible tokens (NFT) from Cadence into EVM on Flow into the corresponding ERC-20 and ERC-721 token types. In the 
opposite direction, it supports bridging of arbitrary EVM on Flow ERC-20 and ERC-721 tokens into the corresponding 
Cadence FT or NFT token types. To facilitate users bridging tokens between VMs the protocol internalizes capabilities 
to deploy new token contracts in either VM state as needed. It serves as a request router & corresponding contract 
registrar to guarantee the synchronization integrity of assets being bridged across VM states. It additionally automates 
accounts & contracts to enforce source VM asset escrow and target VM token mint or unlock. `CadenceOwnedAccounts` (COAs)
introduced in the [EVM support FLIP proposal](https://github.com/onflow/flips/pull/225) enable the Flow VM Bridge to 
operate across both state spaces within the same, atomic transaction resulting in instantaneous asset exchange. 

## Motivation

The success and viability of EVM on Flow depends on the ability for assets to flow unimpeded between VM states. The Flow 
VM bridge resolves the need for token portability at the platform level. Its design is consistent with cross-chain 
bridging protocols common in web3 in order to ensure on-chain transparency. A generalized solution which addresses 
scalability and security concerns arising from arbitrary token bridging can better ensure the security of users on the
platform.

## User Benefit

An efficient, easy to use bridge which modifies state across VMs simultaneously in a single transaction reduces 
complexity for builders and improves end-user experience. The availability of a secure and proven platform capability 
for cross-VM bridging is more secure than leaving developers to implement project specific token bridges. It also 
significantly reduces the effort required of developers who can instead focus on core application logic. 

## Bridge Specification

### Cadence to EVM
Breakdown of the flow for a user bridging a token across VMs from Cadence to EVM. 

#### VM bridge contract functionality

* Configure token & deploy EVM contract ABI
* Maintain Flow <-> EVM contract relationships
* Provide utility methods for information lookup about contracts for either state space

#### Prerequisites to user actions

1. Confirm counterparty EVM contract deployed state, else deploy
2. Setup and configure NFT metadata
3. Safety/integrity checks as needed

#### Cadence transaction: bridge a Cadence NFT to CadenceOwnedAccount

1. Ensure prerequisites
2. Store NFT into VM bridge Cadence contract escrow
3. EVM transaction: unlock, or mint, corresponding NFT in EVM contract

### EVM to Cadence
Sequential breakdown of the flow for a user bridging a token from EVM to Flow. The high level paths below are broken down into their respective forks to help with understanding.

#### VM bridge contract functionality

* Initialize Cadence NFT collections
* Maintain Flow <-> EVM contract relationships
* Provide utility methods for information lookup about contracts for either state space

**Path 1: Cadence originated NFT returning to Cadence**

#### Cadence transaction: COA A bridges an EVM NFT back to Cadence

* COA A calls bridge to request bridging of EVM NFT back into Cadence providing requested NFT type, id, and verification of COA ownership
* Bridge contract calls into EVM to confirm COA A is the owner of the requested NFT. If so, process continues; otherwise reverts
* Bridge contract calls into EVM contract to burn EVM NFT
* Bridge contract unlocks corresponding Cadence NFT from VM bridge contract escrow storage
* Returns NFT on call to bridge contract

#### Cadence transaction: COA B bought COA A's EVM NFT on FlowEVM, and then wants to bridge it back to Cadence

* COA B calls bridge contract to request bridging of EVM NFT back into Cadence providing requested NFT type, id, and verification of COA ownership
* Bridge contract calls into EVM to confirm COA B is the owner of the requested NFT. If so, process continues; otherwise reverts
* Bridge contract calls into EVM contract to burn EVM NFT
* Contract unlocks corresponding Cadence NFT from VM bridge contract escrow storage
* Returns NFT on call to bridge contract

**Path 2: EVM originated NFT bridging to Cadence**

#### Prerequisite steps to user actions

* VM bridge checks, else creates new NFT contract/collection if not registered
* Setup and configure NFT metadata

#### Cadence transaction: COA A wants to bridge an EVM NFT to Cadence

* Ensure prerequisites 
* Upon Cadence call to bridge, COA A must include EVM transaction transferring NFT ownership to bridge account
* Bridge checks that caller is the owner of the NFT on EVM side before transferring to Bridge account COA address
* Bridge executed provided contract call, transferring NFT to Bridge COA address
* Bridge validates its ownership of the NFT on EVM side after transfer, thus locking NFT to be bridged
* Bridge mints NFT from Flow bridge NFT contract and returns to caller
* Collection is configured if necessary and deposited to COA A's account

## Design Proposal

### Context


### Overview

### Implementation Details

#### Interfaces

### Considerations

### Drawbacks

### Considered Alternatives

### Performance Implications


### Best Practices

### Examples

### Compatibility

## Prior Art

## Questions & Discussion Topics
