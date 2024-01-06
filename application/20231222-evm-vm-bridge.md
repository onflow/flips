---
status: draft
flip: 237
title: Flow EVM VM Bridge
authors: Giovanni Sanchez (giovanni.sanchez@dapperlabs.com)
sponsor: Jerome Pimmel (jerome.pimmel@dapperlabs.com)
updated: 2023-12-22
---

# FLIP 237: Flow VM Bridge

> A contract protocol enabling arbitrary token bridging atomically between Flow and FlowEVM

<details>

<summary>Table of contents</summary>

- [Objective](#objective)
- [Motivation](#motivation)
- [User Benefit](#user-benefit)
- [Bridge Specification](#bridge-specification)
  - [Cadence to EVM](#cadence-to-evm)
  - [EVM to Cadence](#evm-to-cadence)
- [Context](#context)
- [Overview](#overview)
- [In Aggregate](#in-aggregate)
- [Implementation Details](#implementation-details)
  - [Contract Roles \& Concerns](#contract-roles--concerns)
  - [Case Studies](#case-studies)
  - [Interfaces](#interfaces)
  - [Considerations](#considerations)
  - [Drawbacks](#drawbacks)
  - [Considered Alternatives](#considered-alternatives)
  - [Performance Implications](#performance-implications)
  - [Examples](#examples)
  - [Compatibility](#compatibility)
- [Prior Art](#prior-art)
- [Questions \& Discussion Topics](#questions--discussion-topics)
</details>

## Objective

This proposal outlines a contract-based protocol enabling the automated bridging of arbitrary Fungible (FT) and
Non-Fungible tokens (NFT) from Cadence into FlowEVM into the corresponding ERC-20 and ERC-721 token types. In the
opposite direction, it supports bridging of arbitrary FlowEVM ERC-20 and ERC-721 tokens into the corresponding
Cadence FT or NFT token types.

To facilitate users bridging tokens between VMs, the protocol internalizes capabilities to deploy new token contracts in
either VM state as needed. It serves as a request router & corresponding contract registrar to guarantee the
synchronization integrity of assets being bridged across VM states. It additionally automates account and contract calls
to enforce source VM asset burn or lock and target VM token mint or unlock. `CadenceOwnedAccounts` (COAs) introduced
in the [EVM support FLIP proposal](https://github.com/onflow/flips/pull/225) enable the Flow VM Bridge to operate across
both state spaces within the same, atomic transaction resulting in instantaneous asset exchange. 

## Motivation

The success and viability of EVM on Flow depends on the ability for assets to move unimpeded between VM states. The Flow
VM bridge resolves the need for token portability at the platform level. Its design is consistent with cross-chain
bridging protocols common in Web 3 in order to ensure onchain transparency. A generalized solution which addresses
scalability and security concerns arising from arbitrary token bridging can better ensure the security of users on the
platform.

## User Benefit

An efficient, easy to use bridge which modifies state across VMs simultaneously in a single transaction reduces
complexity for builders and improves end-user experience for the essential economic activity of moving tokens across
VMs. The abstraction afforded to users by the bridge means that multiple EVM transaction steps can be bundled together
using Cadence to securely realize bridging as a single transaction. The availability of a secure and proven platform
capability for cross-VM bridging can ensure a consistent, higher security bar than leaving developers to implement
project specific token bridges. It also significantly reduces the effort required of developers wishing to build
applications which need to bridge tokens between VMs for an optimal user experience.

## Bridge Specification

Specification outline of the Flow VM bridge for bi-directional flow of FT and NFTs between VM states. While the spec
references NFTs, the VM bridge will treat both FTs and NFTs alike, with the caveat that NFTs have more complexity due to
metadata which needs to be made to work for both VMs.

### Cadence to EVM

Breakdown of the flow for a user bridging a token across VMs from Cadence to EVM. 

#### VM bridge contract functionality

* Configure EVM-defining contract if needed
* Maintain Flow <-> EVM contract relationships
* Provide utility methods for information lookup about contracts for either state space

#### Prerequisites to user actions

1. Confirm counterparty contract deployed state, else deploy
2. Setup and configure NFT metadata
3. Safety/integrity checks as needed

#### Cadence transaction: bridge a Cadence NFT to CadenceOwnedAccount

1. Ensure prerequisites
2. Store NFT into VM bridge Cadence contract escrow
3. EVM transaction: mint corresponding NFT in EVM contract

### EVM to Cadence

Sequential breakdown of the flow for a user bridging a token from EVM to Flow. The high level paths below are broken
down into their respective forks to help with understanding.

#### VM bridge contract functionality

* Initialize Cadence NFT contract
* Maintain Flow <-> EVM contract relationships
* Provide utility methods for information lookup about contracts for either state space

**Path 1: Cadence originated NFT returning to Cadence**

#### Cadence transaction: COA A bridges an EVM NFT back to Cadence

* COA A calls bridge to request bridging of EVM NFT back into Cadence
* Bridge contract calls into EVM to confirm COA A is the owner of the requested NFT. If so, process continues; otherwise
  reverts
* Bridge contract calls into EVM contract to burn EVM NFT
* Bridge contract unlocks corresponding Cadence NFT from VM bridge contract escrow storage
* Returns NFT on call to bridge contract

#### Cadence transaction: COA B bought COA A's EVM NFT on FlowEVM, and then wants to bridge it back to Cadence

* COA B calls bridge contract to request bridging of EVM NFT back into Cadence
* Bridge contract calls into EVM to confirm COA B is the owner of the requested NFT. If so, process continues; otherwise
  reverts
* Bridge contract calls into EVM contract to burn EVM NFT
* Contract unlocks corresponding Cadence NFT from VM bridge contract escrow storage
* Returns NFT on call to bridge contract

**Path 2: EVM originated NFT bridging to Cadence**

#### Prerequisite steps to user actions

* VM bridge checks, else creates new NFT contract/collection if not registered
* Setup and configure NFT metadata

#### Cadence transaction: COA A wants to bridge an EVM NFT to Cadence

* Ensure prerequisites 
* Upon Cadence call to bridge, COA A must include EVM transaction approving bridge account to act on the NFT
* Bridge checks that caller is the owner of the NFT on EVM side before executing approval to Bridge account COA address
* Bridge ensures approval was successful then completes the transfer to the Bridge COA address
* Bridge validates its ownership of the NFT on EVM side after transfer, thus locking NFT to be bridged
* Bridge mints NFT from Flow bridge NFT contract and returns to caller
* Collection is configured if necessary and deposited to COA A's account

# Design Proposal

## Context

It's helpful to understand that the entrypoint to FlowEVM is mediated by the [`CadenceOwnedAccount`
(COA)](https://github.com/onflow/flips/pull/225). This Cadence resource provides access to FlowEVM, enabling encoded
calls into EVM originating from the calling COA's EVM address. In short, it's a resource that also functions as an EVM
account whose access is controlled by resource ownership instead of an offchain signature.

Since COAs are the only cross-VM objects universally available from Cadence and the identity-based access of Solidity
enables any address to receive assets, callers bridging *to* FlowEVM may bridge to any EVM address. In the opposite
direction, only COAs may initiate bridging *from* EVM since there is not yet a mechanism to initiate Cadence state
change from the EVM environment.

Hopefully, this context clarifies that bridging in either direction - Flow -> EVM & EVM -> Flow - is at all times
initiated via call to the bridge's Cadence contracts.

It's also interesting to note that any project could build an asset-specific bridge between VMs using the same
primitives and toolsets leveraged for the design below. However, the intention for this bridge is to provide public
infrastructure to seamlessly and permissionlessly move assets between VMs without requiring the support of the asset
developers.

## Overview

The central bridge contract will act as a request router & corresponding contract registrar, additionally configuring
contracts on either side of the VM to facilitate bridge requests as they arrive. Deployed contracts are “owned” by the
bridge, but owner interactions are mediated by contract logic on either end in addition to multi-sig patterns consistent
with other core network infrastructure accounts.

On the EVM side, a central contract factory will instantiate Solidity ERC20 & ERC721 contracts as directed by calls from
the central Cadence contract’s COA. This factory will also implement a number of helper methods to give the bridge
account visibility into the EVM environment. These methods might include things like retrieving an asset type,
determining if EVM contracts are bridge-owned, validating asset ownership, etc. so the COA has a central trusted source
of truth for critical state assertions.

Diagrams depicting the call flows for both Flow- and EVM-native NFTs bridging according to the proposed design are included later in this FLIP. Immediately below is a birds-eye view of the contract suite across VMs.

## In Aggregate

![FlowEVM VM Bridge Design Overview](20231222-evm-vm-bridge-resources/overview.png)
*The bridge contract can be thought of here as a router for requests to bridge to and from Flow. It then handles the
request dependent on whether the asset is an NFT or FT and if it's Flow- or EVM-native. If needed, it performs contract
initialization on either side of the VM. From there, it routes requests to the appropriate contract which fulfills the
asset bridge request.*

## Implementation Details

### Contract Roles & Concerns

#### Flow

- **Bridge**
  - Unified entry point for bridging assets between VMs
  - Unified query point for NFT Locker contracts & to assess Flow x EVM contract associations
  - Owning (via contract COA) all deployed contracts on EVM side
  - Owning (via contract COA) all EVM-native assets on EVM side when bridged to Flow - equivalent to locking
  - Initializing NFT & FT Locker contracts when Flow-native assets are first bridged to EVM
  - Initializing BridgedFT & BridgedNFT contracts defining EVM-native assets when first bridged to Flow
- **FT/NFT Locker**
  - Lock Flow-native tokens bridging to EVM
  - Unlock Flow-native tokens bridging from EVM
  - Serve query requests about locked tokens
  - Point to the corresponding EVM-defining contract
- **Bridged FT/NFT**
  - Define EVM-native tokens bridged from EVM to Flow
  - Point to the corresponding EVM-native contract
  - Serve as secondary bridging interface, enabling easy bridging back to EVM
      - Result of implementing combination of `CrossVM` contract interface along with `EVMBridgeableVault` or
        `EVMBridgeableCollection` resource interfaces
      - e.g. `collection.bridgeToEVM(id: UInt64, to: EVMAddress, tollFee: @FlowToken.Vault)`
      - e.g. `vault.bridgeToEVM(amount: UFIx64, to: EVMAddress, tollFee: @FlowToken.Vault)`

#### EVM

> :information_source: Self-defined locking functionality is not required as “locked” assets will simply be transferred
> to the FlowEVMBridge.COA.EVMAddress

- **FlowBridgeFactory**
  - Deploys new instances of FlowBridgedFT/NFT
  - Maintains knowledge of all deployed contract addresses & their Flow token identifier correspondence
  - Aids visibility of bridge contract into the EVM environment with assistive methods

- **FlowBridgedFT/NFT**
  - Define Flow-native tokens bridged from Flow to EVM
  - Point to the corresponding Flow-native contract

### Case Studies

The task of bridging FTs & NFTs between VMs can be split into four distinct cases, each with their own unique path. An
asset can either be Flow- or EVM-native, and it can either be bridged from Flow to EVM or EVM to Flow. The following
sections outline an NFT bridge path for each case.

#### Flow-Native: Flow -> EVM

*Lock in Cadence & Mint in EVM*

1. Determine if asset is Flow or EVM-native
    - Check if resource is defined in bridge-hosted NFT contract - if so, EVM-native, else Flow-native
1. Determine if BridgeNFTLocker contract has been initialized for this NFT type - if not, initialize setup
    1. Call into EVM, telling FlowBridgeFactory to deploy a new FlowBridgedNFT (ERC721) contract, passing identifying
       info about the NFT and it's source contract. Note the new EVM contract address
    1. Derive a contract name from the NFT identifier, & deploy template-generated Cadence contract to the bridge
       account, passing the newly deployed EVM contract address
1. Borrow a reference to the newly deployed BridgeNFTLocker contract & passthrough bridge request
    1. Lock the NFT in the contract's Locker resource
    1. Call to FlowBridgedNFT.sol to mint to mint an NFT to the defined recipient
1. Locker contract calls corresponding EVM FlowBridgedNFT contract to mint NFT to the provided EVMAddress

![Flow-native Flow to EVM](20231222-evm-vm-bridge-resources/flow_native_to_evm.png)

#### Flow-Native: EVM -> Flow

*Unlock in Cadence & Burn in EVM*

1. Determine if asset is Flow or EVM-native
    - Call into Factory contract to determine if target of EVM call is FlowBridgedNFT.sol instance && was deployed by
      FlowBridgeFactory.sol
1. Get the Flow NFT type identifier from FlowBridgedNFT.sol
1. Determine if an NFT Locker has been initialized (it will be by construction)
1. Borrow the BridgeNFTLocker.cdc contract, deriving name from from the type identifier returned from FlowBridgedNFT.sol
1. Validate the caller is the current owner (or is approved) of the EVM NFT to be bridged
1. Execute the caller-provided approve() calldata
1. Validate the bridge contract COA is approved to manipulate the NFT in FlowBridgedNFT.sol as a result of the executed call
1. Bridge contract COA calls to FlowBridgedNFT.sol to burn the NFT
1. Bridge contract withdraws NFT from the locker contract and returns to caller

![Flow-native EVM to Flow](20231222-evm-vm-bridge-resources/flow_native_to_flow.png)

#### EVM-Native: EVM -> Flow

*Mint in Flow & Transfer in EVM*

1. Determine if asset is Flow or EVM-native
    - Call into Factory contract, checking if the target of EVM call is FlowBridgedNFT.sol instance && was deployed by
      FlowBridgeFactory.sol
1. Determine if BridgedNFT contract has been deployed - if not initialize
    - Derive a contract name from the EVM NFT contract address + name & deploy template-generated Cadence contract to
      bridge account, passing EVM source contract address
1. Borrow newly deployed BridgedNFT contract & passthrough
1. BridgedNFT validates the caller is currently owner (or is approved) of the EVM NFT to be bridged
1. BridgedNFT executes EVM approve call provided by the caller, approving bridge COA to act on the EVM NFT
1. BridgedNFT executes NFT transfer from bridge COA, completing the transfer to the bridge account in EVM
1. BridgedNFT validates its contract COA is now the owner of the NFT
1. BridgedNFT subsequently mints an NFT from itself & returns
1. Caller then creates & Collection from NFT & configures, finally depositing to their account

![Flow-native Flow to EVM](20231222-evm-vm-bridge-resources/evm_native_to_flow.png)

#### EVM-Native: Flow -> EVM

*Burn in Flow & Transfer in EVM*

1. Determine if asset is Flow or EVM-native
    - Check if resource is defined in bridge-hosted NFT contract - if so, EVM-native, else Flow-native
1. Determine if BridgedNFT contract has been deployed (it will be by construction)
1. Borrow the BridgedNFT contract from the NFT type identifier & passthrough bridge request
1. BridgedNFT burns the NFT
1. BridgedNFT calls to EVM contract, transferring NFT to defined recipient
1. BridgedNFT confirms recipient is owner of the NFT post-transfer

![Flow-native Flow to EVM](20231222-evm-vm-bridge-resources/evm_native_to_evm.png)

### Interfaces

> :information_source: Solidity contracts will largely be boilerplate based on common standards. Interfaces are soon
> to follow

<details>
<summary>FlowEVMBridge.cdc.cdc</summary>

```cadence
access(all) contract FlowEVMBridge {

    /// Amount of $FLOW paid to bridge
    access(all) var fee: UFix64
    /// The COA which orchestrates bridge operations in EVM
    access(self) let coa: @EVM.BridgedAccount

    /// Denotes a contract was deployed to the bridge account, could be either FlowEVMBridgeLocker or FlowEVMBridgedAsset
    access(all) event BridgeContractDeployed(type: Type, name: String, evmContractAddress: EVM.EVMAddress)

    /* --- Public NFT Handling --- */

    /// Public entrypoint to bridge NFTs from Flow to EVM - cross-account bridging supported
    ///
    /// @param token: The NFT to be bridged
    /// @param to: The NFT recipient in FlowEVM
    /// @param tollFee: The fee paid for bridging
    ///
    access(all) fun bridgeNFTToEVM(token: @{NonFungibleToken.NFT}, to: EVM.EVMAddress, tollFee: @FlowToken.Vault) {
        pre {
            tollFee.balance == self.tollAmount: "Insufficient fee paid"
            asset.isInstance(of: Type<&{FungibleToken.Vault}>) == false: "Mixed asset types are not yet supported"
        }
    }

    /// Public entrypoint to bridge NFTs from EVM to Flow
    ///
    /// @param caller: The caller executing the bridge - must be passed to check EVM state pre- & post-call in scope
    /// @param calldata: Caller-provided approve() call, enabling contract COA to operate on NFT in EVM contract
    /// @param id: The NFT ID to bridged
    /// @param evmContractAddress: Address of the EVM address defining the NFT being bridged - also call target
    /// @param tollFee: The fee paid for bridging
    ///
    access(all) fun bridgeNFTFromEVM(
        caller: auth(Callable) &BridgedAccount,
        calldata: [UInt8],
        id: UInt64,
        evmContractAddress: EVM.EVMAddress,
        tollFee: @FlowToken.Vault
    ): @{NonFungibleToken.NFT} {
        pre {
            tollFee.balance == self.tollAmount: "Insufficient fee paid"
            FlowEVMBridgeUtils.isEVMNFT(evmContractAddress: evmContractAddress): "Unsupported asset type"
            FlowEVMBridgeUtils.isOwnerOrApproved(ofNFT: id, owner: caller.address(), evmContractAddress: evmContractAddress):
                "Caller is not the owner of or approved for requested NFT"
        }
    }

    /* --- Public FT Handling --- */

    /// Public entrypoint to bridge NFTs from Flow to EVM - cross-account bridging supported
    ///
    /// @param vault: The FungibleToken Vault to be bridged
    /// @param to: The recipient of tokens in FlowEVM
    /// @param tollFee: The fee paid for bridging
    ///
    access(all) fun bridgeTokensToEVM(vault: @{FungibleToken.Vault}, to: EVM.EVMAddress, tollFee: @FlowToken.Vault) {
        pre {
            tollFee.balance == self.tollAmount: "Insufficient fee paid"
            asset.isInstance(of: Type<&{NonFungibleToken.NFT}>) == false: "Mixed asset types are not yet supported"
        }
        // Handle based on whether Flow- or EVM-native & passthrough to internal method
    }

    /// Public entrypoint to bridge fungible tokens from EVM to Flow
    ///
    /// @param caller: The caller executing the bridge - must be passed to check EVM state pre- & post-call in scope
    /// @param calldata: Caller-provided approve() call, enabling contract COA to operate on tokens in EVM contract
    /// @param amount: The amount of tokens to bridge
    /// @param evmContractAddress: Address of the EVM address defining the tokens being bridged, also call target
    /// @param tollFee: The fee paid for bridging
    ///
    access(all) fun bridgeTokensFromEVM(
        caller: auth(Callable) &BridgedAccount,
        calldata: [UInt8],
        amount: UFix64,
        evmContractAddress: EVM.EVMAddress,
        tollFee: @FlowToken.Vault
    ): @{FungibleToken.Vault} {
        pre {
            tollFee.balance == self.tollAmount: "Insufficient fee paid"
            FlowEVMBridgeUtils.isEVMToken(evmContractAddress: evmContractAddress): "Unsupported asset type"
            FlowEVMBridgeUtils.hasSufficientBalance(amount: amount, owner: caller, evmContractAddress: evmContractAddress):
                "Caller does not have sufficient funds to bridge requested amount"
        }
    }

    /* --- Public Getters --- */

    /// Returns the bridge contract's COA EVMAddress
    access(all) fun getBridgeCOAEVMAddress(): EVM.EVMAddress
    /// Retrieves the EVM address of the contract related to the bridge contract-defined asset
    /// Useful for bridging flow-native assets back from EVM
    access(all) fun getAssetEVMContractAddress(type: Type): EVM.EVMAddress?
    /// Retrieves the Flow address associated with the asset defined at the provided EVM address if it's defined
    /// in a bridge-deployed contract
    access(all) fun getAssetFlowContractAddress(evmAddress: EVM.EVMAddress): Address?

    /* --- Internal Helpers --- */

    // Flow-native NFTs - lock & unlock

    /// Handles bridging Flow-native NFTs to EVM - locks NFT in designated Flow locker contract & mints in EVM
    /// Within scope, locker contract is deployed if needed & passing on call to said contract
    access(self) fun bridgeFlowNativeNFTToEVM(token: @{NonFungibleToken.NFT}, to: EVM.EVMAddress, tollFee: @FlowToken.Vault)
    /// Handles bridging Flow-native NFTs from EVM - unlocks NFT from designated Flow locker contract & burns in EVM
    /// Within scope, locker contract is deployed if needed & passing on call to said contract
    access(self) fun bridgeFlowNativeNFTFromEVM(
        caller: auth(Callable) &BridgedAccount,
        calldata: [UInt8],
        id: UInt256,
        evmContractAddress: EVM.EVMAddress
        tollFee: @FlowToken.Vault
    )

    // EVM-native NFTs - mint & burn

    /// Handles bridging EVM-native NFTs to EVM - burns NFT in defining Flow contract & transfers in EVM
    /// Within scope, defining contract is deployed if needed & passing on call to said contract
    access(self) fun bridgeEVMNativeNFTToEVM(token: @{NonFungibleToken.NFT}, to: EVM.EVMAddress, tollFee: @FlowToken.Vault)
    /// Handles bridging EVM-native NFTs to EVM - mints NFT in defining Flow contract & transfers in EVM
    /// Within scope, defining contract is deployed if needed & passing on call to said contract
    access(self) fun bridgeEVMNativeNFTFromEVM(
        caller: auth(Callable) &BridgedAccount,
        calldata: [UInt8],
        id: UInt256,
        evmContractAddress: EVM.EVMAddress
        tollFee: @FlowToken.Vault
    )

    // Flow-native FTs - lock & unlock

    /// Handles bridging Flow-native assets to EVM - locks Vault in designated Flow locker contract & mints in EVM
    /// Within scope, locker contract is deployed if needed
    access(self) fun bridgeFlowNativeTokensToEVM(vault: @{FungibleToken.Vault}, to: EVM.EVMAddress, tollFee: @FlowToken.Vault)
    /// Handles bridging Flow-native assets from EVM - unlocks Vault from designated Flow locker contract & burns in EVM
    /// Within scope, locker contract is deployed if needed
    access(self) fun bridgeFlowNativeTokensFromEVM(
        caller: auth(Callable) &BridgedAccount,
        calldata: [UInt8],
        amount: UFix64,
        evmContractAddress: EVM.EVMAddress,
        tollFee: @FlowToken.Vault
    )

    // EVM-native FTs - mint & burn

    /// Handles bridging EVM-native assets to EVM - burns Vault in defining Flow contract & transfers in EVM
    /// Within scope, defining contract is deployed if needed
    access(self) fun bridgeEVMNativeTokensToEVM(vault: @{FungibleToken.Vault}, to: EVM.EVMAddress, tollFee: @FlowToken.Vault)
    /// Handles bridging EVM-native assets from EVM - mints Vault from defining Flow contract & transfers in EVM
    /// Within scope, defining contract is deployed if needed
    access(self) fun bridgeEVMNativeTokensFromEVM(
        caller: auth(Callable) &BridgedAccount,
        calldata: [UInt8],
        amount: UFix64,
        evmContractAddress: EVM.EVMAddress,
        tollFee: @FlowToken.Vault
    )
    
    /// Helper for deploying templated Locker contract supporting Flow-native asset bridging to EVM
    /// Deploys either NFT or FT locker depending on the asset type
    access(self) fun deployLockerContract(asset: &AnyResource)
    /// Helper for deploying templated defining contract supporting EVM-native asset bridging to Flow
    /// Deploys either NFT or FT contract depending on the provided type
    access(self) fun deployDefiningContract(type: Type)

    /// Enables other bridge contracts to orchestrate bridge operations from contract-owned COA
    access(account) fun borrowCOA(): &EVM.BridgedAccount
}
```

</details>

<details>
<summary>FlowEVMBridgeUtils.cdc</summary>

```cadence
/// Util contract serving all bridge contracts
access(all) contract FlowEVMBridgeUtils {

    /// Commonly used Solidity function selectors to call into EVM from bridge contracts to support call encoding
    /// e.g. ownerOf(uint256)(address), getApproved(uint256)(address), mint(address, uint256), etc.
    access(self) let functionSelectors: {String: [UInt8]}

    /// Returns the requested function selector
    access(all) fun getFunctionSelector(signature: String): [UInt8]?

    /// Identifies if an asset is Flow- or EVM-native, defined by whether a bridge contract defines it or not
    access(all) fun isFlowNative(asset: &AnyResource): Bool
    /// Identifies if an asset is Flow- or EVM-native, defined by whether a bridge-owned contract defines it or not
    access(all) fun isEVMNative(evmContractAddress: EVM.EVMAddress): Bool
    /// Identifies if an asset is ERC721 && not ERC20
    access(all) fun isEVMNFT(evmContractAddress: EVM.EVMAddress): Bool
    /// Identifies if an asset is ERC20 and not ERC721
    access(all) fun isEVMToken(evmContractAddress: EVM.EVMAddress): Bool

    /// Determines if the owner is in fact the owner of the NFT at the ERC721 contract address
    access(all) fun isOwnerOrApproved(ofNFT: UInt64, owner: EVM.EVMAddress, evmContractAddress: EVM.EVMAddress): Bool
    /// Determines if the owner has sufficient funds to bridge the given amount at the ERC20 contract address
    access(all) fun hasSufficientBalance(amount: UFix64, owner: EVM.EVMAddress, evmContractAddress: EVM.EVMAddress): Bool

    /// Derives the Cadence contract name for a given Type
    access(all) fun deriveLockerContractName(fromType: Type): String?
    /// Derives the Cadence contract name for a given EVM asset
    access(all) fun deriveBridgedAssetContractName(fromEVMContract: EVM.EVMAddress): String?
    
    /// Deposits fees to the bridge account's FlowToken Vault - helps fund asset storage
    access(account) fun depositTollFee(_ tollFee: @FlowToken.Vault)
    /// Upserts the function selector of the given signature
    access(account) fun upsertFunctionSelector(signature: String)
}
```
</details>

</details>

<details>
<summary>FlowEVMBridgeTemplates.cdc</summary>

```cadence
/// Helper contract serving templates
access(all) contract FlowEVMBridgeTemplates {
    /// Serves Locker contract code for a given type, deriving the contract name from the type identifier
    access(all) fun getLockerContractCode(forType: Type): [UInt8]?
    /// Serves bridged asset contract code for a given type, deriving the contract name from the EVM contract info
    access(all) fun getBridgedAssetContractCode(forEVMContract: EVM.EVMAddress): [UInt8]?
}
```
</details>

<details>
<summary>ICrossVM.cdc</summary>

```cadence
/// Contract interface denoting a cross-VM implementation, exposing methods to query EVM-associated addresses
access(all) contract interface ICrossVM {
    /// Retrieves the corresponding EVM contract address, assuming a 1:1 relationship between VM implementations
    access(all) fun getEVMContractAddress(): EVM.EVMAddress
    /// Retrieves the owner of the EVM contract address
    access(all) fun getEVMContractOwner(): EVM.EVMAddress
}
```
</details>

<details>
<summary>CrossVMAsset.cdc</summary>

```cadence
/// Contract defining cross-VM asset interfaces
access(all) contract CrossVMAsset {
    /// Enables a bridging entrypoint to EVM on an implementing Vault
    access(all) resource interface EVMBridgeableVault {
        access(all) fun bridgeToEVM(amount: UFix64, to: EVM.EVMAddress, tollFee: @FlowToken.Vault)
    }

    /// Enables a bridging entrypoint to EVM on an implementing Collection
    access(all) resource interface EVMBridgeableCollection {
        access(all) fun bridgeToEVM(id: UInt64, to: EVM.EVMAddress, tollFee: @FlowToken.Vault)
    }
}
```
</details>

<details>
<summary>FlowEVMBridgeFactory.sol</summary>

```solidity
// TODO - Contract factory & EVM inspector
```
</details>

<details>
<summary>IFlowBridgedAsset.sol</summary>

```solidity
// TODO - Identifies corresponding Flow contract address
```
</details>

<details>
<summary>FlowBridgedNFT.sol</summary>

```solidity
// TODO - Template for bridged Flow-native NFTs
```
</details>

<details>
<summary>FlowBridgedFT.sol</summary>

```solidity
// TODO - Template for bridged EVM-native FTs
```
</details>

### Considerations

#### NFT Metadata

Platform expectations between EVM & Flow NFT metadata storage differ. Whereas Flow projects generally prioritize onchain
metadata (with the exception of image data), EVM projects typically store NFT metadata in offchain systems - typically
in json blobs found in IPFS.

As the bridge is public infrastructure, there is a need to generalize the breadth of migrated metadata. Minimizing
metadata would mean:

- Looking solely at the Solidity contract, bridged Flow-native NFTs would have very little identifying information per
  the ERC721 standard - ID & perhaps an image pointer.
- Looking solely at the Cadence contract, bridged EVM-native NFTs would have very little available onchain metadata - ID
  & perhaps an IPFS URI.

For typical bridge infrastructure connecting separate zones of sovereignty, metadata migration could be handled by their
offchain system components. However, the requirement to atomically move assets between VMs prevents the inclusion of
such offchain systems as they would break atomicity and introduce the need for undesirable trust assumptions. For
example, upon bridging an NFT from Flow to EVM, an offchain listener would need to recognize a request to post metadata
to IPFS, post the metadata, and commit that URI to the defining EVM contract for the relevant NFT. We must trust that a/
the request is served, b/ metadata committed to IPFS, c/ commitment to the EVM contract succeeds and d/ that URI is
correct and contains correct metadata. Then there is the issue of IPFS storage funding.

Alternatively, if Flow projects want their metadata to be served well across VMs may take it on themselves to add
offchain IPFS metadata. This would allow the ecosystem to both maintain onchain metadata advantages as well as reach
parity with EVM platform expectations. We may then want to consider a Cadence metadata view specifically for IPFS-stored
metadata to support this use case.

Yet another alternative, it may be possible to expose an API matching that of IPFS services so that bridge-stored NFTs
metadata could be served to IPFS clients as they would request URI material from any other provider.

Lastly, there is also the option of creating an API similar to OpenSea's metadata API, which would allow for querying of
bridged NFT metadata from a centralized source. This would be a centralized solution, but would allow for a secondary
source of truth platforms could leverage to serve metadata to their users.

This problem and potential solutions are presented as a point of discussion and are not necessarily in scope for the
contract bridge's contract design.

#### NFT IDs

NFT ID values are some of the most critical token metadata, indentifying each token as unique. While Flow NFTs define
IDs as `UInt64`, ERC721 IDs are `uint256`. It remains an open question as to how this reduction from `UInt256` to
`UInt64` would affect the uniqueness of NFTs bridged from EVM to Flow and how the bridge should handle such conversions
while safeguarding ownership guarantees upon bridging back.

### Drawbacks

- All contracts deployed by the bridge have the same minimized functionality, restricted to their respective ecosystem
  standards
- Centralized storage of all bridged assets and their definitions presents a single, high-value target for potential
  hacks.
  - This vector is minimized by the fact that the bridge exists solely at the protocol and contract levels. Taking
    protocol security for granted, contract logic and key management are the primary considerations for compromise. This
    bridge benefits from the contained state space afforded by virtualizing the EVM environment in that offchain systems
    are not required for its function between VMs.

### Considered Alternatives

Motivated by the aforementioned drawback of centralized storage, previous iterations involved a network of distributed
accounts around a primary bridge account and contract. This primary contract served as the entrypoint for bridging
between VMs, deploying templated locking & asset-defining auxiliary contracts to distinct, contract-generated accounts.
The primary contract would conditionally deploy these auxiliary contracts on a per-asset basis as needed, maintaining a
registry for where each asset is locked/defined and routing bridge requests to their appropriate contracts.

The current approach was largely adapted from this design, but with auxiliary (locking & defining) contracts deployed to
the central bridge contract. Due to Cadence's contract namespace, it was also decided to dynamically name these
contracts using contract names derived from the relevant asset type. 

This design optimized for distributed asset storage and contract-mediated access. However, it also introduced additional
complexity and secondarily obscurity for what should be a highly transparent system. Additionally, since multisig access
is planned for the bridge and auxiliary accounts, centralization is ultimately no further improved by this design, at
least while custody is maintained on these accounts.

### Performance Implications

#### Migrations & Storage Usage

Previous network migrations have been complicated by single accounts using large amounts of storage. With a centralized
storage design, it's likely that (over time) the bridge account will consume a large amount of storage and that, given
the need to store bridged Flow-native assets indefinitely, that storage will likely only ever increase. Even if this
assumption is not true, it's to our benefit to consider and plan as if it is if account storage usage is a network-wide
issue.

Informed by initial conversations, saving state to a single account shouldn't be problematic until storage usage reaches
\>10GB of data which should give the team some time to figure out how to handle this edge case during migrations before
the problem is
encountered.

### Examples

At the current design stage, no working examples exist. However, examples will be a fast-follow as work on a proof of
concept continues. With that said, below are example transactions demonstrating the bridging of an NFT to and then from
EVM using the interface defined above.

<details>

<summary>Bridge Flow-Native NFT to EVM</summary>

```cadence
import "FungibleToken"
import "FlowToken"
import "NonFungibleToken"
import "ExampleNFT"

import "EVM"
import "FlowEVMBridge"
import "FlowEVMBridgeUtils"

transaction(id: UInt64) {
    
    let nft: &{NonFungibleToken.NFT}
    let nftType: Type
    let evmRecipient: &EVM.EVMAddress
    let evmContractAddress: EVM.EVMAddress
    let tollFee: @FlowToken.Vault
    
    prepare(signer: auth(BorrowValue) &Account) {
        // Withdraw the requested NFT
        let collection = signer.storage.borrow<auth(NonFungibleToken.Withdrawable) &{NonFungibleToken.Collection}>(
                from: ExampleNFT.CollectionStoragePath
            )!
        self.nft <- collection.withdraw(withdrawID: id)
        // Save the type for our post-assertion
        self.nftType = nft.getType()
        // Get the signer's COA EVMAddress as recipient
        self.evmRecipient = signer.storage.borrow<&EVM.BridgedAccount>(from: /storage/evm)!.address()
        // Pay the bridge toll
        self.tollFee <- signer.storage.borrow<auth(FungibleToken.Withdrawable) &FlowToken.Vault>(
                from: /storage/flowTokenVault
            )!.withdraw(amount: FlowEVMBridge.fee)
    }

    execute {
        // Execute the bridge
        FlowEVMBridge.bridgeNFTToEVM(token: <-self.nft, to: evmRecipient.address(), tollFee: <-self.tollFee)
    }

    // Post-assert bridge completed successfully on EVM side
    post {
        FlowEVMBridgeUtils.isOwnerOrApproved(
            ofNFT: id,
            owner: evmRecipient.address(),
            evmContractAddress: FlowEVMBridge.getAssetEVMContractAddress(forType: nftType)
        ): "Problem bridging to signer's COA!"
    }
}
```
</details>

<details>

<summary>Bridge Flow-Native NFT Back to Flow</summary>

```cadence
import "FungibleToken"
import "FlowToken"
import "NonFungibleToken"
import "ExampleNFT"

import "EVM"
import "FlowEVMBridge"
import "FlowEVMBridgeUtils"

transaction(id: UInt64) {
    
    let collection: &{NonFungibleToken.Collection}
    let coa: auth(Callable) &BridgedAccount
    let tollFee: @FlowToken.Vault
    
    prepare(signer: auth(BorrowValue) &Account) {
        // Borrow the NFT Collection
        self.collection = signer.storage.borrow<auth(NonFungibleToken.Withdrawable) &{NonFungibleToken.Collection}>(
                from: ExampleNFT.CollectionStoragePath
            )!
        // Pay the bridge toll
        self.tollFee <- signer.storage.borrow<auth(FungibleToken.Withdrawable) &FlowToken.Vault>(
                from: /storage/flowTokenVault
            )!.withdraw(amount: FlowEVMBridge.fee)
    }

    execute {
        // Get the bridge-deployed EVM contract address
        let evmContractAddress = FlowEVMBridge.getAssetEVMContractAddress(type: Type<@ExampleNFT.NFT>())
            ?? panic("No bridge-deployed EVM contract found for ExampleNFT")
        // Encode the approve call
        let calldata = EVM.encodeABIWithSignature(
            "approve(address,uint256)",
            [evmContractAddress, UInt256(id)]
        )
        // Execute the bridge & deposit
        let bridgedNFT <- FlowEVMBridge.bridgeNFTFromEVM(
            caller: self.coa,
            calldata: calldata,
            id: id,
            evmContractAddress: evmContractAddress,
            tollFee: <-self.tollFee
        )
        self.collection.deposit(token: <-bridgedNFT)
    }

    // Post-assert bridge completed successfully on EVM side
    post {
        self.collection.borrowNFT(id) != nil: "NFT was not bridged back to signer's collection"
    }
}
```
</details>

### Compatibility

This bridge design will support bridging fungible and non-fungible token assets between VMs, and doesn't account for
cases where the type instances overlap - i.e. semi-fungible tokens or multi-token contracts.

Of course, this bridge also dovetails with the ongoing virtualized EVM work, so is dependent on the existence of that
environment.

## Prior Art

While the work is happening concurrently, there may be some cross-pollination between this project and the [Axelar
Interchain Token Service](https://github.com/AnChainAI/anchain-axelar-dapper-flowbridge).

## Questions & Discussion Topics

- What does the interplay between risk vectors, tokenomics, and UX requirements imply for the fee amounts charged for
  bridging between VMs
  - Do we charge on a per instance or per locked storage unit basis?
- How will we handle either bridging or serving metadata for bridged Flow-native NFTs given the difference in metadata
  standards between Cadence & EVM?
- Is there an upper bound to how many contracts a single account can host?
- What if any issues with the NFT ID type differences - `UInt64` in Cadence & `uint256` in Solidity - present for
  potential overflow & collisions when moving between EVM -> Flow?
- Are there additional metadata mechanisms we should consider in the contract designs that would help platforms better
  represent bridged assets?