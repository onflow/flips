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
Cadence FT or NFT token types. To facilitate users bridging tokens between VMs the protocol internalizes capabilities to
deploy new token contracts in either VM state as needed. It serves as a request router & corresponding contract
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
complexity for builders and improves end-user experience for the essential economic activity of moving tokens across
VMs. The abstraction afforded to users by the bridge means that multiple EVM transaction steps can be bundled together
using Cadence to securely realize bridging as a single transaction. The availability of a secure and proven platform
capability for cross-VM bridging can ensure a consistent, higher security bar than leaving developers to implement
project specific token bridges. It also significantly reduces the effort required of developers wishing to build
applications which need to bridge tokens between VMs for an optimal user experience.

## Bridge Specification

Specification outline of the Flow VM bridge for bi-directional flow of FT and NFTs between VM states. While the spec
references NFTs the VM bridge will treat both FTs and NFTs alike, with the caveat that NFTs have more complexity due to
metadata which needs to be made to work for both VMs.

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
Sequential breakdown of the flow for a user bridging a token from EVM to Flow. The high level paths below are broken
down into their respective forks to help with understanding.

#### VM bridge contract functionality

* Initialize Cadence NFT collections
* Maintain Flow <-> EVM contract relationships
* Provide utility methods for information lookup about contracts for either state space

**Path 1: Cadence originated NFT returning to Cadence**

#### Cadence transaction: COA A bridges an EVM NFT back to Cadence

* COA A calls bridge to request bridging of EVM NFT back into Cadence providing requested NFT type, id, and verification
  of COA ownership
* Bridge contract calls into EVM to confirm COA A is the owner of the requested NFT. If so, process continues; otherwise
  reverts
* Bridge contract calls into EVM contract to burn EVM NFT
* Bridge contract unlocks corresponding Cadence NFT from VM bridge contract escrow storage
* Returns NFT on call to bridge contract

#### Cadence transaction: COA B bought COA A's EVM NFT on FlowEVM, and then wants to bridge it back to Cadence

* COA B calls bridge contract to request bridging of EVM NFT back into Cadence providing requested NFT type, id, and
  verification of COA ownership
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
* Upon Cadence call to bridge, COA A must include EVM transaction transferring NFT ownership to bridge account
* Bridge checks that caller is the owner of the NFT on EVM side before transferring to Bridge account COA address
* Bridge executed provided contract call, transferring NFT to Bridge COA address
* Bridge validates its ownership of the NFT on EVM side after transfer, thus locking NFT to be bridged
* Bridge mints NFT from Flow bridge NFT contract and returns to caller
* Collection is configured if necessary and deposited to COA A's account

## Design Proposal

### Context
<!-- TODO -->

### Overview

The central bridge contract will act as a request router & corresponding contract registrar, additionally configuring
contracts on either side of the VM to facilitate bridge requests as they arrive. Deployed contracts are “owned” by the
bridge, but owner interactions are mediated by contract logic on either end in addition to multi-sig patterns consistent
with other core network infrastructure accounts.

On the EVM side, a central contract factory will instantiate Solidity ERC20 & ERC721 contracts as directed by calls from
the central Cadence contract’s Cadence Owned Account. This factory will also implement a number of helper methods to
give the bridge account visibility into the EVM environment. These methods might include things like retrieving an asset
type, determining if EVM contracts are bridge-owned, validating asset ownership, etc. so the COA has a central trusted
source of truth for critical state assertions.

Below are diagrams depicting the call flows for both Flow and EVM-native NFTs.

#### Flow-Native: Flow -> EVM

*Lock in Cadence & Mint in EVM*

1. Determine if asset is FT or NFT
    - Check if resource is an instance of FT.Vault or NFT.NFT (excluding overlapping instances, at least for POC)
2. Determine if asset is Flow or EVM-native
    - Check if resource is defined in bridge-hosted NFT or NFT contract - if so, EVM-native, else Flow-native
