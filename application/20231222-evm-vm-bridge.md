---
status: implemented
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
- [Bridge Onboarding](#bridge-onboarding)
- [In Aggregate](#in-aggregate)
- [Implementation Details](#implementation-details)
  - [Contract Roles \& Concerns](#contract-roles--concerns)
  - [Case Studies](#case-studies)
  - [Interfaces](#interfaces)
  - [Special Cases](#special-cases)
  - [Considerations](#considerations)
  - [Drawbacks](#drawbacks)
  - [Considered Alternatives](#considered-alternatives)
  - [Performance Implications](#performance-implications)
  - [Examples](#examples)
  - [Compatibility](#compatibility)
- [Prior Art](#prior-art)
- [Source Code](#source-code)
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
their more detailed metadata which needs to be made to work for both VMs. With that said, many FTs on Flow define useful
onchain contract-level metadata, so we may also choose to bridge select metadata for FTs as well.

### Cadence to EVM

Breakdown of the steps required to bridge a token across VMs from Cadence to EVM. 

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
3. EVM transaction: mint corresponding NFT in EVM contract or transfer if the NFT already exists

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
* Bridge executes the protected `safeTransfer` call provided by COA A, transferring the ERC721 token to the bridge
  for escrow
* Bridge contract unlocks corresponding Cadence NFT from VM bridge contract escrow storage
* Returns NFT on call to bridge contract

#### Cadence transaction: COA B bought COA A's EVM NFT on FlowEVM, and then wants to bridge it back to Cadence

* COA B calls bridge contract to request bridging of EVM NFT back into Cadence
* Bridge contract calls into EVM to confirm COA B is the owner of the requested NFT. If so, process continues; otherwise
  reverts
* Bridge executes the protected `safeTransfer` call provided by COA A, transferring the ERC721 token to the bridge for
  escrow
* Contract unlocks corresponding Cadence NFT from VM bridge contract escrow storage
* Returns NFT on call to bridge contract

**Path 2: EVM originated NFT bridging to Cadence**

#### Prerequisite steps to user actions

* Defining corresponding Cadence contract must been deployed to the bridge. This is done via bridge contract templates
  where the deployed contract is "owned" by the bridge and actions on it orchestrated by contract logic.

#### Cadence transaction: COA A wants to bridge an EVM NFT to Cadence

* Ensure prerequisites 
* Within the Cadence call to bridge, a protected `safeTransfer` call is included, transferring the ERC721 from the
  caller to the bridge escrow
* Bridge checks that the named owner is the owner of the NFT on EVM side before executing approval to Bridge account COA
  address
* Bridge ensures transfer was successful
* Bridge validates its ownership of the NFT on EVM side after transfer, thus locking NFT to be bridged
* Bridge unlocks the Cadence NFT if escrowed, otherwise mints NFT 
from the bridge-owned NFT contract and returns it to the caller
* Collection is configured if necessary and deposited to COA A's account

# Design Proposal

## Context

It's helpful to understand that the entrypoint to FlowEVM is mediated by the [`CadenceOwnedAccount`
(COA)](https://github.com/onflow/flips/pull/225). This Cadence resource provides access to FlowEVM, enabling encoded
calls into EVM originating from the calling COA's EVM address. In short, it's a resource that also functions as an EVM
account whose access is controlled by resource ownership instead of an offchain signature.

COAs are the only cross-VM objects universally available from Cadence. This type provides the means to use Solidity's
identity-based approach for access and enables any address to receive assets; callers bridging *to* FlowEVM from Cadence
may transfer tokens to any EVM address. In the opposite direction, only COAs may initiate bridging *from* EVM since
there is not yet a mechanism to initiate Cadence state change from the EVM environment.

Hopefully, this context clarifies that bridging in either direction - Cadence -> EVM & EVM -> Cadence - is at all times
initiated via call to the bridge's Cadence contracts.

It's also interesting to note that any project could build an asset-specific bridge between VMs using the same
primitives and toolsets leveraged for the design below. However, the intention for this bridge is to provide public
infrastructure to seamlessly and permissionlessly move assets between VMs without requiring the support of the asset
developers while maintaining a high security bar for all users of the bridge.

## Overview

Given the variety of problem components and drivers of complexity - namely storage and the potential spam vector -
initial designs targeted a distributed approach. Based on feedback alleviating concerns around storage and the
complexity introduced by said distribution, this iteration aggregates all bridge-related contracts & resources in a
single account.

Updates to this implementation also include deeper integration with the CadenceOwnedAccount resource (COA), enabling
asset bridging via a simple interface available directly from a COA.

The central bridge contract will act as a secondary (to the COA interface) entry point for bridge requests,
orchestrating asset onboarding deployments & configuration, and minting/burning + lock/unlock as needed to fulfill
requests. Deployed contracts are “owned” by the bridge, but owner interactions are mediated by contract logic on either
end in addition to multi-sig patterns consistent with other core network infrastructure accounts.

On the EVM side, a central contract factory will instantiate ERC20 & ERC721 contracts as directed by calls from the
bridge’s COA. This factory will also implement a number of helper methods to give the bridge contracts visibility into the
EVM environment. These methods might include things like retrieving an asset’s interface conformance, determining if EVM
contracts are bridge-owned, validating asset ownership, etc. so the COA has a central trusted source of truth for
critical state assertions.

> :information_source: An important conceptual point worth noting is that bridge requests are only initiated via
> Cadence. Since EVM cannot initiate state change in the Cadence environment, bridging in either direction is always
> performed via calls on Cadence resources and contracts.

## Bridge Onboarding

Given some asset in either Cadence or EVM, there is necessarily some source contract that defines it. The environment in
which that source contract resides is the environment to which the asset is considered “native”.

In order for the bridge to fulfill a given request, it’s necessary for a defining contract to exist in the target VM, a
known association between source and target contracts to exist, and for the bridge to have some mechanism for either
escrow or minting/burning.

Due to how contract deployments function with relationship to state availability, this deployment and configuration must
be completed in a distinct transaction prior to fulfilling bridge requests. **The bridge cannot fulfill a request in the
same transaction in which it also deploys the target contract as the deployed contract will not exist in the state space
until after the transaction is complete.**

This configuration process will be termed “onboarding”, and an asset for which the bridge is configured to handle will
be considered “onboarded”. This action, unless a project has explicitly disabled bridging, can be executed by anyone as
its protected by contract logic. In virtually all cases, the bridge will deploy and own a contract in the non-native
environment enabling it to perform privileged functions such as minting and updating metadata, again protected by bridge
contract logic.

There is a small minority of assets that do not conform to the standard configuration and require special handling based
on business or protocol requirements. What follows addresses call flows for the vast majority of NFT bridge requests.
See [Special Cases](#special-cases) below for more information and context on such assets.

## In Aggregate

Diagrams depicting the call flows for both Flow- and EVM-native NFTs bridging according to the proposed design are
included later in this FLIP. Beside those pictured, auxiliary contracts will exist to support bridge operations such as
interfaces, utilities, configuration, etc., but the are not pictured below as they are not critical to high-level
bridging operations. Immediately below is a birds-eye view of the contract suite across VMs.

![FlowEVM VM Bridge Design Overview](20231222-evm-vm-bridge-resources/overview.png) *Bridge functionality is enabled
directly on the COA resource for simplicity, with call passing through the EVM contract to a routing resource to the
central VM bridge. The central bridge contract then handles bridging requests dependent on whether the asset is an NFT
or FT, whether the asset is Cadence- or EVM-native, and the direction of the request. Due to the need to special case
some assets, Handlers are built in to the bridge enabling custom passthroughs based on non-standard business logic. For
most all assets, at asset onboarding, the bridge performs contract initialization on either side of the VM by
dynamically deploying EVM or Cadence contracts and maintaining associations between source and target contract
definitions.*

## Implementation Details

### Contract Roles & Concerns

#### Flow

- **EVM Contract Integration (EVM & EVMBridgeRouter)**
  - Enable COA to bridge FT/NFTs between VMs
  - Route requests from COAs to VM bridge

- **Bridge**
  - Unified entry point for bridging assets between VMs
  - Unified query point for NFT escrow contracts & to assess Cadence x EVM contract associations
  - Owning (via contract COA) all deployed contracts on EVM side
  - Escrowing (via contract COA) all assets on EVM side when bridged to Flow
  - Initializing BridgedFT & BridgedNFT contracts defining EVM-native assets when first bridged to Cadence
- **FT/NFT Escrow**
  - Lock tokens bridging to EVM
  - Unlock tokens bridging from EVM
  - Serve query requests about locked tokens - e.g. resolving token metadata
- **Bridged FT/NFT**
  - Define EVM-native tokens bridged from EVM to Flow
  - Point to the corresponding EVM-native contract
  - Maintain up-to-date token metadata with source ERC20/721 as ground truth

#### EVM

> :information_source: Note: Self-defined locking functionality is not required as “locked” assets will simply be
> transferred to the FlowEVMBridge.COA.EVMAddress

- **FlowBridgeDeploymentRegistry**
  - Maintain a registry of all bridge-deployed EVM contracts
- **ERC20Deployer**
  - Deploys EVM-defining implementations of fungible tokens as ERC20 from template
- **ERC721Deployer**
  - Deploys EVM-defining implementations of non-fungible tokens as ERC721 from template

- **FlowBridgeFactory**
  - Deploys new instances of FlowBridgedFT/NFT via deployers
  - Passes through the named registry for knowledge of all bridge-deployed EVM contract addresses & their Cadence token
    identifier correspondence
  - Aids visibility of bridge contract into the EVM environment with assistive methods - isERC721, isERC20, etc.

- **FlowBridgedFT/NFT**
  - Define Cadence-native tokens bridged from Cadence to EVM
  - Point to the corresponding Cadence-native contract
  - Contain bridged token metadata updated at time of last bridging

### Case Studies

The task of bridging FTs & NFTs between VMs can be split into four distinct cases, each with their own unique path. An
asset can either be Cadence- or EVM-native, and it can either be bridged from Cadence to EVM or EVM to Cadence. The
following sections outline an NFT bridge path for each case.

> :information_source: A note about bridge "onboarding" - assets moving from one VM to another must at minimum have
> contracts defining them in their target VM. These contracts must be deployed in a transaction preceding the movement
> of the asset as deployed contracts are not available in the Cadence's state space until the deploying transaction has
> finalized. Thus, onboarding cannot be done on the fly and must be a discrete step of the bridging process.

#### Cadence-Native NFT Onboarding

1. Pre-flight checks: Assert the type is supported, hasn't opted out, requires onboarding, and fee provider can provide
   onboarding fee
2. Withdraw onboard fee from given fee Provider & deposit to bridge Vault
3. Does the bridge have a custom Handler for this type? a. If so, revert as a Handler implies custom configuration.
    b. Otherwise, continue
4. Is this an NFT or FT? - NFT, continue as NFT
5. Deploy ERC721 from factory, passing on identifying information from the type & its defining Cadence contract
    a. Factory retains deployed address & Cadence type identifier in mapping & reverse mapping
    b. Returns the deployed ERC721 address
6. Call to BridgeNFTEscrow to initialize escrow locker, passing on the type and deployed ERC721 address
7. Associate the type with the deployed ERC721 address

![Cadence-native NFT Onboarding](20231222-evm-vm-bridge-resources/cadence_native_nft_onboarding.png)

#### Cadence-Native: Cadence -> EVM

*Lock in Cadence & Mint/Transfer in EVM*

1. Pre-flight checks: Assert the type is supported, has been onboarded, and fee provider can provide onboarding fee
2. Get a URI for the NFT
    - Default to project-specified URI
    - If not defined, serialize the NFT as JSON data URL
3. Lock NFT in escrow
4. Calculate the bridge fee based on flat fee + storage usage
5. Withdraw onboard fee from given fee Provider & deposit to bridge Vault
6. Get the token's corresponding ERC721 contract address
5. Execute the bridge
    - Check if the ERC721 is factory-deployed - Cadence-native: return true
        - If the token exists, transfer and update the URI
        - Otherwise mint the token to the recipient with the URI
    - Otherwise transfer the token to the recipient


![Cadence-native Cadence to EVM](20231222-evm-vm-bridge-resources/cadence_native_nft_to_evm.png)

#### Cadence-Native: EVM -> Cadence

*Unlock in Cadence & Transfer in EVM*

1. Pre-flight checks: Assert the type is supported, has been onboarded, and fee provider can provide onboarding fee
2. Withdraw the bridge fee from the given Provider & deposit to bridge Vault
3. Get the requested type's associated EVM contract address
4. Assert the caller is the requested ERC721 token owner or approved
5. Execute the provided protected callback
6. Assert that now the bridge COA is the ERC721 token owner after executing the protected callback
7. Derive the requested type's contract address & name & attempt to borrow the contract as EVMBridgeMinter
8. Get the tokenURI from the ERC721 contract
9. If the corresponding Cadence NFT is in escrow, unlock the NFT
    - If the defining Cadence contract is EVMBridgeMinter, update the token URI <- false in this case
10. Return the unlocked NFT


![Cadence-native EVM to Cadence](20231222-evm-vm-bridge-resources/cadence_native_nft_from_evm.png)

#### EVM-Native NFT Onboarding

1. Pre-flight checks: Assert the contract address is supported ERC, hasn't opted out, requires onboarding, and fee provider can provide onboarding fee
2. Withdraw onboard fee from given fee Provider & deposit to bridge Vault
3. Does the bridge have a custom Handler for this EVM Address?
    a. If so, revert as a Handler implies custom configuration.
    b. Otherwise, continue
4. Is this an ERC20 or ERC721? - ERC721, continue as ERC721
5. Deploy BridgedNFT Cadence contract from template, passing on identifying information from the ERC721 contract & deriving Cadence contract name
6. Associate the deployed NFT type with the ERC721 address
7. Initialize escrow for the deployed NFT type

![EVM-native NFT Onboarding](20231222-evm-vm-bridge-resources/evm_native_nft_onboarding.png)

#### EVM-Native: EVM -> Cadence

*Mint/Unlock in Cadence & Transfer in EVM*

1. Pre-flight checks: Assert the type is supported, has been onboarded, and fee provider can provide onboarding fee
2. Withdraw the bridge fee from the given Provider & deposit to bridge Vault
3. Get the requested type's associated EVM contract address
4. Assert the caller is the requested ERC721 token owner or approved
5. Execute the provided protected callback
6. Assert that now the bridge COA is the ERC721 token owner after executing the protected callback
7. Derive the requested type's contract address & name & attempt to borrow the contract as EVMBridgeMinter
8. Get the tokenURI from the ERC721 contract
9. If the corresponding Cadence NFT is in escrow, unlock the NFT <- false in this case
10. Otherwise, mint a new NFT with the ERC721 URI & define the source contract's ERC721 ID and return

![Cadence-native Cadence to EVM](20231222-evm-vm-bridge-resources/evm_native_nft_from_evm.png)

#### EVM-Native: Flow -> EVM

*Lock in Flow & Transfer in EVM*

1. Assert the asset is either an NFT or FT & has been onboarded  (it will be by construction)
2. Determine if asset is FT or NFT
    - Check if resource is an instance of FT.Vault or NFT.NFT (excluding overlapping instances, at least for POC)
3. Determine if asset is Flow or EVM-native
    - Check if resource is defined in bridge-hosted NFT or NFT contract - if so, EVM-native, else Flow-native
4. Borrow the relevant BridgedNFT contract
5. Borrow the BridgedNFT contract from the NFT type identifier & passthrough bridge request
6. BridgedNFT burns the NFT
7. BridgedNFT calls to EVM contract, transferring NFT to defined recipient
8. BridgedNFT confirms recipient is owner of the NFT post-transfer

![Flow-native Flow to EVM](20231222-evm-vm-bridge-resources/evm_native_nft_to_evm.png)

### Interfaces

<details>

<summary>EVM.cdc</summary>

```cadence
access(all)
contract EVM {
    // The following encompasses updates proposed to the EVM contract to integrate the COA resource with the VM Bridge
    //
    access(all) entitlement Bridge

    access(all)
    resource CadenceOwnedAccount: Addressable {
        /* ... existing implementation ... */

        /* COA Additions */
        //
        /// Bridges the given NFT to the EVM environment, requiring a Provider from which to withdraw a fee to fulfill
        /// the bridge request
        access(all)
        fun depositNFT(
            nft: @{NonFungibleToken.NFT},
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        )
        /// Bridges the given NFT to the EVM environment, requiring a Provider from which to withdraw a fee to fulfill
        /// the bridge request
        access(Owner | Bridge)
        fun withdrawNFT(
            type: Type,
            id: UInt256,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        ): @{NonFungibleToken.NFT}
        /// Bridges the given NFT to the EVM environment, requiring a Provider from which to withdraw a fee to fulfill
        /// the bridge request
        access(all)
        fun depositTokens(
            from: @{FungibleToken.Vault},
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        )
        /// Bridges the given NFT to the EVM environment, requiring a Provider from which to withdraw a fee to fulfill
        /// the bridge request
        access(Owner | Bridge)
        fun withdrawTokens(
            type: Type,
            amount: UInt256,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        ): @{FungibleToken.Vault}
    }
    /// Returns a reference to the BridgeAccessor designated for internal bridge requests
    access(self)
    view fun borrowBridgeAccessor(): auth(Bridge) &{BridgeAccessor}

    /// Interface for a resource which acts as an entrypoint to the VM bridge. This is implemented in a bridge contract
    /// and Capability on it is saved in the BridgeRouter resource within the EVM account's storage.
    access(all)
    resource interface BridgeAccessor {
        /// Endpoint enabling the bridging of an NFT to EVM
        access(Bridge)
        fun depositNFT(
            nft: @{NonFungibleToken.NFT},
            to: EVM.EVMAddress,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        )
        /// Endpoint enabling the bridging of an NFT from EVM
        access(Bridge)
        fun withdrawNFT(
            caller: auth(Call) &CadenceOwnedAccount,
            type: Type,
            id: UInt256,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        ): @{NonFungibleToken.NFT}
        /// Endpoint enabling the bridging of an fungible tokens to EVM
        access(Bridge)
        fun depositTokens(
            from: @{FungibleToken.Vault},
            to: EVM.EVMAddress,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        )
        /// Endpoint enabling the bridging of an fungible tokens from EVM
        access(Bridge)
        fun withdrawTokens(
            caller: auth(Call) &CadenceOwnedAccount,
            type: Type,
            amount: UInt256,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        ): @{FungibleToken.Vault}
    }

    /// Interface which captures a Capability to the bridge Accessor, saving it within the BridgeRouter resource
    access(all)
    resource interface BridgeRouter {
        /// Returns a reference to the BridgeAccessor designated for internal bridge requests
        access(Bridge) view fun borrowBridgeAccessor(): auth(Bridge) &{BridgeAccessor}
        /// Sets the BridgeAccessor Capability in the BridgeRouter
        access(Bridge) fun setBridgeAccessor(_ accessor: Capability<auth(Bridge) &{BridgeAccessor}>) {
            pre {
                accessor.check(): "Invalid BridgeAccessor Capability provided"
                emit BridgeAccessorUpdated(
                    routerType: self.getType(),
                    routerUUID: self.uuid,
                    routerAddress: self.owner?.address ?? panic("Router must have an owner to be identified"),
                    accessorType: accessor.borrow()!.getType(),
                    accessorUUID: accessor.borrow()!.uuid,
                    accessorAddress: accessor.address
                )
            }
        }
    }
}
```
</details>

<details>

<summary>FlowEVMBridgeAccessor.cdc</summary>

```cadence
/// This contract defines a mechanism for routing bridge requests from the EVM contract to the Flow-EVM bridge contract
access(all)
contract FlowEVMBridgeAccessor {

    access(all) let StoragePath: StoragePath
    
    /// BridgeAccessor implementation used by the EVM contract to route bridge calls from COA resources
    access(all)
    resource BridgeAccessor : EVM.BridgeAccessor {
        /// Passes along the bridge request to dedicated bridge contract
        access(EVM.Bridge)
        fun depositNFT(
            nft: @{NonFungibleToken.NFT},
            to: EVM.EVMAddress,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        )
        /// Passes along the bridge request to the dedicated bridge contract, returning the bridged NFT
        access(EVM.Bridge)
        fun withdrawNFT(
            caller: auth(EVM.Call) &EVM.CadenceOwnedAccount,
            type: Type,
            id: UInt256,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        ): @{NonFungibleToken.NFT}
        /// Passes along the bridge request to dedicated bridge contract
        access(EVM.Bridge)
        fun depositTokens(
            vault: @{FungibleToken.Vault},
            to: EVM.EVMAddress,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        )
        /// Passes along the bridge request to the dedicated bridge contract, returning the bridged NFT
        access(EVM.Bridge)
        fun withdrawTokens(
            caller: auth(EVM.Call) &EVM.CadenceOwnedAccount,
            type: Type,
            amount: UInt256,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        ): @{FungibleToken.Vault}
        /// Returns a BridgeRouter resource in which to store a Capability on this BridgeAccessor
        access(EVM.Bridge) fun createBridgeRouter(): @BridgeRouter
    }

    /// BridgeRouter implementation used by the EVM contract to capture a BridgeAccessor Capability and route bridge
    /// calls from COA resources to the FlowEVMBridge contract
    access(all) resource BridgeRouter : EVM.BridgeRouter {
        /// Capability to the BridgeAccessor resource, initialized to nil
        access(self) var bridgeAccessorCap: Capability<auth(EVM.Bridge) &{EVM.BridgeAccessor}>?

        access(EVM.Bridge) fun setBridgeAccessorCap(_ cap: Capability<auth(EVM.Bridge) &{EVM.BridgeAccessor}>) {
            pre {
                cap.check(): "BridgeAccessor capability already set"
            }
        }
        /// Returns an EVM.Bridge entitled reference to the underlying BridgeAccessor resource
        access(EVM.Bridge) view fun borrowBridgeAccessor(): auth(EVM.Bridge) &{EVM.BridgeAccessor}
    }
}
```
</details>

<details>
<summary>IFlowEVMNFTBridge.cdc</summary>

```cadence
access(all) contract interface IFlowEVMNFTBridge {
    
    /* Events */
    //
    /// Broadcasts an NFT was bridged from Cadence to EVM
    access(all)
    event BridgedNFTToEVM(
        type: Type,
        id: UInt64,
        evmID: UInt256,
        to: String,
        evmContractAddress: String,
        bridgeAddress: Address
    )
    /// Broadcasts an NFT was bridged from EVM to Cadence
    access(all)
    event BridgedNFTFromEVM(
        type: Type,
        id: UInt64,
        evmID: UInt256,
        caller: String,
        evmContractAddress: String,
        bridgeAddress: Address
    )

    /* Getters */
    //
    /// Returns the EVM address associated with the provided type
    access(all) view fun getAssociatedEVMAddress(with type: Type): EVM.EVMAddress?
    /// Returns the EVM address of the bridge coordinating COA
    access(all) view fun getBridgeCOAEVMAddress(): EVM.EVMAddress

    /* Public Bridge Entrypoints */
    //
    /// Public entrypoint to bridge NFTs from Cadence to EVM.
    access(all) fun bridgeNFTToEVM(
        token: @{NonFungibleToken.NFT},
        to: EVM.EVMAddress,
        feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
    ) {
        pre {
            emit BridgedNFTToEVM(
                type: token.getType(),
                id: token.id,
                evmID: CrossVMNFT.getEVMID(from: &token as &{NonFungibleToken.NFT}) ?? UInt256(token.id),
                to: FlowEVMBridgeUtils.getEVMAddressAsHexString(address: to),
                evmContractAddress: FlowEVMBridgeUtils.getEVMAddressAsHexString(
                    address: self.getAssociatedEVMAddress(with: token.getType())
                        ?? panic("Could not find EVM Contract address associated with provided NFT")
                ), bridgeAddress: self.account.address
            )
        }
    }

    /// Public entrypoint to bridge NFTs from EVM to Cadence
    access(account) fun bridgeNFTFromEVM(
        owner: EVM.EVMAddress,
        type: Type,
        id: UInt256,
        feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider},
        protectedTransferCall: fun (): EVM.Result
    ): @{NonFungibleToken.NFT} {
        post {
            emit BridgedNFTFromEVM(
                type: result.getType(),
                id: result.id,
                evmID: id,
                caller: FlowEVMBridgeUtils.getEVMAddressAsHexString(address: owner),
                evmContractAddress: FlowEVMBridgeUtils.getEVMAddressAsHexString(
                    address: self.getAssociatedEVMAddress(with: result.getType())
                        ?? panic("Could not find EVM Contract address associated with provided NFT")
                ), bridgeAddress: self.account.address
            )
        }
    }
}
```
</details>

<details>
<summary>IFlowEVMTokenBridge.cdc</summary>

```cadence
access(all) contract interface IFlowEVMTokenBridge {
    
    /* Events */
    //
    /// Broadcasts fungible tokens were bridged from Cadence to EVM
    access(all) event BridgedTokensToEVM(
        type: Type,
        ufixAmount: UFix64,
        uintAmount: UInt256,
        to: String,
        evmContractAddress: String,
        bridgeAddress: Address
    )
    /// Broadcasts fungible tokens were bridged from EVM to Cadence
    access(all) event BridgedTokensFromEVM(
        type: Type,
        ufixAmount: UFix64,
        uintAmount: UInt256,
        caller: String,
        evmContractAddress: String,
        bridgeAddress: Address
    )

    /* Getters */
    //
    /// Returns the EVM address associated with the provided type
    access(all) view fun getAssociatedEVMAddress(with type: Type): EVM.EVMAddress?
    /// Returns the EVM address of the bridge coordinating COA
    access(all) view fun getBridgeCOAEVMAddress(): EVM.EVMAddress

    /* Public Bridge Entrypoints */
    //
    /// Public entrypoint to bridge fungibleTokens from Cadence to EVM.
    access(all) fun bridgeTokensToEVM(
        tokens: @{FungibleToken.Vault},
        to: EVM.EVMAddress,
        feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
    ) {
        pre {
            emit BridgedTokensToEVM(
                type: tokens.getType(),
                ufixAmount: tokens.balance,
                uintAmount: FlowEVMBridgeUtils.ufix64ToUInt256(value: tokens.balance, decimals: 8),
                to: FlowEVMBridgeUtils.getEVMAddressAsHexString(address: to),
                evmContractAddress: FlowEVMBridgeUtils.getEVMAddressAsHexString(
                    address: self.getAssociatedEVMAddress(with: token.getType())
                        ?? panic("Could not find EVM Contract address associated with provided NFT")
                ), bridgeAddress: self.account.address
            )
        }
    }

    /// Public entrypoint to bridge fungible tokens from EVM to Cadence
    access(account) fun bridgeTokensFromEVM(
        owner: EVM.EVMAddress,
        type: Type,
        amount: UInt256,
        feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider},
        protectedTransferCall: fun (): EVM.Result
    ): @{FungibleToken.Vault} {
        post {
            emit BridgedTokensFromEVM(
                type: result.getType(),
                ufixAmount: result.balance,
                uintAmount: amount,
                caller: FlowEVMBridgeUtils.getEVMAddressAsHexString(address: owner),
                evmContractAddress: FlowEVMBridgeUtils.getEVMAddressAsHexString(
                    address: self.getAssociatedEVMAddress(with: result.getType())
                        ?? panic("Could not find EVM Contract address associated with provided NFT")
                ), bridgeAddress: self.account.address
            )
        }
    }
}
```
</details>

<details>
<summary>FlowEVMBridge.cdc</summary>

```cadence
access(all) contract FlowEVMBridge : IFlowEVMNFTBridge, IFlowEVMTokenBridge {

    /* Events */
    //
    /// Emitted any time a new asset type is onboarded to the bridge
    access(all)
    event Onboarded(type: Type, cadenceContractAddress: Address, evmContractAddress: String)
    /// Denotes a defining contract was deployed to the bridge accountcode
    access(all)
    event BridgeDefiningContractDeployed(
        contractName: String,
        assetName: String,
        symbol: String,
        isERC721: Bool,
        evmContractAddress: String
    )

    /* Onboarding */
    //
    /// Onboards a given asset by type to the bridge. Since we're onboarding by Cadence Type, the asset must be defined
    /// in a third-party contract. Attempting to onboard a bridge-defined asset will result in an error as the asset has
    /// already been onboarded to the bridge.
    access(all) fun onboardByType(_ type: Type, feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}) {
        pre {
            feeProvider.isAvailableToWithdraw(amount: FlowEVMBridgeConfig.onboardFee):
                "Insufficient fee available via feeProvider"
            self.typeRequiresOnboarding(type) == true: "Onboarding is not needed for this type"
            FlowEVMBridgeUtils.isCadenceNative(type: type): "Only Cadence-native assets can be onboarded by Type"
        }
    }
    /// Onboards a given EVM contract to the bridge. Since we're onboarding by EVM Address, the asset must be defined in
    /// a third-party EVM contract. Attempting to onboard a bridge-defined asset will result in an error as onboarding
    /// is not required.
    access(all) fun onboardByEVMAddress(
        _ address: EVM.EVMAddress,
        feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
    ) {
        pre {
            feeProvider.isAvailableToWithdraw(amount: FlowEVMBridgeConfig.onboardFee):
                "Insufficient fee available via feeProvider"
        }
    }

    /* Public NFT Handling */
    //
    /// Public entrypoint to bridge NFTs from Cadence to EVM.
    ///
    /// @param token: The NFT to be bridged
    /// @param to: The NFT recipient in FlowEVM
    /// @param feeProvider: A reference to a FungibleToken Provider from which the bridging fee is withdrawn in $FLOW
    ///
    access(all)
    fun bridgeNFTToEVM(
        token: @{NonFungibleToken.NFT},
        to: EVM.EVMAddress,
        feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
    ) {
        pre {
            !token.isInstance(Type<@{FungibleToken.Vault}>()): "Mixed asset types are not yet supported"
            self.typeRequiresOnboarding(token.getType()) == false: "NFT must first be onboarded"
        }
    }

    /// Public entrypoint to bridge NFTs from EVM to Cadence
    ///
    /// @param owner: The EVM address of the NFT owner. Current ownership and successful transfer (via
    ///     `protectedTransferCall`) is validated before the bridge request is executed.
    /// @param type: The Cadence Type of the NFT to be bridged
    /// @param id: The NFT ID to bridged
    /// @param feeProvider: A reference to a FungibleToken Provider from which the bridging fee is withdrawn in $FLOW
    /// @param protectedTransferCall: A function that executes the transfer of the NFT from the named owner to the
    ///     bridge's COA. This function is expected to return a Result indicating the status of the transfer call.
    ///
    /// @returns The bridged NFT
    ///
    access(account)
    fun bridgeNFTFromEVM(
        owner: EVM.EVMAddress,
        type: Type,
        id: UInt256,
        feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider},
        protectedTransferCall: fun (): EVM.Result
    ): @{NonFungibleToken.NFT} {
        pre {
            feeProvider.isAvailableToWithdraw(amount: FlowEVMBridgeUtils.calculateBridgeFee(used: 0, includeBase: true)):
                "Insufficient fee paid"
            !type.isSubtype(of: Type<@{FungibleToken.Vault}>()): "Mixed asset types are not yet supported"
            self.typeRequiresOnboarding(type) == false: "NFT must first be onboarded"
        }

    /* Public FT Handling */
    //
    /// Public entrypoint to bridge NFTs from Flow to EVM - cross-account bridging supported
    ///
    /// @param vault: The FungibleToken Vault to be bridged
    /// @param to: The recipient of tokens in FlowEVM
    /// @param tollFee: The fee paid for bridging
    ///
    access(all) fun bridgeTokensToEVM(vault: @{FungibleToken.Vault}, to: EVM.EVMAddress, tollFee: @FlowToken.Vault) {
        pre {
            tollFee.balance == self.tollAmount: "Insufficient fee paid"
            vault.isInstance(of: Type<&{NonFungibleToken.NFT}>) == false: "Mixed asset types are not yet supported"
        }
        // Handle based on whether Flow- or EVM-native & passthrough to internal method
    }

    /// Public entrypoint to bridge fungible tokens from EVM to Flow
    ///
    /// @param caller: The caller executing the bridge - must be passed to check EVM state pre- & post-call in scope
    /// @param type: The Cadence Type of the tokens to be bridged
    /// @param amount: The amount of tokens to bridge
    /// @param feeProvider: A reference to a FungibleToken Provider from which the bridging fee is withdrawn in $FLOW
    /// @param protectedTransferCall: A function that executes the transfer of the tokens from the named owner to the
    ///     bridge's COA. This function is expected to return a Result indicating the status of the transfer call.
    ///
    access(account) fun bridgeTokensFromEVM(
        owner: EVM.EVMAddress,
        type: Type,
        amount: UInt256,
        feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider},
        protectedTransferCall: fun (): EVM.Result
    ): @{FungibleToken.Vault} {
        pre {
            feeProvider.isAvailableToWithdraw(amount: FlowEVMBridgeUtils.calculateBridgeFee(used: 0, includeBase: true)):
                "Insufficient fee paid"
            !type.isSubtype(of: Type<@{NonFungibleToken.NFT}>()): "Mixed asset types are not yet supported"
            self.typeRequiresOnboarding(type) == false: "NFT must first be onboarded"
        }
    }

    /* Public Getters */
    //
    /// Returns the EVM address associated with the provided type
    access(all) view fun getAssociatedEVMAddress(with type: Type): EVM.EVMAddress?
    /// Returns the EVM address of the bridge coordinating COA
    access(all) view fun getBridgeCOAEVMAddress(): EVM.EVMAddress
    /// Returns whether an asset needs to be onboarded to the bridge
    access(all) view fun typeRequiresOnboarding(_ type: Type): Bool?
    /// Returns whether an EVM-native asset needs to be onboarded to the bridge
    access(all) fun evmAddressRequiresOnboarding(_ address: EVM.EVMAddress): Bool?

    /* Internal Helpers */
    //
    /// Deploys templated EVM contract via Solidity Factory contract supporting bridging of a given asset type
    access(self)
    fun deployEVMContract(forAssetType: Type): EVM.EVMAddress
    /// Deploys templated ERC721 contract supporting EVM-native asset bridging to Cadence
    access(self) fun deployERC721(_ forNFTType: Type): EVM.EVMAddress
    /// Deploys templated ERC721 contract supporting EVM-native asset bridging to Cadence
    access(self) fun deployERC20(_ forFTType: Type): EVM.EVMAddress
    /// Helper for deploying templated defining contract supporting EVM-native asset bridging to Cadence
    /// Deploys either NFT or FT contract depending on the provided type
    access(self) fun deployDefiningContract(evmContractAddress: EVM.EVMAddress)
}
```

</details>

<details>
<summary>FlowEVMBridgeConfig.cdc</summary>

```cadence
/// This contract is used to store configuration information shared by FlowEVMBridge contracts. May also serve as
/// repository of other shared configuration-related values
///
access(all) contract FlowEVMBridgeConfig {

    /* Contract values */
    //
    /// Amount of FLOW paid to onboard a Type or EVMAddress to the bridge
    access(all) var onboardFee: UFix64
    /// Flat rate fee for all bridge requests
    access(all) var baseFee: UFix64
    /// Fee rate per storage unit consumed by bridged assets
    access(all) var storageRate: UFix64
    /// Mapping of Type to its associated EVMAddress as relevant to the bridge
    access(self) let typeToEVMAddress: {Type: EVM.EVMAddress}
    /// Mapping of Type to special-case Handlers for custom bridge handling
    access(self) let handlers: @{Type: {BridgeHandler.Handler}}
    /// Mapping of EVMAddress values to whether onboarding explicitly disabled for associated contracts
    access(self) let evmFilters: {EVM.EVMAddress: Bool}
    
    /* Path Constants */
    //
    /// StoragePath where bridge Cadence Owned Account is stored
    access(all) let coaStoragePath: StoragePath
    /// StoragePath where bridge config Admin is stored
    access(all) let adminStoragePath: StoragePath
    /// Cannonical StoragePath for the ScopedProvider capability in caller accounts
    access(all) let providerCapabilityStoragePath: StoragePath

    /* Events */
    //
    /// Emitted whenever the onboarding fee is updated
    access(all) event BridgeFeeUpdated(old: UFix64, new: UFix64, isOnboarding: Bool)
    /// Emitted whenever baseFee or storageRate is updated
    access(all) event StorageRateUpdated(old: UFix64, new: UFix64)

    /* Getters */
    //
    /// Retrieves the EVMAddress associated with a given Type if it has been onboarded to the bridge
    access(all)
    view fun getEVMAddressAssociated(with type: Type): EVM.EVMAddress? {
        return self.typeToEVMAddress[type]
    }

    /* Bridge Account Methods */
    //
    /// Enables bridge contracts to update the typeToEVMAddress mapping
    access(account)
    fun associateType(_ type: Type, with evmAddress: EVM.EVMAddress) {
        self.typeToEVMAddress[type] = evmAddress
    }
    
    /* Config Admin */
    //
    // Admin resource enabling updating fee amounts
    access(all) resource Admin {
        /// Updates the onboarding fee
        access(all) fun updateOnboardingFee(_ new: UFix64)
        /// Updates the base fee
        access(all) fun updateBaseFee(_ new: UFix64)
        /// Updates the storage rate
        access(all) fun updateStorageRate(_ new: UFix64)
    }
}
```

</details>

<details>
<summary>FlowEVMBridgeUtils.cdc</summary>

```cadence
/// Util contract serving all bridge contracts
access(all) contract FlowEVMBridgeUtils {

    /// Address of the bridge factory Solidity contract
    access(all) let bridgeFactoryEVMAddress: EVM.EVMAddress
    /// Delimeter used to derive contract names
    access(self) let delimiter: String
    /// Mapping containing contract name prefixes
    access(self) let contractNamePrefixes: {Type: {String: String}}

    /* Public Bridge Utils */
    //
    /// Returns an EVMAddress as a hex string without a 0x prefix
    access(all) view fun getEVMAddressAsHexString(address: EVM.EVMAddress): String
    /// Returns an EVMAddress as a hex string without a 0x prefix, truncating the string's last 20 bytes if exceeded
    access(all) fun getEVMAddressFromHexString(address: String): EVM.EVMAddress?
    /// Validates the Vault used to pay the bridging fee
    access(all) view fun calculateBridgeFee(used: UInt64, includeBase: Bool): UFix64
    /// Returns whether the given type is allowed to be bridged as defined by the BridgePermissions contract interface.
    /// If the type's defining contract does not implement BridgePermissions, the method returns true as the bridge
    /// operates permissionlessly by default. Otherwise, the result of {BridgePermissions}.allowsBridging() is returned
    access(all) view fun typeAllowsBridging(_ type: Type): Bool
    /// Returns whether the given address has opted out of enabling bridging for its defined assets. Reverts on EVM call
    /// failure.
    access(all) fun evmAddressAllowsBridging(_ address: EVM.EVMAddress): Bool
    /// Identifies if an asset is Cadence- or EVM-native, defined by whether a bridge contract defines it or not
    access(all) fun isCadenceNative(type: Type): Bool
    /// Identifies if an asset is Cadence- or EVM-native, defined by whether a bridge-owned contract defines it or not
    access(all) fun isEVMNative(evmContractAddress: EVM.EVMAddress): Bool
    /// Determines if the given EVM contract address was deployed by the bridge by querying the factory contract
    /// Reverts on EVM call failure.
    access(all) fun isEVMContractBridgeOwned(evmContractAddress: EVM.EVMAddress): Bool
    /// Identifies if an asset is ERC721 && not ERC20
    access(all) fun isERC721(evmContractAddress: EVM.EVMAddress): Bool
    /// Identifies if an asset is ERC20 and not ERC721
    access(all) fun isERC20(evmContractAddress: EVM.EVMAddress): Bool
    /// Returns whether the given type is either an NFT or FT exclusively
    access(all) view fun isValidFlowAsset(type: Type): Bool
    /// Determines if the owner is in fact the owner of the NFT at the ERC721 contract address
    access(all) fun isOwnerOrApproved(ofNFT: UInt64, owner: EVM.EVMAddress, evmContractAddress: EVM.EVMAddress): Bool
    /// Determines if the owner has sufficient funds to bridge the given amount at the ERC20 contract address
    access(all) fun hasSufficientBalance(amount: UFix64, owner: EVM.EVMAddress, evmContractAddress: EVM.EVMAddress): Bool


    /* EVM Call Wrappers */
    //
    /// Retrieves the NFT/FT name from the given EVM contract address - applies for both ERC20 & ERC721
    access(all) fun getName(evmContractAddress: EVM.EVMAddress): String
    /// Retrieves the NFT/FT symbol from the given EVM contract address - applies for both ERC20 & ERC721
    access(all) fun getSymbol(evmContractAddress: EVM.EVMAddress): String
    /// Retrieves the number of decimals for a given ERC20 contract address
    /// Retrieves the NFT/FT symbol from the given EVM contract address - applies for both ERC20 & ERC721
    access(all) fun getTokenURI(evmContractAddress: EVM.EVMAddress, id: UInt256): String
    /// Retrieves the contract URI from the given EVM contract address.
    access(all) fun getContractURI(evmContractAddress: EVM.EVMAddress): String?
    /// Retrieves the number of decimals for a given ERC20 contract address.
    access(all) fun getTokenDecimals(evmContractAddress: EVM.EVMAddress): UInt8

    /// Determines if the provided owner address is either the owner or approved for the NFT in the ERC721 contract
    access(all)
    fun isOwnerOrApproved(ofNFT: UInt256, owner: EVM.EVMAddress, evmContractAddress: EVM.EVMAddress): Bool
    /// Returns whether the given owner is the owner of the given NFT.
    access(all)
    fun isOwner(ofNFT: UInt256, owner: EVM.EVMAddress, evmContractAddress: EVM.EVMAddress): Bool
    /// Returns whether the given owner is approved for the given NFT.
    access(all)
    fun isApproved(ofNFT: UInt256, owner: EVM.EVMAddress, evmContractAddress: EVM.EVMAddress): Bool
    /// Determines if the owner has sufficient funds to bridge the given amount at the ERC20 contract address
    access(all)
    fun hasSufficientBalance(amount: UFix64, owner: EVM.EVMAddress, evmContractAddress: EVM.EVMAddress): Bool
    
    /* Derivation Utils */
    //
    /// Derives the Cadence contract name for a given Type
    access(all) fun deriveLockerContractName(fromType: Type): String?
    /// Derives the Cadence contract name for a given EVM asset
    access(all) fun deriveBridgedAssetContractName(fromEVMContract: EVM.EVMAddress): String?

    /* Math Utils */
    //
    /// Raises the base to the exponent
    access(all) view fun pow(base: UInt256, exponent: UInt8): UInt256
    /// Converts a UInt256 to a UFix64
    access(all) view fun uint256ToUFix64(value: UInt256, decimals: UInt8): UFix64
    /// Converts a UFix64 to a UInt256
    access(all) view fun ufix64ToUInt256(value: UFix64, decimals: UInt8): UInt256
    /// Returns the value as a UInt64 if it fits, otherwise panics
    access(all) view fun uint256ToUInt64(value: UInt256): UInt64

    /* Type-identifier Utils */
    //
    /// Returns the a type's defining contract address
    access(all) view fun getContractAddress(fromType: Type): Address?
    /// Returns the a type's defining contract name
    access(all) fun getContractName(fromType: Type): String?
    /// Returns the object's name from the given Type's identifier
    access(all) view fun getObjectName(fromType: Type): String?
    /// Splits a standard type identifier of format A.<CONTRACT_ADDRESS>.<CONTRACT_NAME>.<RESOURCE_NAME>
    access(all) view fun splitObjectIdentifier(identifier: String): [String]?
    // Builds a composite type from the given identifier parts
    access(all)
    view fun buildCompositeType(address: Address, contractName: String, resourceName: String): Type?
    
    /* Shared Bridge Helpers */
    //
    /// Deposits fees to the bridge account's FlowToken Vault - helps fund asset storage
    access(account) fun depositTollFee(_ tollFee: @FlowToken.Vault)
    /// Enables other bridge contracts to orchestrate bridge operations from contract-owned COA
    access(account) fun borrowCOA(): &EVM.BridgedAccount
    /// Shared helper simplifying calls using the bridge account's COA
    access(account) fun call(
        signature: String,
        targetEVMAddress: EVM.EVMAddress,
        args: [AnyStruct],
        gasLimit: UInt64,
        value: UFix64
    ): EVM.Result
}
```
</details>

</details>

<details>
<summary>FlowEVMBridgeTemplates.cdc</summary>

```cadence
/// This contract serves Cadence code from chunked templates, replacing the contract name with the name derived from
/// given arguments - either Cadence Type or EVM contract address.
access(all) contract FlowEVMBridgeTemplates {
    /// Canonical path for the Admin resource
    access(all) let AdminStoragePath: StoragePath
    /// Chunked Hex-encoded Cadence contract code, to be joined on derived contract name
    access(self) let templateCodeChunks: {String: [[UInt8]]}

    /// Emitted whenever there is a change to templated code
    access(all) event Updated(name: String, isNew: Bool?)

    access(all) fun getBridgedAssetContractCode(evmContractAddress: EVM.EVMAddress, isERC721: Bool): [UInt8]?
    
    /* Internal */
    //
    access(self) fun getBridgedNFTContractCode(contractName: String): [UInt8]?
    access(self) fun joinChunks(_ chunks: [[UInt8]], with name: String): [UInt8]

    access(self) fun getNFTLockerContractCode(forType: Type): [UInt8]?
    access(self) fun getFTLockerContractCode(forType: Type): [UInt8]?

    /// Resource enabling updates to the contract template code
    access(all) resource Admin {
        access(all) fun addNewContractCodeChunks(newTemplate: String, chunks: [String]) {
            pre {
                FlowEVMBridgeTemplates.templateCodeChunks[newTemplate] == nil: "Code already exists for template"
            }
        }
        access(all) fun upsertContractCodeChunks(forTemplate: String, chunks: [String])
        access(all) fun removeTemplate(name: String): Bool
    }
}
```
</details>

<details>
<summary>ICrossVM.cdc</summary>

```cadence
/// Contract interface denoting a cross-VM implementation, exposing methods to query EVM-associated addresses
access(all) contract interface ICrossVM {
    /// Retrieves the corresponding EVM contract address, assuming a 1:1 relationship between VM implementations
    access(all) view fun getEVMContractAddress(): EVM.EVMAddress
}
```
</details>

<details>
<summary>IEVMBridgeNFTMinter</summary>

```cadence

/// Contract interface enabling FlowEVMBridge to mint NFTs
access(all) contract interface IEVMBridgeNFTMinter {
    /// Account-only method to mint an NFT
    access(account) fun mintNFT(id: UInt256, tokenURI: String): @{NonFungibleToken.NFT}
    /// Allows the bridge to update the URI of bridged NFTs. This assumes that the EVM-defining project may contain
    /// logic (onchain or offchain) which updates NFT metadata in the source ERC721 contract. On bridging, the URI can
    /// then be updated in this contract to reflect the source ERC721 contract's metadata.
    access(account) fun updateTokenURI(evmID: UInt256, newURI: String)
}
```
</details>

<details>
<summary>IEVMBridgeFTMinter</summary>

```cadence
/// Contract interface enabling FlowEVMBridge to mint NFTs
access(all) contract interface IEVMBridgeFTMinter {
    /// Account-only method to mint an NFT
    access(account) fun mintFT(amount: UFix64): @{FungibleToken.Vault}
}
```
</details>

<details>
<summary>CrossVMNFT.cdc</summary>

```cadence
/// Contract defining cross-VM NFT & Collection interfaces
access(all) contract CrossVMNFT {
    
    /* Supporting Views - see https://github.com/onflow/flow-nft/pull/203 for feedback on inclusion to MetadataViews */
    //
    /// A struct to represent a general case URI, used to represent the URI of the NFT where the type of URI is not
    /// able to be determined (i.e. HTTP, IPFS, etc.)
    access(all) struct URI : MetadataViews.File {
        /// The base URI prefix, if any. Not needed for all URIs, but helpful for some use cases
        /// For example, updating a whole NFT collection's image host easily
        access(all) let baseURI: String?
        /// The URI value
        /// NOTE: this is set on init as a concatenation of the baseURI and the value if baseURI != nil
        access(self) let value: String

        access(all) view fun uri(): String {
            return self.value
        }
    }

    /// Proof of concept metadata to represent the ERC721 values of the NFT
    access(all) struct EVMBridgedMetadata {
        /// The name of the NFT
        access(all) let name: String
        /// The symbol of the NFT
        access(all) let symbol: String
        /// The URI of the NFT - this can either be contract-level or token-level URI depending on where the metadata
        /// is requested. See the ViewResolver contract interface to discover how contract & resource-level metadata
        /// requests are handled.
        access(all) let uri: {MetadataViews.File}
    }

    /// A simple interface for an NFT that is bridged from EVM. This may be necessary in some cases as there is
    /// discrepancy between Flow NFT standard IDs (UInt64) and EVM NFT standard IDs (UInt256). Discrepancies on IDs
    /// gone unaccounted for have the potential to induce loss of ownership bridging between VMs, so it's critical to
    /// retain identifying token information on bridging.
    ///
    /// See discussion https://github.com/onflow/flow-nft/pull/126#discussion_r1462612559 where @austinkline raised
    /// differentiating IDs in a minimal interface incorporated into the one below
    access(all) resource interface EVMNFT : NonFungibleToken.NFT {
        access(all) let evmID: UInt256
        access(all) let name: String
        access(all) let symbol: String
        access(all) view fun tokenURI(): String
        access(all) view fun getEVMContractAddress(): EVM.EVMAddress
    }

    /// A simple interface for a collection of EVMNFTs
    access(all) resource interface EVMNFTCollection {
        access(all) view fun getEVMIDs(): [UInt256]
        access(all) view fun getCadenceID(from evmID: UInt256): UInt64?
        access(all) view fun getEVMID(from cadenceID: UInt64): UInt256?
        access(all) view fun contractURI(): String?
    }

    /// Retrieves the EVM ID of an NFT if it implements the EVMNFT interface, returning nil if not
    access(all) view fun getEVMID(from token: &{NonFungibleToken.NFT}): UInt256?
}
```
</details>

<details>
<summary>CrossVMFungibleToken.cdc</summary>

```cadence
/// Contract defining cross-VM NFT & Collection interfaces
access(all) contract CrossVMFungibleToken {
    /// A simple interface for a Vault to retain identifying information from its EVM definition.
    access(all) resource interface EVMVault : FungibleToken.Vault {
        access(all) let name: String
        access(all) let symbol: String
        access(all) let decimals: UInt8
        access(all) view fun contractURI(): String?
        access(all) view fun getEVMContractAddress(): EVM.EVMAddress
    }
}
```
</details>

<details>
<summary>FlowEVMBridgeNFTEscrow.cdc</summary>

```cadence
/// This escrow contract handles the locking of assets that are bridged from Flow to EVM and retrieval of locked
/// assets in escrow when they are bridged back to Flow.
///
access(all) contract FlowEVMBridgeNFTEscrow {

    /* Getters */
    //
    /// Returns whether the Locker has been initialized for the given NFT type
    access(all) view fun isInitialized(forType: Type): Bool
    /// Returns whether an NFT with the given ID is locked
    access(all) view fun isLocked(type: Type, id: UInt64): Bool
    /// Retrieves the locked NFT's Cadence ID as defined in the NFT standard's NFT.id value if it is locked
    access(all) view fun getLockedCadenceID(type: Type, evmID: UInt256): UInt64?
    /// Returns the EVM NFT ID associated with the Cadence NFT ID. The goal is to retrieve the ERC721 ID value
    /// corresponding to the Cadence NFT.
    /// As far as the bridge is concerned, a bridge-deployed ERC721 assigns IDs based on NFT.id value at the time of
    /// bridging unless it implements the CrossVMNFT.EVMNFT in such case .evmID is used.
    /// Following this pattern, if locked, the NFT is checked for EVMNFT conformance returning .evmID,
    /// otherwise the NFT's ID is returned as a UInt256 as this is how the bridge would handle minting in the
    /// corresponding ERC721 contract.
    access(all) view fun getLockedEVMID(type: Type, cadenceID: UInt64): UInt256?
    /// Returns the views supported by the defined NFT type and ID
    access(all) view fun getViews(nftType: Type, id: UInt256): [Type]
    /// Resolves the requested view type for the given NFT type if it is locked and supports the requested view type
    access(all) fun resolveLockedNFTView(nftType: Type, id: UInt256, viewType: Type): AnyStruct?

    /* Bridge Methods */
    //
    /// Initializes the Locker for the given NFT type if it hasn't been initialized yet
    access(account) fun initializeEscrow(forType: Type, erc721Address: EVM.EVMAddress)
    /// Locks the NFT in escrow, returning the amount of storage used by the locker after storing
    access(account) fun lockNFT(_ nft: @{NonFungibleToken.NFT}): UInt64
    /// Unlocks the NFT of the given type and ID, reverting if it isn't in escrow
    access(account) fun unlockNFT(type: Type, id: UInt64): @{NonFungibleToken.NFT}
    /// Retrieves a reference to the NFT of the given type and ID if it is locked, otherwise returns nil
    access(account) view fun borrowLockedNFT(type: Type, id: UInt64): &{NonFungibleToken.NFT}?

    /* Locker */
    //
    /// The resource managing the locking & unlocking of NFTs via this contract's interface
    ///
    access(all) resource Locker : CrossVMNFT.EVMNFTCollection, NonFungibleToken.Collection {
        /// The type of NFTs this Locker escrows
        access(all) let lockedType: Type
        /// Corresponding ERC721 address for the locked NFTs
        access(all) let erc721Address: EVM.EVMAddress
        /// Count of locked NFTs as lockedNFTs.length may exceed computation limits
        access(self) var lockedNFTCount: Int
        /// Indexed on NFT UUID to prevent collisions
        access(self) let lockedNFTs: @{UInt64: {NonFungibleToken.NFT}}
        /// Maps EVM NFT ID to Flow NFT ID, covering cross-VM project NFTs
        access(self) let evmIDToFlowID: {UInt256: UInt64}

        /* CrossVMNFT.EVMNFTCollection conformance */
        //
        /// Returns all the EVM IDs of the locked NFTs if the locked token implements CrossVMNFT.EVMNFT
        access(all) view fun getEVMIDs(): [UInt256]
        /// Returns the Flow NFT ID associated with the EVM NFT ID if the locked token implements CrossVMNFT.EVMNFT
        access(all) view fun getCadenceID(from evmID: UInt256): UInt64?
        /// Returns the EVM NFT ID associated with the Cadence NFT ID. The goal is to retrieve the ERC721 ID value.
        /// As far as the bridge is concerned, an ERC721 defined by the bridge is the NFT's ID at the time of bridging
        /// or the value of the NFT.evmID if it implements the CrossVMNFT.EVMNFT interface when bridged.
        /// Following this pattern, if locked, the NFT is checked for EVMNFT conformance returning .evmID if so,
        /// otherwise the NFT's ID is returned as a UInt256 since that's how the bridge would handle minting in the
        /// corresponding ERC721 contract.
        access(all) view fun getEVMID(from cadenceID: UInt64): UInt256?
        access(all) view fun contractURI(): String?

        /* NFT.Collection conformance */
        //
        /// Returns the number of locked NFTs
        access(all) view fun getLength(): Int
        /// Depending on the number of locked NFTs, this may fail.
        access(all) view fun getIDs(): [UInt64]
        /// Returns a reference to the NFT if it is locked
        access(all) view fun borrowNFT(_ id: UInt64): &{NonFungibleToken.NFT}?
        /// Returns a map of supported NFT types - at the moment Lockers only support the lockedNFTType defined by
        /// their contract
        access(all) view fun getSupportedNFTTypes(): {Type: Bool}
        /// Returns true if the NFT type is supported
        access(all) view fun isSupportedNFTType(type: Type): Bool
        /// Returns the NFT as a Resolver if it is locked
        access(all) view fun borrowViewResolver(id: UInt64): &{ViewResolver.Resolver}?
        /// Deposits the NFT into this locker, noting its EVM ID if it implements CrossVMNFT.EVMNFT
        access(all) fun deposit(token: @{NonFungibleToken.NFT})
        /// Withdraws the NFT from this locker, removing it from the collection and returning it
        access(NonFungibleToken.Withdraw) fun withdraw(withdrawID: UInt64): @{NonFungibleToken.NFT}
        /// Creates an empty Collection - added here for NFT.Collection conformance
        access(all) fun createEmptyCollection(): @{NonFungibleToken.Collection}
    }
}

```
</details>

<details>
<summary>FlowEVMBridgeFTEscrow.cdc</summary>

```cadence
/// This escrow contract handles the locking of assets that are bridged from Flow to EVM and retrieval of locked
/// assets in escrow when they are bridged back to Flow.
access(all) contract interface FlowEVMBridgeFTEscrow {

    /* Getters */
    //
    access(all) view fun getLockedBalance(forVault: Type): UFix64
    access(all) view fun getViews(forVault: Type): [Type]
    access(all) fun resolveView(forVault: Type, viewType: Type): AnyStruct?

    /* Bridge Methods */
    //
    /// Initializes the Locker for the given Vault type if it hasn't been initialized yet
    access(account) fun initializeEscrow(forType: Type, erc20Address: EVM.EVMAddress)
    /// Locks the tokens in escrow, returning the amount of storage used by the locker after storing
    access(account) fun lockVault(_ vault: @{FungibleToken.Vault}): UInt64
    /// Unlocks the NFT of the given type and ID, reverting if it isn't in escrow
    access(account) fun unlockTokens(type: Type, amount: UFix64): @{FungibleToken.Vault}

    /* Locker */
    //
    /// The resource managing the locking & unlocking of FTs via this contract's interface
    ///
    access(all) resource Locker : CrossVMFT.EVMFT, FungibleToken.Vault {
        /// The type of FT this Locker escrows
        access(all) let lockedType: Type
        /// Corresponding ERC721 address for the locked NFTs
        access(all) let erc20Address: EVM.EVMAddress
        // Vault to hold all relevant locked FTs
        access(self) let lockedVault: @{FungibleToken.Vault}
        /// Field that tracks the balance of a vault
        access(all) var balance: UFix64

        // Returns the relevant fungible token contract
        access(all) view fun borrowContract(): &FungibleToken
        /// Returns the number of locked tokens in this locker
        access(all) view fun getBalance(): UFix64
        /// Returns a map of supported FT types - at the moment Lockers only support the lockedNFTType defined by
        /// their contract
        ///
        access(all) view fun getSupportedFTTypes(): {Type: Bool}
        /// Returns true if the NFT type is supported
        access(all) view fun isSupportedFTType(type: Type): Bool
        /// Deposits the given FT vault into this locker
        access(all) fun deposit(from: @{FungibleToken.Vault})
        access(all) view fun getSupportedVaultTypes(): {Type: Bool}
        /// Withdraws an amount of FT from this locker, removing it from the vault and returning it
        access(FungibleToken.Withdraw) fun withdraw(amount: UFix64): @{FungibleToken.Vault}
        /// createEmptyVault allows any user to create a new Vault that has a zero balance
        access(all) fun createEmptyVault(): @{Vault}
        access(all) view fun getViews(): [Type]
        access(all) fun resolveView(_ view: Type): AnyStruct?
    }
}
```
</details>

<details>
<summary>BridgePermissions</summary>

```cadence
/// This contract defines a simple interface which can be implemented by any resource to prevent it from being
/// onboarded to the Flow-EVM bridge
///
/// NOTE: This is suggested only for cases where your asset (NFT/FT) incorporates non-standard logic that would
///      break your project if not handles properly
///      e.g. assets are reclaimed after a certain period of time, NFTs share IDs, etc.
///
access(all) contract interface BridgePermissions {
    /// Contract-level method enabling implementing contracts to identify whether they allow bridging for their
    /// project's assets. Implementers may consider adding a hook which would later enable an update to this value
    /// should either the project be updated or the bridge be updated to handle the asset's non-standard logic which 
    /// would otherwise prevent them from supporting VM bridging at the outset.
    access(all) view fun allowsBridging(): Bool
}

```
</details>

<details>
<summary>FlowEVMBridgeFactory.sol</summary>

```solidity
// Factory contract enabling contract deployment, Cadence to EVM associations, and EVM inspection
contract FlowBridgeFactory is Ownable {
    // Address of the deployment registry where deployed contract associations are registered. Note that this is a
    // registry for EVM contracts deployed by the bridge factory and does not include those EVM-native contracts that
    // have been onboarded to the bridge via Cadence contracts. The global source of truth is found in the Cadence
    // side of the bridge, however this registry and publicly accessible methods can serve as a source of truth
    // within EVM. Given some EVM contract, its bridge-supported Cadence type association can be found (and vice-versa)
    // by querying this contract, thus preventing impersonation attacks.
    address private deploymentRegistry;
    // Mapping of deployer tags to their implementation addresses
    mapping(string => address) private deployers;

    // Emitted when a deployer is added to the factory
     */
    event DeployerAdded(string tag, address deployerAddress);
    // Emitted when a deployer is updated in the factory
     */
    event DeployerUpdated(string tag, address oldAddress, address newAddress);
    // Emitted when a deployer is removed from the factory
     */
    event DeployerRemoved(string tag, address oldAddress);
    // Emitted when the deployment registry is updated
     */
    event DeploymentRegistryUpdated(address oldAddress, address newAddress);

    constructor() Ownable(msg.sender) {}

    // Deploys a new asset contract via a registered deployer
    function deploy(
        string memory deployerTag,
        string memory name,
        string memory symbol,
        string memory cadenceAddress,
        string memory cadenceIdentifier,
        string memory contractURI
    ) public onlyOwner returns (address)
    // Retrieves the Cadence type identifier associated with the bridge-deployed contract
    function getCadenceIdentifier(address contractAddr) public view returns (string memory)
    // Retrieves the address of a bridge-deployed contract by its associated Cadence type identifier
    function getContractAddress(string memory cadenceIdentifier) public view returns (address)
    // Checks if a contract address is associated with a registered deployment
    function isBridgeDeployed(address contractAddr) public view returns (bool)
    // Makes a best guess if the contract address is an ERC20 token by calling the publicly accessible ERC20
    function isERC20(address contractAddr) public view returns (bool)
    // Determines if a contract is an ERC721 token by checking if it implements the ERC721 interface via ERC165
    function isERC721(address contractAddr) public view returns (bool)
    // Determines if a contract is a valid asset by checking if it is either an ERC20 or ERC721 implementation
    function isValidAsset(address contractAddr) public view returns (bool)
    // Retrieves the address of the deployment registry
    function getRegistry() public view returns (address)
    // Retrieves the address of a deployer by its tag
    function getDeployer(string memory tag) public view returns (address)
    // Sets the address of the deployment registry
    function setDeploymentRegistry(address _deploymentRegistry) public onlyOwner
    // Adds a new deployer to the factory
    function addDeployer(string memory tag, address deployerAddress) public onlyOwner
    // Adds a deployer to the factory, or updates the address of an existing deployer
    function upsertDeployer(string memory tag, address deployerAddress) public onlyOwner
    // Removes a deployer from the factory
    function removeDeployer(string memory tag) public onlyOwner
    // Registers a new deployment in the deployment registry
    function _registerDeployment(string memory cadenceIdentifier, address contractAddr) internal
    // Asserts that the registry address is non-zero and implements the IFlowEVMDeploymentRegistry interface
    function _requireIsValidRegistry(address registryAddr) internal view
    // Asserts that the contract address is non-zero and implements the IFlowEVMBridgeDeployer interface
    function _requireIsValidDeployer(address contractAddr) internal view
    // Checks if a contract implements a specific interface
    function _implementsInterface(address contractAddr, bytes4 interfaceId) internal view returns (bool)
    // Asserts that the address is non-zero
    function _requireNotZeroAddress(address addr) internal pure
}
```
</details>


<details>
<summary>FlowBridgeDeploymentRegistry.sol</summary>

```solidity
// Store associations between bridged Flow EVM contracts and a corresponding Cadence resource type.
contract FlowBridgeDeploymentRegistry is FlowEVMDeploymentRegistry, Ownable {
    constructor() Ownable(msg.sender)

    // The address of the registrar who can register new deployments
    address public registrar;
    // Association between Cadence type identifiers and deployed contract addresses
    mapping(string => address) private cadenceIdentifierToContract;
    // Reverse association between deployed contract addresses and Cadence type identifiers
    mapping(address => string) private contractToCadenceIdentifier;

    modifier onlyRegistrar()

    // ERC165 introspection
    function supportsInterface(bytes4 interfaceId) public view override(IERC165, ERC165) returns (bool)
    // Set the registrar address as the entity that can register new deployments. Only the owner can call this
    function setRegistrar(address _registrar) external onlyOwner
    // Get the Cadence type identifier associated with a contract address
    function getCadenceIdentifier(address contractAddr) external view returns (string memory)
    // Get the contract address associated with a Cadence type identifier
    function getContractAddress(string memory cadenceIdentifier) external view returns (address)
    // Check if a contract address is a registered deployment
    function isRegisteredDeployment(string memory cadenceIdentifier) external view returns (bool)
    // Check if a Cadence type identifier is associated with a registered deployment
    function isRegisteredDeployment(address contractAddr) external view returns (bool)
    // Register a new deployment address with the given Cadence type identifier. Can only be called by the
    function registerDeployment(string memory cadenceIdentifier, address contractAddr) external onlyRegistrar
    // Internal function to register a new deployment address with the given Cadence type identifier
    function _registerDeployment(string memory cadenceIdentifier, address contractAddr) internal
}

```
</details>


<details>

<summary>FlowBridgeERC20Deployer.sol</summary>

```solidity
// Deploys FlowEVMBridgedERC20 contracts with named associations to Cadence resources types. Only the
// delegated deployer can deploy new contracts. This contract is used by the Flow EVM bridge to deploy and define
// bridged ERC20 tokens which are defined natively in Cadence.
contract FlowEVMBridgedERC20Deployer is IFlowEVMBridgeDeployer, ERC165, Ownable {
    // The address of the delegated deployer who can deploy new contracts
    address public delegatedDeployer;

    event ERC20Deployed(
        address contractAddress, string name, string symbol, string cadenceTokenAddress, string cadenceVaultIdentifier
    );

    constructor() Ownable(msg.sender) {}

    modifier onlyDelegatedDeployer()

    // ERC165 introspection
    function supportsInterface(bytes4 interfaceId) public view override(IERC165, ERC165) returns (bool)
    // Deploy a new FlowEVMBridgedERC20 contract with the given name, symbol, and association to a Cadence
    function deploy(
        string memory name,
        string memory symbol,
        string memory cadenceAddress,
        string memory cadenceIdentifier,
        string memory contractURI
    ) external onlyDelegatedDeployer returns (address)
    // Set the delegated deployer address as the entity that can deploy new contracts. Only the owner can call this
    function setDelegatedDeployer(address _delegatedDeployer) external onlyOwner
}
```
</details>

<summary>FlowBridgeERC721Deployer.sol</summary>

```solidity
// Deploys FlowEVMBridgedERC721 contracts with named associations to Cadence resources types. Only the
// delegated deployer can deploy new contracts. This contract is used by the Flow EVM bridge to deploy and define
// bridged ERC721 tokens which are defined natively in Cadence.
contract FlowEVMBridgedERC721Deployer is IFlowEVMBridgeDeployer, ERC165, Ownable {
    // The address of the delegated deployer who can deploy new contracts
    address public delegatedDeployer;

    event ERC721Deployed(
        address contractAddress, string name, string symbol, string cadenceTokenAddress, string cadenceVaultIdentifier
    );

    constructor() Ownable(msg.sender) {}

    modifier onlyDelegatedDeployer()

    // ERC165 introspection
    function supportsInterface(bytes4 interfaceId) public view override(IERC165, ERC165) returns (bool)
    // Deploy a new FlowEVMBridgedERC721 contract with the given name, symbol, and association to a Cadence
    function deploy(
        string memory name,
        string memory symbol,
        string memory cadenceAddress,
        string memory cadenceIdentifier,
        string memory contractURI
    ) external onlyDelegatedDeployer returns (address)
    // Set the delegated deployer address as the entity that can deploy new contracts. Only the owner can call this
    function setDelegatedDeployer(address _delegatedDeployer) external onlyOwner
}
```
</details>

<details>
<summary>FlowEVMBridgedERC721.sol</summary>

```solidity
contract FlowBridgedERC721 is ERC721, ERC721URIStorage, ERC721Burnable, ERC721Enumerable, Ownable {
    string public flowNFTAddress;
    string public flowNFTIdentifier;

    constructor(
        address owner,
        string memory name,
        string memory symbol,
        string memory _flowNFTAddress,
        string memory _flowNFTIdentifier
    ) ERC721(name, symbol) Ownable(owner)

    // Enables minting of of Flow-native NFTs
    function safeMint(address to, uint256 tokenId, string memory uri) public onlyOwner
    // Retrieves the URI for the given NFT
    function tokenURI(uint256 tokenId) public view override(ERC721, ERC721URIStorage) returns (string memory)
    // ERC165 conformance & pass-through
    function supportsInterface(bytes4 interfaceId) public view override(ERC721, ERC721Enumerable, ERC721URIStorage) returns (bool)
    // Returns the Flow type identifier associated with this bridge-deployed contract
    function getFlowNFTAddress() public view returns (string memory)
    // Returns the Flow type identifier associated with this bridge-deployed contract
    function getFlowNFTIdentifier() public view returns (string memory)
    // Returns whether the given ID exists or not - deprecated by OZ, so added here for bridge validation
    function exists(uint256 tokenId) public view returns (bool)
    // Updates the URI for the given NFT which is done on bridging metadata from Cadence to EVM
    function updateTokenURI(uint256 tokenId, string memory uri) public onlyOwner
    // Getter for the contract's URI metadata
    function contractURI() public view returns (string memory)
}
```
</details>

<details>
<summary>FlowEVMBridgedERC20.sol</summary>

```solidity
contract FlowEVMBridgedERC20 is ERC20, ERC20Burnable, Ownable, ERC20Permit {
    string public flowTokenAddress;
    string public flowTokenIdentifier;
    string public contractMetadata;

    constructor(
        address owner,
        string memory name,
        string memory symbol,
        string memory _flowTokenAddress,
        string memory _flowTokenIdentifier,
        string memory _contractMetadata
    ) ERC20(name, symbol) Ownable(owner) ERC20Permit(name)

    // Enables the bridge to mint new tokens
    function mint(address to, uint256 amount) public onlyOwner
    // Returns the Flow type identifier associated with this bridge-deployed contract
    function getFlowTokenAddress() public view returns (string memory)
    // Returns the Flow type identifier associated with this bridge-deployed contract
    function getFlowTokenIdentifier() public view returns (string memory)
    // Returns the contract's metadata URI
    function contractURI() public view returns (string memory)
}
```
</details>

<details>

<summary>BridgePermissions.sol</summary>

```solidity
// Contract for which implementation is checked by the Flow VM bridge as an opt-out mechanism
// for non-standard asset contracts that wish to opt-out of bridging between Cadence & EVM. By
// default, the VM bridge operates on a permissionless basis, meaning anyone can request an asset
// be onboarded. However, some small subset of non-standard projects may wish to opt-out of this
// and this contract provides a way to do so while also enabling future opt-in.
//
// Note: The Flow VM bridge checks for permissions at asset onboarding. If your asset has already
// been onboarded, setting `permissions` to `false` will not affect movement between VMs.
//
abstract contract BridgePermissions is ERC165, IBridgePermissions {
    // The permissions for the contract to allow or disallow bridging of its assets.
    bool private _permissions;

    constructor() { _permissions = false; }

    // See {IERC165-supportsInterface}.
    function supportsInterface(bytes4 interfaceId) public view virtual override(ERC165, IERC165) returns (bool)
    // Returns true if the contract allows bridging of its assets. Checked by the Flow VM
    // bridge at asset onboarding to enable non-standard asset contracts to opt-out of bridging
    // between Cadence & EVM. Implementing this contract opts out by default but can be
    // overridden to opt-in or used in conjunction with a switch to enable opting in.
    function allowsBridging() external view virtual returns (bool)
    // Set the permissions for the contract to allow or disallow bridging of its assets.
    function _setPermissions(bool permissions) internal
}

```
</details>

### Special Cases

There are certain types of assets which may require special cased handling according to business demands or because the
asset itself, while implemented according to common standards, plays a non-standard role in either ecosystem. One such
asset falling into this category is `FlowToken`, though there may be others in the future. Here, we’ll discuss how these
two cases are handled by the bridge.

#### FlowToken

**Context**

Since $FLOW is the native token in both VMs, moving $FLOW between Cadence and EVM is integrated directly into the COA
resource and can be done without dependency on the VM bridge.

Bridging the token between VMs can be done easily via the EVM Cadence contract interface and its constructs. Depositing
\$FLOW is integrated directly into the `EVMAddress` object and can be deposited directly to any address in EVM without
the need to involve a COA resource at all. Bridging $FLOW from EVM can be done via any COA, resulting in a vault that
can be handled like any other Cadence Vault resource.

**Bridge Configuration**

Given the protocol-native integration of \$FLOW, the bridge will then special case on receipt of a request to bridge
\$FLOW to/from either VM and handle each direction accordingly. This can be done simply by filtering on the `Type` of
the requested asset.

On observing a request to bridge $FLOW to EVM, the bridge will simply leverage the existing `EVMAddress.deposit`
interface to deposit the provided Vault to the named recipient. However, since bridging $FLOW from EVM is built into the
COA resource and must be called from the owning COA - a call which is fundamentally different than that required to
transfer ERC20 to escrow - the same passthrough functionality cannot be leveraged. Due to these factors, requests to
bridge $FLOW from EVM will be filtered and reverted.

#### Generalized Cases

> :information_source: **TL;DR:** Requests to bridge special cased assets in either direction will be filtered,
> identified, and passed through to a custom handler. This handler will be initialized to disable bridging until
> explicitly enabled, at which point an association between the defining Cadence & EVM contracts will be simultaneously
> established. Once active, this handler will be routed related bridge requests, performing case-specific logic as
> needed to fulfill requests via the typical bridge entrypoints.

Building in special-cased functionality will aid code extensibility as the resources handling these requests can define
logic as needed on a case-by-case basis while maintaining a unified entry point for all bridge requests. As a
generalized solution for handling special cases, the bridge will leverage a `Handler` resource interface and maintain a
mapping of Cadence Types to Handler resource implementations.

<details>
<summary>Handler interface</summary>

```cadence
/// Handler for bridging Cadence native fungible tokens to EVM. In the event a Cadence project migrates native
/// support to EVM, this Hander can be configured to facilitate bridging the Cadence tokens to EVM. This Handler
/// then effectively allows the bridge to treat such tokens as bridge-defined on the Cadence side and EVM-native on
/// the EVM side minting/burning in Cadence and escrowing in EVM.
/// In order for this to occur, neither the Cadence token nor the EVM contract can be onboarded to the bridge - in
/// essence, neither side of the asset can be onboarded to the bridge.
/// The Handler must be configured in the bridge via the HandlerConfigurator. Once added, the bridge will filter
/// requests to bridge the token Vault to EVM through this Handler which cannot be enabled until a target EVM
/// address is set. Once the corresponding EVM contract address is known, it can be set and the Handler. It's also
/// suggested that the Handler only be enabled once sufficient liquidity has been arranged in bridge escrow on the
/// EVM side.
access(all) resource CadenceNativeTokenHandler : FlowEVMBridgeHandlerInterfaces.TokenHandler {
    /// Flag determining if request handling is enabled
    access(self) var enabled: Bool
    /// The Cadence Type this handler fulfills requests for
    access(self) var targetType: Type
    /// The EVM contract address this handler fulfills requests for. This field is optional in the event the EVM
    /// contract address is not yet known but the Cadence type must still be filtered via Handler to prevent the
    /// type from being onboarded otherwise.
    access(self) var targetEVMAddress: EVM.EVMAddress?
    /// The expected minter type for minting tokens on fulfillment
    access(self) let expectedMinterType: Type
    /// The Minter enabling minting of Cadence tokens on fulfillment from EVM
    access(self) var minter: @{FlowEVMBridgeHandlerInterfaces.TokenMinter}?

    /* --- HandlerInfo --- */

    /// Returns the enabled status of the handler
    access(all) view fun isEnabled(): Bool
    /// Returns the type of the asset the handler is configured to handle
    access(all) view fun getTargetType(): Type?
    /// Returns the EVM contract address the handler is configured to handle
    access(all) view fun getTargetEVMAddress(): EVM.EVMAddress?
    /// Returns the expected minter type for the handler
    access(all) view fun getExpectedMinterType(): Type?

    /* --- TokenHandler --- */

    /// Fulfill a request to bridge tokens from Cadence to EVM, burning the provided Vault and transferring from
    /// EVM escrow to the named recipient. Assumes any fees are handled by the caller within the bridge contracts
    ///
    /// @param tokens: The Vault containing the tokens to bridge
    /// @param to: The EVM address to transfer the tokens to
    ///
    access(account)
    fun fulfillTokensToEVM(
        tokens: @{FungibleToken.Vault},
        to: EVM.EVMAddress
    ) {
        pre {
            self.isEnabled(): "Handler is not yet enabled"
            tokens.getType() == self.getTargetType(): "Invalid Vault type"
        }
    }
    /// Fulfill a request to bridge tokens from EVM to Cadence, minting the provided amount of tokens in Cadence
    /// and transferring from the named owner to bridge escrow in EVM.
    ///
    /// @param owner: The EVM address of the owner of the tokens. Should also be the caller executing the protected
    ///              transfer call.
    /// @param type: The type of the asset being bridged
    /// @param amount: The amount of tokens to bridge
    ///
    /// @return The minted Vault containing the the requested amount of Cadence tokens
    ///
    access(account)
    fun fulfillTokensFromEVM(
        owner: EVM.EVMAddress,
        type: Type,
        amount: UInt256,
        protectedTransferCall: fun (): EVM.Result
    ): @{FungibleToken.Vault} {
        pre {
            self.isEnabled(): "Handler is not yet enabled"
        }
        post {
            result.getType() == self.getTargetType(): "Invalid Vault type returned"
        }
    }

    /* --- Admin --- */

    /// Sets the target type for the handler
    access(FlowEVMBridgeHandlerInterfaces.Admin)
    fun setTargetType(_ type: Type) {
        pre {
            self.getTargetType() == nil: "Target Type has already been set"
        }
        post {
            self.getTargetType()! == type: "Problem setting target type"
        }
    }
    /// Sets the target EVM address for the handler
    access(FlowEVMBridgeHandlerInterfaces.Admin)
    fun setTargetEVMAddress(_ address: EVM.EVMAddress) {
        pre {
            self.getTargetEVMAddress() == nil: "Target EVM address has already been set"
        }
        post {
            self.getTargetEVMAddress()!.bytes == address!.bytes: "Problem setting target EVM address"
        }
    }
    /// Sets the target type for the handler
    access(FlowEVMBridgeHandlerInterfaces.Admin)
    fun setMinter(_ minter: @{FlowEVMBridgeHandlerInterfaces.TokenMinter}) {
        pre {
            self.getExpectedMinterType() == minter.getType(): "Minter is not of the expected type"
            self.minter == nil: "Minter has already been set"
        }
    }
    /// Enables the handler for request handling. The
    access(FlowEVMBridgeHandlerInterfaces.Admin)
    fun enableBridging() {
        pre {
            self.minter != nil: "Cannot enable handler without a minter"
            self.getTargetType() != nil && self.getTargetEVMAddress() != nil:
                "Cannot enable before setting bridge targets"
            !self.isEnabled(): "Handler already enabled"
        }
        post {
            self.isEnabled(): "Problem enabling Handler"
            emit HandlerEnabled(
                handlerType: self.getType(),
                targetType: self.getTargetType()!,
                targetEVMAddress: self.getTargetEVMAddress()!
            )
        }
    }

    /* --- Internal --- */

    /// Returns an entitled reference to the encapsulated minter resource
    access(self)
    view fun borrowMinter(): auth(FlowEVMBridgeHandlerInterfaces.Mint) &{FlowEVMBridgeHandlerInterfaces.TokenMinter}?
}
```
</details>

Any use-case specific `Handler` will then be added to the bridge's `handlers` mapping indexed on the relevant asset
Cadence Type. Once the target EVM address is known, it can be associated with the `Handler`'s type which will prevent
the EVM contract from being onboarded as a bridged Cadence asset.

Once the `Handler` is enabled and the type -> EVM contract address association is established, the bridge will route
requests associated with the special-cased type through the `Handler` for fulfillment.

As you might imagine, this approach requires coordination between bridge maintainers and special-cased asset developers
and it is unlikely that many assets will require such treatment; however, it will be helpful to at least a couple
currently Cadence-native assets to have the option to support non-standard bridge configurations available.

### Considerations

#### NFT Metadata

##### Problem

Platform expectations between EVM & Flow NFT metadata storage differ. Whereas Flow projects generally prioritize onchain
metadata (with the exception of image data), EVM projects typically store NFT metadata in offchain systems - typically
in json blobs found in IPFS, though other patterns are also commonly used.

Since the bridge is public infrastructure, there is a need to generalize the breadth of migrated metadata. Minimizing
metadata would mean:

- Looking solely at the Solidity contract, bridged Flow-native NFTs would have very little identifying information per
  the ERC721 standard - ID & perhaps an image pointer.
- Looking solely at the Cadence contract, bridged EVM-native NFTs would have very little available onchain metadata - ID
  & perhaps an IPFS URI.

For typical bridge infrastructure connecting separate zones of sovereignty, metadata migration could be handled by their
offchain system components. However, the requirement for atomic asset migration between VMs prevents the inclusion of
such offchain systems as they would break atomicity and introduce the need for undesirable trust assumptions. For
example, upon bridging an NFT from Flow to EVM, an offchain listener would need to recognize a request to post metadata
to IPFS, post the metadata, and commit that URI to the defining EVM contract for the relevant NFT. We must trust that a/
the request is served, b/ metadata committed to IPFS, c/ commitment to the EVM contract succeeds and d/ that URI is
correct and contains correct metadata. Then there is the issue of IPFS storage funding.

##### Potential Solutions
___
ℹ️ **Update:** Since the original writing of this FLIP, another solution was proposed and implemented (currently in review
[here](https://github.com/onflow/flow-evm-bridge/pull/20)). While ideating in conversation with
[@austinkline](https://github.com/austinkline), he pointed out that data URLs are a common method for formatting and
inscribing ERC721 token URIs and mentioned he had an initial implementation of serialization in Cadence. This format
opened the door for on-the-fly serialization during the bridging flow, enabling the bridge to take a given NFT and
serialize it according to JSON schemas most commonly recognized in EVM and commit the serialized value as a data URL for
the token's URI.

Since serialization and URI data is all onchain, this URI can be updated as the Cadence NFT's metadata
changes as is the case for NFTs with sub-NFTs for instance. The format targeted for this initial implementation happens
to be OpenSea's metadata standards ([reference docs](https://docs.opensea.io/docs/metadata-standards)). A Mainnet
compatible script can be run here against any account and NFT for easy use and feedback.

Also note that a metadata view serving URI values is leveraged and defaulted to in the event projects wish to define
their bridged asset's metadata themselves. This was introduced as `EVMBridgedMetadata`, but as a standard view in
`MetadataViews` in [this PR](https://github.com/onflow/flow-nft/pull/203). This view can be used to resolve contract-
and token-level metadata as well as enables NFT projects to specify their collection's symbol - a value unsupported by
other views.
___

Alternatively, if Flow projects want their metadata to be served well across VMs, they may take it on themselves to add
offchain IPFS metadata. This would allow the ecosystem to both maintain onchain metadata advantages as well as reach
parity with EVM platform expectations. We may then want to consider a Cadence metadata view specifically for IPFS-stored
metadata to support this use case.

Yet another alternative, it may be possible to expose an API matching that of IPFS gateways so that bridge-stored NFTs
metadata could be served to IPFS clients as they would request URI material from any other provider. Imagine some IPFS
HTTP gateway say `https://ipfs.flow.com/ipfs/CID` where CID is some content-based addressing of the NFT. Requests to the
gateway could be served by access/observer or other data-availability node which, upon receiving a request, would derive
the information about where to locate the NFT from the CID, query for the NFT metadata and serialize it into a JSON blob
conforming to ERC721 metadata standards before returning the blob. On bridging a Flow-native NFT to Flow EVM, the IPFS
URI would be derived and committed to the minted NFT in the EVM side.

This construction would mean that NFT metadata never has to leave the Cadence runtime environment, but can be served on
request as if it was being retrieved from any other IPFS gateway. Of course, the engineering effort is then displaced to
Cadence serialization APIs, the mechanism to construct a CID from the provided NFT content, and an IPFS-like API bolted
on to some DA client-serving node.

Lastly, and likely more appropriately, there is also the option of creating an API similar to OpenSea's metadata API,
which would allow for querying of bridged NFT metadata from a centralized source. This service could in theory be run by
anyone - Flow Foundation or otherwise. While an offchain API would be a centralized solution, it would enable a
secondary source of truth platforms could leverage to serve metadata to their users and is in keeping with expectations
in the EVM ecosystem.

This problem and potential solutions are presented as a point of discussion and are not necessarily in scope for the
bridge's contract design. In the meantime, generalized scripts will be made available so that metadata can be retrieved
from locked NFTs using information solely available on ERC721 contracts defining bridged Flow-native NFTs (see
[@onflow/flow-evm-bridge#7](https://github.com/onflow/flow-evm-bridge/issues/7)).

#### NFT IDs
___
ℹ️ **Update:** While the id values on Cadence NFTs remains, the path forward on this problem was to introduce a simple
NFT interface (`CrossVMNFT.EVMNFT`) containing all ERC721 relevant values including name, symbol, EVM ID, token URI and
defining EVM contract address. Thanks again to [@austinkline](https://github.com/austinkline) for raising this idea in a
[GitHub comment](https://github.com/onflow/flow-nft/pull/126/files#r1463811764).
___

NFT ID values are some of the most critical token metadata, identifying each token as unique. While Flow NFTs define
IDs as `UInt64`, ERC721 IDs are `uint256`. It remains an open question as to how this reduction from `UInt256` to
`UInt64` would affect the uniqueness of NFTs bridged from EVM to Flow and how the bridge should handle such conversions
while safeguarding ownership guarantees upon bridging back.

### Drawbacks

- All contracts deployed by the bridge have the same minimized functionality restricted to their respective ecosystem
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

<summary>Bridge NFT to EVM</summary>

```cadence
import "FungibleToken"
import "NonFungibleToken"
import "ViewResolver"
import "MetadataViews"
import "FlowToken"

import "ScopedFTProviders"

import "EVM"

import "FlowEVMBridge"
import "FlowEVMBridgeConfig"
import "FlowEVMBridgeUtils"

/// Bridges an NFT from the signer's collection in Cadence to the signer's COA in FlowEVM
///
/// NOTE: This transaction also onboards the NFT to the bridge if necessary which may incur additional fees
///     than bridging an asset that has already been onboarded.
///
/// @param nftContractAddress: The Flow account address hosting the NFT-defining Cadence contract
/// @param nftContractName: The name of the NFT-defining Cadence contract
/// @param id: The Cadence NFT.id of the NFT to bridge to EVM
///
transaction(nftContractAddress: Address, nftContractName: String, id: UInt64) {
    
    let nft: @{NonFungibleToken.NFT}
    let coa: auth(EVM.Bridge) &EVM.CadenceOwnedAccount
    let requiresOnboarding: Bool
    let scopedProvider: @ScopedFTProviders.ScopedFTProvider
    
    prepare(signer: auth(CopyValue, BorrowValue, IssueStorageCapabilityController, PublishCapability, SaveValue) &Account) {
        /* --- Reference the signer's CadenceOwnedAccount --- */
        //
        // Borrow a reference to the signer's COA
        self.coa = signer.storage.borrow<auth(EVM.Bridge) &EVM.CadenceOwnedAccount>(from: /storage/evm)
            ?? panic("Could not borrow COA from provided gateway address")
        
        /* --- Retrieve the NFT --- */
        //
        // Borrow a reference to the NFT collection, configuring if necessary
        let viewResolver = getAccount(nftContractAddress).contracts.borrow<&{ViewResolver}>(name: nftContractName)
            ?? panic("Could not borrow ViewResolver from NFT contract")
        let collectionData = viewResolver.resolveContractView(
                resourceType: nil,
                viewType: Type<MetadataViews.NFTCollectionData>()
            ) as! MetadataViews.NFTCollectionData? ?? panic("Could not resolve NFTCollectionData view")
        let collection = signer.storage.borrow<auth(NonFungibleToken.Withdraw) &{NonFungibleToken.Collection}>(
                from: collectionData.storagePath
            ) ?? panic("Could not access signer's NFT Collection")

        // Withdraw the requested NFT & calculate the approximate bridge fee based on NFT storage usage
        let currentStorageUsage = signer.storage.used
        self.nft <- collection.withdraw(withdrawID: id)
        let withdrawnStorageUsage = signer.storage.used
        var approxFee = FlowEVMBridgeUtils.calculateBridgeFee(
                bytes: currentStorageUsage - withdrawnStorageUsage
            ) * 1.10
        // Determine if the NFT requires onboarding - this impacts the fee required
        self.requiresOnboarding = FlowEVMBridge.typeRequiresOnboarding(self.nft.getType())
            ?? panic("Bridge does not support this asset type")
        if self.requiresOnboarding {
            approxFee = approxFee + FlowEVMBridgeConfig.onboardFee
        }

        /* --- Configure a ScopedFTProvider --- */
        //
        // Issue and store bridge-dedicated Provider Capability in storage if necessary
        if signer.storage.type(at: FlowEVMBridgeConfig.providerCapabilityStoragePath) == nil {
            let providerCap = signer.capabilities.storage.issue<auth(FungibleToken.Withdraw) &{FungibleToken.Provider}>(
                /storage/flowTokenVault
            )
            signer.storage.save(providerCap, to: FlowEVMBridgeConfig.providerCapabilityStoragePath)
        }
        // Copy the stored Provider capability and create a ScopedFTProvider
        let providerCapCopy = signer.storage.copy<Capability<auth(FungibleToken.Withdraw) &{FungibleToken.Provider}>>(
                from: FlowEVMBridgeConfig.providerCapabilityStoragePath
            ) ?? panic("Invalid Provider Capability found in storage.")
        let providerFilter = ScopedFTProviders.AllowanceFilter(approxFee)
        self.scopedProvider <- ScopedFTProviders.createScopedFTProvider(
                provider: providerCapCopy,
                filters: [ providerFilter ],
                expiration: getCurrentBlock().timestamp + 1.0
            )
    }

    execute {
        if self.requiresOnboarding {
            // Onboard the NFT to the bridge
            FlowEVMBridge.onboardByType(
                self.nft.getType(),
                feeProvider: &self.scopedProvider as auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
            )
        }
        // Execute the bridge
        self.coa.depositNFT(
            nft: <-self.nft,
            feeProvider: &self.scopedProvider as auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        )
        // Destroy the ScopedFTProvider
        destroy self.scopedProvider
    }
}

```
</details>

<details>

<summary>Bridge NFT from EVM</summary>

```cadence
import "FungibleToken"
import "NonFungibleToken"
import "ViewResolver"
import "MetadataViews"
import "FlowToken"

import "ScopedFTProviders"

import "EVM"

import "FlowEVMBridge"
import "FlowEVMBridgeConfig"
import "FlowEVMBridgeUtils"

/// This transaction bridges an NFT from EVM to Cadence assuming it has already been onboarded to the FlowEVMBridge
/// NOTE: The ERC721 must have first been onboarded to the bridge. This can be checked via the method
///     FlowEVMBridge.evmAddressRequiresOnboarding(address: self.evmContractAddress)
///
/// @param nftContractAddress: The Flow account address hosting the NFT-defining Cadence contract
/// @param nftContractName: The name of the NFT-defining Cadence contract
/// @param id: The ERC721 id of the NFT to bridge to Cadence from EVM
///
transaction(nftContractAddress: Address, nftContractName: String, id: UInt256) {

    let nftType: Type
    let collection: &{NonFungibleToken.Collection}
    let scopedProvider: @ScopedFTProviders.ScopedFTProvider
    let coa: auth(EVM.Bridge) &EVM.CadenceOwnedAccount
    
    prepare(signer: auth(BorrowValue, CopyValue, IssueStorageCapabilityController, PublishCapability, SaveValue, UnpublishCapability) &Account) {
        /* --- Reference the signer's CadenceOwnedAccount --- */
        //
        // Borrow a reference to the signer's COA
        self.coa = signer.storage.borrow<auth(EVM.Bridge) &EVM.CadenceOwnedAccount>(from: /storage/evm)
            ?? panic("Could not borrow COA from provided gateway address")

        // Get the ERC721 contract address for the given NFT type
        self.nftType = FlowEVMBridgeUtils.buildCompositeType(
                address: nftContractAddress,
                contractName: nftContractName,
                resourceName: "NFT"
            ) ?? panic("Could not construct NFT type")

        /* --- Reference the signer's NFT Collection --- */
        //
        // Borrow a reference to the NFT collection, configuring if necessary
        let viewResolver = getAccount(nftContractAddress).contracts.borrow<&{ViewResolver}>(name: nftContractName)
            ?? panic("Could not borrow ViewResolver from NFT contract")
        let collectionData = viewResolver.resolveContractView(
                resourceType: self.nftType,
                viewType: Type<MetadataViews.NFTCollectionData>()
            ) as! MetadataViews.NFTCollectionData? ?? panic("Could not resolve NFTCollectionData view")
        if signer.storage.borrow<&{NonFungibleToken.Collection}>(from: collectionData.storagePath) == nil {
            signer.storage.save(<-collectionData.createEmptyCollection(), to: collectionData.storagePath)
            signer.capabilities.unpublish(collectionData.publicPath)
            let collectionCap = signer.capabilities.storage.issue<&{NonFungibleToken.Collection}>(collectionData.storagePath)
            signer.capabilities.publish(collectionCap, at: collectionData.publicPath)
        }
        self.collection = signer.storage.borrow<&{NonFungibleToken.Collection}>(from: collectionData.storagePath)
            ?? panic("Could not borrow collection from storage path")

        /* --- Configure a ScopedFTProvider --- */
        //
        // Calculate the bridge fee - bridging from EVM consumes no storage, so flat fee
        let approxFee = FlowEVMBridgeUtils.calculateBridgeFee(bytes: 0)
        // Issue and store bridge-dedicated Provider Capability in storage if necessary
        if signer.storage.type(at: FlowEVMBridgeConfig.providerCapabilityStoragePath) == nil {
            let providerCap = signer.capabilities.storage.issue<auth(FungibleToken.Withdraw) &{FungibleToken.Provider}>(
                /storage/flowTokenVault
            )
            signer.storage.save(providerCap, to: FlowEVMBridgeConfig.providerCapabilityStoragePath)
        }
        // Copy the stored Provider capability and create a ScopedFTProvider
        let providerCapCopy = signer.storage.copy<Capability<auth(FungibleToken.Withdraw) &{FungibleToken.Provider}>>(
                from: FlowEVMBridgeConfig.providerCapabilityStoragePath
            ) ?? panic("Invalid Provider Capability found in storage.")
        let providerFilter = ScopedFTProviders.AllowanceFilter(approxFee)
        self.scopedProvider <- ScopedFTProviders.createScopedFTProvider(
                provider: providerCapCopy,
                filters: [ providerFilter ],
                expiration: getCurrentBlock().timestamp + 1.0
            )
    }

    execute {
        // Execute the bridge
        let nft: @{NonFungibleToken.NFT} <- self.coa.withdrawNFT(
            type: self.nftType,
            id: id,
            feeProvider: &self.scopedProvider as auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        )
        // Deposit the bridged NFT into the signer's collection
        self.collection.deposit(token: <-nft)
        // Destroy the ScopedFTProvider
        destroy self.scopedProvider
    }
}
```
</details>

### Compatibility

This bridge design will support bridging fungible and non-fungible token assets between VMs, and explicitly reverts on
mixed tokens - i.e. semi-fungible tokens or multi-token contracts.

Also note that the existence of this VM bridge does not preclude anyone else from building another one. The design
of this bridge and its integration with the COA resource may mean that self-rolled or project-specific bridges - while
increasing cross-VM flexibility and customization - may create infrastructure fragmentation. If this becomes a more
widespread issue, it may be worth considering a standardization of bridge interfaces and/or the addition of a routing
layer to consolidate such fragmentation.

## Prior Art

While the work is happening concurrently, there may be some cross-pollination between this project and the [Axelar
Interchain Token Service](https://github.com/AnChainAI/anchain-axelar-dapper-flowbridge).

## Source Code

The implementation against this design can be found at
[onflow/flow-evm-bridge](https://github.com/onflow/flow-evm-bridge/tree/main)