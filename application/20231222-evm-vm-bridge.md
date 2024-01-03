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
complexity for builders and improves end-user experience for the essential economic activity of moving tokens across VMs. 
The abstraction afforded to users by the bridge means that multiple EVM transaction steps can be bundled together using 
Cadence to securely realize bridging as a single transaction. The availability of a secure and proven platform 
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

#### NFT Metadata

### Considerations

### Drawbacks

- All contracts deployed by the bridge have the same minimized functionality, restricted to their respective ecosystem standards
- Centralized storage of all bridged assets and their definitions presents a single, high-value target for potential hacks.
  - This vector is minimized by the fact that the bridge exists solely at the protocol and contract levels. Taking protocol security for granted, contract logic and key management are the primary considerations for compromise. This bridge benefits from the contained state space afforded by virtualizing the EVM environment in that offchain systems are not required for its function between VMs.

### Considered Alternatives

Motivated by the aforementioned drawback of centralized storage, previous iterations involved a network of distributed accounts around a primary bridge account and contract. This primary contract served as the entrypoint for bridging between VMs, deploying templated locking & asset-defining auxiliary contracts to distinct, contract-generated accounts. The primary contract would conditionally deploy these auxiliary contracts on a per-asset basis as needed, maintaining a registry for where each asset is locked/defined and routing bridge requests to their appropriate contracts.

This design optimized for distributed asset storage and contract-mediated access. However, it also introduced additional complexity and secondarily obscurity for what should be a highly transparent system. Additionally, since multisig access is planned for the bridge and auxiliary accounts, centralization is ultimately no further improved by this design, at least while custody is maintained on these accounts.

### Performance Implications

#### Migrations & Storage Usage

Previous network migrations have been complicated by single accounts using large amounts of storage. With a centralized storage design, it's likely that (over time) the bridge account will consume a large amount of storage and that, given the need to store bridge Flow-native assets indefinitely, that storage will likely only ever increase. Even if this assumption is not true, it's to our benefit to consider and plan as if it is if account storage usage is a network-wide issue.

With that said, saving state to a single account shouldn't be problematic until storage usage reaches >10GB of data which should give the team some time to figure out how to handle this edge case during migrations before the problem is encountered.

### Best Practices

### Examples

### Compatibility

This bridge design will support bridging fungible and non-fungible token assets between VMs, and doesn't account for cases where the instances where types overlap - i.e. semi-fungible tokens or multi-token contracts. Of course, this bridge also dovetails with the ongoing virtualized EVM work, so is dependent on the existence of that environment.

## Prior Art

While the work is happening someone concurrently, there may be some cross-pollination between the [Axelar Interchain Token Service](https://github.com/AnChainAI/anchain-axelar-dapper-flowbridge) and this project.

## Questions & Discussion Topics

- What does the interplay between risk vectors, tokenomics, and UX requirements imply for the fee amounts charged for bridging between VMs
  - Do we charge on a per instance or per locked storage unit basis?
- How will we handle either bridging or serving metadata for bridged Flow-native NFTs given the difference in metadata standards between Cadence & EVM?