3. Determine if BridgeNFTLocker contract has been initialized for this NFT type - if not, initialize setup
    1. Call into EVM, telling BridgedNFTFactory to deploy a new FlowBridgedNFT contract, passing identifying info about the NFT & noting new EVM contract address
    2. Derive a contract name from the NFT identifier, & deploy template-generated Cadence contract to bridge account, passing EVM contract Address
4. Borrow a reference to the newly deployed BridgeNFTLocker contract & passthrough bridge request
    1. Lock the NFT in the Locker resource
    2. Call to FlowBridgedNFT.sol to mint to caller
5. Locker contract calls corresponding EVM FlowBridgedNFT contract to mint NFT to the provided EVMAddress // Simplified with .sol interface + delegateCall in Factory.sol?

![Flow-native Flow to EVM](20231222-evm-vm-bridge-resources/flow_native_to_evm.png)

#### Flow-Native: EVM -> Flow

*Unlock in Cadence & Burn in EVM*

1. Determine if asset is FT or NFT
    - Call into Coordinator to determine if EVM address target is ERC20 or ERC721 instance or invalid
2. Determine if asset is Flow or EVM-native
    - Call into Factory contract to determine if target of EVM call is IFlowCrossVMContract.sol instance && is contained in bridgedNFTContracts
3. Determine if an NFT Locker has been initialized (it will be by construction)
4. Borrow the BridgeNFTLocker.cdc contract, deriving name from from the type identifier returned from FlowBridgedNFT.sol
4. Validate the caller is the current owner (or getApproved(tokenID)) of the EVM NFT to be bridged
5. Execute the caller-provided approve() calldata
7. Validate the bridge contract COA is approved to manipulate the NFT in FlowBridgedNFT.sol as result of executed call
8. Bridge contract COA calls to FlowBridgedNFT.sol to burn the NFT // Simplified with a .sol interface + delegateCall in Factory.sol?
6. Bridge contract withdraws NFT from NFTLocker and returns to caller


![Flow-native EVM to Flow](20231222-evm-vm-bridge-resources/flow_native_to_flow.png)

#### EVM-Native: EVM -> Flow

*Mint in Flow & Transfer in EVM*

1. Determine if asset is FT or NFT
    - Call into Factory to determine if EVM address target is ERC20 or ERC721 instance or invalid
2. Determine if asset is Flow or EVM-native
    - Call into Factory contract, checking if target of EVM call is IFlowCrossVMContract.sol instance && is contained in bridgedNFTContracts
3. Determine if BridgedNFT contract has been deployed - if not initialize
    - Derive a contract name from the EVM NFT contract address + name & deploy template-generated Cadence contract to bridge account, passing EVM source contract address
4. Borrow newly deployed BridgedNFT contract & passthrough
5. BridgedNFT validates the caller is currently owner (or getApproved(tokenID)) of EVM NFT
6. BridgedNFT executes EVM approve call provided by the caller, approving bridge COA to act on NFT
6. BridgedNFT executes NFT transfer from bridge COA, completing the transfer to the bridge account in EVM
7. BridgedNFT validates its contract COA is now the owner
8. BridgedNFT subsequently mints an NFT from itself & returns
9. Caller then creates & Collection from NFT & configures, finally depositing to their account

![Flow-native Flow to EVM](20231222-evm-vm-bridge-resources/evm_native_to_flow.png)

#### EVM-Native: Flow -> EVM

*Burn in Flow & Transfer in EVM*

1. Determine if asset is FT or NFT
    - Check if resource is an instance of FT.Vault or NFT.NFT (excluding overlapping instances, at least for POC)
2. Determine if asset is Flow or EVM-native
    - Check if resource is defined in bridge-hosted NFT or NFT contract - if so, EVM-native, else Flow-native
3. Determine if BridgedNFT contract has been deployed (it will be by construction)
4. Borrow the BridgedNFT contract from the NFT type identifier & passthrough bridge request
5. BridgedNFT burns the NFT
6. BridgedNFT calls to EVM contract, transferring NFT to defined recipient
7. BridgedNFT confirms recipient is owner of the NFT post-transfer

![Flow-native Flow to EVM](20231222-evm-vm-bridge-resources/evm_native_to_evm.png)

#### In Aggregate
![FlowEVM VM Bridge Design Overview](20231222-evm-vm-bridge-resources/overview.png) *The bridge contract can be thought
of here as a router for requests to bridge to and from Flow. It determines if the requested asset is Flow or EVM native
and whether it’s an FT or NFT. If needed, it performs contract initialization on either side of the VM. From there, it
routes requests to the appropriate contract which fulfills the asset bridge request.*

### Implementation Details

### Contract Roles & Concerns

#### Flow

- **Bridge**
  - Unified entry point for bridging assets between VMs
  - Unified query point for NFT Locker contracts & to assess Flow x EVM contract associations
  - Owning (via contract COA) all deployed contracts on EVM side
  - Owning (via contract COA) all EVM-native assets on EVM side when bridged to Flow - equivalent to locking
  - Initializing NFT & FT Locker contracts when Flow-native assets first bridged to EVM
  - Initializing BridgedFT & BridgedNFT contracts defining EVM-native assets when first bridged to Flow
- **FT/NFT Locker**
  - Lock Flow-native tokens bridging to EVM
  - Unlock Flow-native tokens bridging from EVM
  - Serve query requests about locked tokens
  - Point to the corresponding EVM contract
- **Bridged FT/NFT**
  - Define EVM-native tokens bridged from EVM to Flow
  - Point to the corresponding EVM-native contract
  - Serve as secondary bridging interface, enabling easy bridging back to EVM
      - Result of implementing combination of `CrossVM` contract interface along with `BridgeableVault` or
        `BridgeableCollection` resource interfaces
      - e.g. `collection.bridgeToEVM(id: UInt64, to: EVMAddress)`
      - e.g. `vault.bridgeToEVM(amount: UFIx64, to: EVMAddress)`

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

#### Interfaces

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
        access(all) fun bridgeToEVM(amount: UFix64, to: EVM.EVMAddress)
    }

    /// Enables a bridging entrypoint to EVM on an implementing Collection
    access(all) resource interface EVMBridgeableCollection {
        access(all) fun bridgeToEVM(id: UInt64, to: EVM.EVMAddress)
    }
}
```
</details>

<details>
<summary>FlowEVMBridgeFactory.sol</summary>

```solidity
// TODO - Factory & EVM inspector
```
</details>

<details>
<summary>FlowBridgedAsset.sol</summary>

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
<summary>IFlowBridgedFT.sol</summary>

```solidity
// TODO - Template for bridged EVM-native FTs
```
</details>

#### NFT Metadata

### Considerations

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

This design optimized for distributed asset storage and contract-mediated access. However, it also introduced additional
complexity and secondarily obscurity for what should be a highly transparent system. Additionally, since multisig access
is planned for the bridge and auxiliary accounts, centralization is ultimately no further improved by this design, at
least while custody is maintained on these accounts.

### Performance Implications

#### Migrations & Storage Usage

Previous network migrations have been complicated by single accounts using large amounts of storage. With a centralized
storage design, it's likely that (over time) the bridge account will consume a large amount of storage and that, given
the need to store bridge Flow-native assets indefinitely, that storage will likely only ever increase. Even if this
assumption is not true, it's to our benefit to consider and plan as if it is if account storage usage is a network-wide
issue.

With that said, saving state to a single account shouldn't be problematic until storage usage reaches >10GB of data
which should give the team some time to figure out how to handle this edge case during migrations before the problem is
encountered.

### Best Practices

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
cases where the instances where types overlap - i.e. semi-fungible tokens or multi-token contracts. Of course, this
bridge also dovetails with the ongoing virtualized EVM work, so is dependent on the existence of that environment.

## Prior Art

While the work is happening someone concurrently, there may be some cross-pollination between the [Axelar Interchain
Token Service](https://github.com/AnChainAI/anchain-axelar-dapper-flowbridge) and this project.

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
___

<!-- TODO: Wipe -->
## Thoughtpad

- What schema to use for derived contract names?
  - Flow-native -> Lockers
  - EVM-native -> Bridged Asset
  - 