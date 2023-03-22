---
status: Proposed 
flip: 72
title: AuthAccount Capability Management
forum: https://forum.onflow.org/t/account-linking-authaccount-capabilities-management/4314
authors: Giovanni Sanchez (giovanni.sanchez@dapperlabs.com)
editor: Jerome Pimmel (jerome.pimmel@dapperlabs.com)
updated: 2023-02-23
---

# AuthAccount Capability Management

<details>
<summary>Table of Contents</summary>

- [Context](#context)
- [Objective](#objective)
    - [Non-goals](#non-goals)
    - [Essential](#essential)
    - [For Consideration](#for-consideration)
- [Existing Work](#existing-work)
- [Motivation](#motivation)
- [Design Proposal](#design-proposal)
- [Example Implementation](#example-implementation)
- [Considered For Inclusion](#considered-for-inclusion)
    - [Events](#events)
    - [NonFungibleToken & MetadataViews Standards Implementation](#nonfungibletoken--metadataviews-standards-implementation)
    - [Storage Iteration Convenience Methods](#storage-iteration-convenience-methods)
- [Drawbacks](#drawbacks)
    - [Considerations](#considerations)
        - [Visibility into All Sub-Account Storage](#visibility-into-all-sub-account-storage)
        - [Sources of Truth](#sources-of-truth)
        - [Auditability and Revocation](#auditability-and-revocation)
        - [Limiting Delayed Attack Vectors](#limiting-delayed-attack-vectors)
        - [Lack of Ultimate Control](#lack-of-ultimate-control)
- [Alternatives Considered](#alternatives-considered)
    - [Key-Based Child Accounts](#key-based-child-accounts)
    - [Native Notion of Child Accounts](#native-notion-of-child-accounts)
- [Best Practices](#best-practices)
    - [Creating and Funding Child Accounts](#creating-and-funding-child-accounts)
        - [DApp-Funded, DApp-Custodied](#dapp-funded-dapp-custodied)
        - [DApp-Funded, User-Custodied](#dapp-funded-user-custodied)
        - [User-Funded, DApp-Custodied](#user-funded-dapp-custodied)
        - [User-Funded, User-Custodied](#user-funded-user-custodied)
    - [Linking existing account as child account](#linking-existing-account-as-child-account)
    - [Revoking a child account](#revoking-a-child-account)
    - [Granting a child account Capabilities](#granting-a-child-account-capabilities)
- [Tutorials and Examples](#tutorials-and-examples)
    - [Adding an Account as a Child Account](#adding-an-account-as-a-child-account-aka-account-linking)
    - [Using a Child Account's FlowToken Vault](#using-a-child-accounts-flowtoken-vault)
- [Compatibility](#compatibility)
- [User Impact](#user-impact)
- [Questions and Discussion Topics](#questions-and-discussion-topics)
    - [Verbiage](#verbiage)
    - [Open Questions](#open-questions)
- [References](#references)

</details>

# Context

One of Flow’s biggest advantages is its ease of user onboarding; however, as discussed recently, the current state does not [go far enough](https://flow.com/post/flow-blockchain-mainstream-adoption-easy-onboarding-wallets). With the focus on [hybrid custody](https://forum.onflow.org/t/hybrid-custody/4016) and [progressive onboarding flows](https://youtu.be/0eYX_S4jUYM) and the recent work on [AuthAccount capabilities](https://github.com/onflow/flips/pull/53), there is a need for a mechanism to both maintain these capabilities as well as enable dApps to facilitate user interactions that deal with those sub-accounts and the assets within them.

# Objective

## Non-Goals

Before continuing, let’s clear up some common misconceptions about what’s being proposed. Here’s what this Flip is **not** proposing:

- Create a standard for shared access on a user’s primary account
- Introduce guardrails for the use of AuthAccount Capabilities

## Essential

This FLIP proposes a standard for the creation and management of child accounts to support  progressive onboarding models. The standard defines the resource representing the parent-child relationship between accounts, identifies an account as a parent and/or child, and related management functions including:

- Child account creation
- Child AuthAccount capability management
- Viewing existing child accounts
- Adding an account as a child account
- Revoking hybrid custody approval/access granted to a child account
- Identifying an account’s child accounts
- Identifying an account’s parent account(s)
- Implement useful events builders can rely on

## For Consideration

These are features we have thought about including in this standard, but are uncertain if they should be included in the standard

- Delegating Capabilities to a child account from the parent’s Collection
- Easily viewing the assets in an user’s child accounts

# Existing Work

Guidance on implementation of this standard for wallets & marketplaces can be seen [here](https://developers.flow.com/account-linking). Below are contract and dApp implementations that were used as sandboxes for on-chain gaming, native attachments, progressive onboarding and eventually culminated in the development of this Flip.

- [Simplified linked accounts Cadence](https://github.com/onflow/linked-accounts)
- [Linked accounts Cadence in context](https://github.com/onflow/sc-eng-gaming/tree/sisyphusSmiling/child-account-auth-acct-cap)
- [Walletless onboarding dApp example](https://github.com/onflow/walletless-arcade-example)
- [v1 Testnet contract deployment](https://f.dnz.dev/0x1b655847a90e644a/ChildAccount)
- [v2 Testnet contract deployment](https://f.dnz.dev/0x1b655847a90e644a/LinkedAccounts)

The work thus far is a product of lots of iteration. Much thought has been put into alternative approaches ([more details below](#alternatives-considered)) including key-based and language API approaches. However, due to a combination of consequential technical issues, prioritization of iteration speed, and avoiding dependency on external actors, the Capability-based contract & resource design outlined in this FLIP is the one proposed. Of course, this FLIP is just that - a proposal - and alternative approaches and challenges to this design are welcome for discussion.

# Motivation

We can imagine that a user can have any number of child accounts tied to their main account, one for each dApp they’re using. They will need to maintain AuthAccount capabilities for each, create new child accounts, add existing accounts as child accounts, issue child account capabilities and easily manage assets across their sub-accounts without the need to first transfer assets to their main account.

For example, a user might have a child account associated with a game dApp containing an NFT and in-game FungibleTokens. They should be able to sign into an NFT marketplace and view the NFTs across all of their linked accounts. Then, without needing to first transfer the NFT to the signing account, list that NFT for sale or purchase an NFT with the FungibleTokens in their child account, signing as the parent account. Similarly, if an offer is submitted on an NFT in a child account, a wallet provider managing the parent or dApp in which the parent accounts is authenticated (i.e. a marketplace) should be able to notify the owner and allow them to accept the offer, again signing as the parent account.

To put it simply, this standard sets down primitives through which well-known web2 account-to-application authorization schemes can be modeled in our decentralized context.

Accomplishing this vision successfully - success here meaning building a secure and universally useful construct that serves as solid ground truth while satisfying the aforementioned objectives - can further distinguish Flow among other chains as a platform for builders to create totally new dApps not possible elsewhere.

# FAQ

- I’m concerned my main account can get adopted by another Flow account, how is that prevented?
    - Issuing Capabilities on an AuthAccount pose the same sort of vulnerability vector as adding keys onto an account. Since these features are in a similar class - that of delegated authority - the community has worked to introduce a new [“Super User Account” feature](https://forum.onflow.org/t/super-user-account/4088), similar to `sudo` in Linux systems. This means issuing a Capability on your AuthAccount, as well as adding keys, deploying/deleting contracts, and other potentially dangerous operations, can only occur in transactions for which you have given explicit sudo-like permission for. Note that the concept is not the focus of this Flip, and is still in discussion meaning the exact construct will likely evolve.
- Why would I want someone else to have access to my account?
    - To clarify, the design isn’t for you to give a dApp access to your main account. The dApp creates an account it maintains access to that it uses to interact with Flow. Then, when you’re ready to control the dApp’s account more directly, the dApp issues a Capability on its AuthAccount to you, thereby linking its account as a child of your main account. This lets the dApp act on your behalf for only the assets in the child account. Everything in your main account remains only in your control, while allowing you to act on the assets in the child account without the need for the dApp to mediate.
- Why would I want a separate account I share access with? Doesn’t that put all of my assets at custodial risk?
    - Again, only the child account shares access with another party, meaning your main account is safe from custodial risk. Additionally, the second party’s access can be revoked by the parent account at any time without the need to maintain a signing key on the child account. In fact, partitioning assets across accounts in this way enhances security over a model that requires all transactions be signed by your main account. A user can keep all of their more valuable assets in their main account, out of reach without a user-signed transaction, while keeping less valuable dApp assets in a shared account for ease of use.

# User Benefit

A standard resource managing all child account capabilities allows for a unified user experience, enabling any dApp in the ecosystem to connect a user’s primary wallet and get global context into the entirety of the user’s accounts and assets that would otherwise be fragmented and unknown. For the end user, this means:

- greater organization of their app accounts and assets across them
- reduced risk of forgetting about assets in secondary accounts
- reduced custodial risk to assets in their main account due to partitioned access between parent and child accounts
- fewer transfers needed to engage with assets in child accounts as connected dApps can leverage the standard child account managing resource to utilize multiple AuthAccount capabilities within a single signed transaction
- unified asset balance views across all of their accounts (e.g. Alice has 3 accounts with TokenA Vaults, but only needs to know the sum of all their balances; Alice has 3 accounts with NFTs but is only interested in viewing her total collection overview on a marketplace profile page)

And for builders:

- clear expectations on where child account capabilities will reside and how to engage with them
- the ability to create self-custodial dApps leveraging web2 onboarding flows with the added flexibility of delegating shared control of an app account to a user in the future

# Design Proposal

> :warning: The diagram below will be updated shortly to be in sync with the recently updated implementation

![child_account_resources_overview](./20230223-auth-account-capability-management-standard-resources/child_account_resources_overview.jpg)
*The hierarchical model between accounts is reflected by the `Collection` in the parent account & `Handler` in the child account. A `Collection` identifies a parent account, map its child accounts to `NFT`, and enables a user to create & manage multiple child accounts. `Handler`s identify a child account, its parent, & enables its creating Manager to grant the child account Capabilities.*

> ℹ️ Note that AuthAccount Capabilities are not currently enabled on mainnet, only testnet. You may also use a [preview version of flow-cli](https://github.com/onflow/flow-cli/releases/tag/v0.45.1-cadence-attachments-dev-wallet) to utilize the feature in your local emulator environment

Taking a look at our [updated prototype implementation](https://github.com/onflow/linked-accounts/blob/rename-refactor/contracts/LinkedAccounts.cdc), you'll find the following constructs:

- `Collection` - a resource associated with a parent account that will allow its owner to store and access (currently via reference) AuthAccount Capabilities to which it has been delegated access. Enables creation of child accounts, linking existing accounts as child accounts and issuing/revoking Capabilities directly to/from child accounts that are accessed via reference. Note that any accounts created by this resource are, at least via single signed transactions, funded via the parent account (AKA user-funded) as far as the contracts are concerned.
- `Handler` - a resource that is held by any account and identifies it as a child / secondary account. This will store its parent / main account address along with some metadata info that will identify the secondary account (e.g. what dApp created it) and methods related to managing the child account.
- `ChildAccountInfo` - a metadata struct containing information about the intended purpose of a given child account. In a world where dApps create these use-case specific accounts for users, it would be helpful to know at a glance the context of a given child account.
- `NFT` - a resource containing a Capability to the child’s AuthAccount and its `Handler`, created to be stored as a nested resource in `Collection`. This construct is a resource for two reasons
    1. We want to leverage the existence safeguards inherent to resources.
    2. The uniqueness guarantees of resources prevent copying which would be very difficult to detect and prevent with structs.

## Example Implementation

The constructs listed above have been prototyped and are available for reference below. For more context on how these function together, the [demo dApp Cadence repo](https://github.com/onflow/sc-eng-gaming/tree/sisyphusSmiling/child-account-auth-acct-cap) and simplified [linked accounts](https://github.com/onflow/linked-accounts) repos will be helpful.

<details>
<summary>Collection</summary>

```js
/** --- Collection --- */
//
/// Interface that allows one to view information about the owning account's
/// child accounts including the addresses for all child accounts and information
/// about specific child accounts by Address
///
pub resource interface CollectionPublic {
    pub fun getAddressToID(): {Address: UInt64}
    pub fun getLinkedAccountAddresses(): [Address]
    pub fun getIDOfNFTByAddress(address: Address): UInt64?
    pub fun getIDs(): [UInt64]
    pub fun isLinkActive(onAddress: Address): Bool
    pub fun borrowNFT(id: UInt64): &NonFungibleToken.NFT {
        post {
            result.id == id: "The returned reference's ID does not match the requested ID"
        }
    }
    pub fun borrowNFTSafe(id: UInt64): &NonFungibleToken.NFT? {
        post {
            result == nil || result!.id == id: "The returned reference's ID does not match the requested ID"
        }
    }
    pub fun borrowLinkedAccountsNFTPublic(id: UInt64): &LinkedAccounts.NFT{LinkedAccounts.NFTPublic}? {
        post {
            (result == nil) || (result?.id == id):
                "Cannot borrow ExampleNFT reference: the ID of the returned reference is incorrect"
        }
    }
    pub fun borrowViewResolverFromAddress(address: Address): &{MetadataViews.Resolver}
}

/// A Collection of LinkedAccounts.NFTs, maintaining all delegated AuthAccount & Handler Capabilities in NFTs.
/// One NFT (representing delegated account access) per linked account can be maintained in this Collection,
/// enabling public view Capabilities and owner-related management methods, including removing linked accounts, as
/// well as granting & revoking Capabilities. 
/// 
pub resource Collection : CollectionPublic, NonFungibleToken.Provider, NonFungibleToken.Receiver, NonFungibleToken.CollectionPublic, MetadataViews.ResolverCollection {
    /// Mapping of contained LinkedAccount.NFTs as NonFungibleToken.NFTs
    pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}
    /// Mapping linked account Address to relevant NFT.id
    pub let addressToID: {Address: UInt64}

    /// Returns the NFT as a Resolver for the specified ID
    ///
    /// @param id: The id of the NFT
    ///
    /// @return A reference to the NFT as a Resolver
    ///
    pub fun borrowViewResolver(id: UInt64): &{MetadataViews.Resolver}

    /// Returns the IDs of the NFTs in this Collection
    ///
    /// @return an array of the contained NFT resources
    ///
    pub fun getIDs(): [UInt64]

    /// Returns a reference to the specified NonFungibleToken.NFT with given ID
    ///
    /// @param id: The id of the requested NonFungibleToken.NFT
    ///
    /// @return The requested NonFungibleToken.NFT, panicking if there is not an NFT with requested id in this
    ///         Collection
    ///
    pub fun borrowNFT(id: UInt64): &NonFungibleToken.NFT

    /// Returns a reference to the specified NonFungibleToken.NFT with given ID or nil
    ///
    /// @param id: The id of the requested NonFungibleToken.NFT
    ///
    /// @return The requested NonFungibleToken.NFT or nil if there is not an NFT with requested id in this
    ///         Collection
    ///
    pub fun borrowNFTSafe(id: UInt64): &NonFungibleToken.NFT?
    
    /// Returns a reference to the specified LinkedAccounts.NFT as NFTPublic with given ID or nil
    ///
    /// @param id: The id of the requested LinkedAccounts.NFT as NFTPublic
    ///
    /// @return The requested LinkedAccounts.NFTublic or nil if there is not an NFT with requested id in this
    ///         Collection
    ///
    pub fun borrowLinkedAccountsNFTPublic(id: UInt64): &LinkedAccounts.NFT{LinkedAccounts.NFTPublic}?

    /// Returns whether this Collection has an active link for the given address.
    ///
    /// @return True if there is an NFT in this collection associated with the given address that has active
    /// AuthAccount & Handler Capabilities and a Handler in the linked account that is set as active
    ///
    pub fun isLinkActive(onAddress: Address): Bool

    /// Takes a given NonFungibleToken.NFT and adds it to this Collection's mapping of ownedNFTs, emitting both
    /// Deposit and AddedLinkedAccount since depositing LinkedAccounts.NFT is effectively giving a Collection owner
    /// delegated access to an account
    ///
    /// @param token: NonFungibleToken.NFT to be deposited to this Collection
    ///
    pub fun deposit(token: @NonFungibleToken.NFT)
    
    /// Withdraws the LinkedAccounts.NFT with the given id as a NonFungibleToken.NFT, emitting standard Withdraw
    /// event along with RemovedLinkedAccount event, denoting the delegated access for the account associated with
    /// the NFT has been removed from this Collection
    ///
    /// @param withdrawID: The id of the requested NFT
    ///
    /// @return The requested LinkedAccounts.NFT as a NonFungibleToken.NFT
    ///
    pub fun withdraw(withdrawID: UInt64): @NonFungibleToken.NFT

    /// Withdraws the LinkedAccounts.NFT with the given Address as a NonFungibleToken.NFT, emitting standard 
    /// Withdraw event along with RemovedLinkedAccount event, denoting the delegated access for the account
    /// associated with the NFT has been removed from this Collection
    ///
    /// @param address: The Address associated with the requested NFT
    ///
    /// @return The requested LinkedAccounts.NFT as a NonFungibleToken.NFT
    ///
    pub fun withdrawByAddress(address: Address): @NonFungibleToken.NFT

    /// Getter method to make indexing linked account Addresses to relevant NFT.ids easy
    ///
    /// @return This collection's addressToID mapping, identifying a linked account's associated NFT.id
    ///
    pub fun getAddressToID(): {Address: UInt64}

    /// Returns an array of all child account addresses
    ///
    /// @return an array containing the Addresses of the linked accounts
    ///
    pub fun getLinkedAccountAddresses(): [Address] 

    /// Returns the id of the associated NFT wrapping the AuthAccount Capability for the given
    /// address
    ///
    /// @param ofAddress: Address associated with the desired LinkedAccounts.NFT
    ///
    /// @return The id of the associated LinkedAccounts.NFT or nil if it does not exist in this Collection
    ///
    pub fun getIDOfNFTByAddress(address: Address): UInt64?

    /// Returns a reference to the NFT as a Resolver based on the given address
    ///
    /// @param address: The address of the linked account
    ///
    /// @return A reference to the NFT as a Resolver
    ///
    pub fun borrowViewResolverFromAddress(address: Address): &{MetadataViews.Resolver}

    /// Allows the Collection to retrieve a reference to the NFT for a specified child account address
    ///
    /// @param address: The Address of the child account
    ///
    /// @return the reference to the child account's Handler
    ///
    pub fun borrowLinkedAccountNFT(address: Address): &LinkedAccounts.NFT?

    /// Returns a reference to the specified linked account's AuthAccount
    ///
    /// @param address: The address of the relevant linked account
    ///
    /// @return the linked account's AuthAccount as ephemeral reference or nil if the
    ///         address is not of a linked account
    ///
    pub fun getChildAccountRef(address: Address): &AuthAccount?

    /// Returns a reference to the specified linked account's Handler
    ///
    /// @param address: The address of the relevant linked account
    ///
    /// @return the child account's Handler as ephemeral reference or nil if the
    ///         address is not of a linked account
    ///
    pub fun getHandlerRef(address: Address): &Handler?

    /// Add an existing account as a linked account to this Collection. This would be done in either a multisig
    /// transaction or by the linking account linking & publishing its AuthAccount Capability for the Collection's
    /// owner.
    ///
    /// @param childAccountCap: AuthAccount Capability for the account to be added as a child account
    /// @param childAccountInfo: Metadata struct containing relevant data about the account being linked
    ///
    pub fun addAsChildAccount(
        linkedAccountCap: Capability<&AuthAccount>,
        linkedAccountMetadata: AnyStruct{LinkedAccountMetadataViews.AccountMetadata},
        linkedAccountMetadataResolver: AnyStruct{LinkedAccountMetadataViews.MetadataResolver}?,
        handlerPathSuffix: String
    ) {
        pre {
            linkedAccountCap.check():
                "Problem with given AuthAccount Capability!"
            !self.addressToID.containsKey(linkedAccountCap.borrow()!.address):
                "Collection already has LinkedAccount.NFT for given account!"
            self.owner != nil:
                "Cannot add a linked account without an owner for this Collection!"
        }
    }

    /// Adds the given Capability to the Handler at the provided Address
    ///
    /// @param to: Address which is the key for the Handler Cap
    /// @param cap: Capability to be added to the Handler
    ///
    pub fun addCapability(to: Address, _ cap: Capability) {
        pre {
            self.addressToID.containsKey(to):
                "No linked account NFT with given Address!"
        }
    }

    /// Removes the capability of the given type from the Handler with the given Address
    ///
    /// @param from: Address indexing the Handler Capability
    /// @param type: The Type of Capability to be removed from the Handler
    ///
    pub fun removeCapabilities(from: Address, types: [Type]): [Type] {
        pre {
            self.addressToID.containsKey(from):
                "No linked account with given Address!"
        }
    }

    /// Remove Handler, returning its Address if it exists.
    /// Note, removing a Handler does not revoke key access linked account if it has been added. This should be
    /// done in the same transaction in which this method is called.
    ///
    /// @param withAddress: The Address of the linked account to remove from the mapping
    ///
    /// @return the Address of the account removed or nil if it wasn't linked to begin with
    ///
    pub fun removeLinkedAccount(withAddress: Address): [Type] {
        pre {
            self.addressToID.containsKey(withAddress):
                "This Collection does not have NFT with given Address: ".concat(withAddress.toString())
        }
    } 
}
```
</details>

<details>
<summary>Handler</summary>
<code>

```js
/** --- Handler --- */
//
pub resource interface HandlerPublic {
    pub fun getParentAddress(): Address
    pub fun getGrantedCapabilityTypes(): [Type]
    pub fun isCurrentlyActive(): Bool
}

/// Identifies an account as a child account and maintains info about its parent & association as well as
/// Capabilities granted by its parent account's Collection
///
pub resource Handler : HandlerPublic, MetadataViews.Resolver {
    /// Pointer to this account's parent account
    access(contract) var parentAddress: Address
    /// The address of the account where the ccountHandler resource resides
    access(contract) let address: Address
    /// Metadata about the purpose of this child account guarantees standard minimum metadata is stored
    /// about linked accounts
    access(contract) let metadata: AnyStruct{LinkedAccountMetadataViews.AccountMetadata}
    /// Resolver struct to increase the flexibility, allowing implementers to resolve their own structs
    access(contract) let resolver: AnyStruct{LinkedAccountMetadataViews.MetadataResolver}?
    /// Capabilities that have been granted by the parent account
    access(contract) let grantedCapabilities: {Type: Capability}
    /// Flag denoting whether link to parent is still active
    access(contract) var isActive: Bool

    /// Returns the metadata view types supported by this Handler
    ///
    /// @return An array of metadata view types
    ///
    pub fun getViews(): [Type]
    
    /// Returns the requested view if supported or nil otherwise
    ///
    /// @param view: The Type of metadata struct requests
    ///
    /// @return The metadata of requested Type if supported and nil otherwise
    ///
    pub fun resolveView(_ view: Type): AnyStruct?

    pub fun getAddress(): Address {
        pre {
            self.owner != nil:
                "This Handler does not currently reside within an account!"
        }
        post {
            result == self.owner!.address:
                "This Handler is not located in the correct linked account!"
        }
    }

    /// Returns the Address of this linked account's parent Collection
    ///
    pub fun getParentAddress(): Address
    
    /// Returns the metadata related to this account's association
    ///
    pub fun getAccountMetadata(): AnyStruct{LinkedAccountMetadataViews.AccountMetadata}

    /// Returns the optional resolver contained within this Handler
    ///
    pub fun getResolver(): AnyStruct{LinkedAccountMetadataViews.MetadataResolver}?

    /// Returns the types of Capabilities this Handler has been granted
    ///
    /// @return An array of the Types of Capabilities this resource has access to in its grantedCapabilities
    ///         mapping
    ///
    pub fun getGrantedCapabilityTypes(): [Type]
    
    /// Returns whether the link between this Handler and its associated Collection is still active - in
    /// practice whether the linked Collection has removed this Handler's Capability
    ///
    pub fun isCurrentlyActive(): Bool

    /// Retrieves a granted Capability as a reference or nil if it does not exist.
    /// 
    //  **Note**: This is a temporary solution for Capability auditing & easy revocation 
    /// their way to Cadence, enabling a parent account to issue, audit and easily revoke Capabilities to linked
    /// accounts.
    /// 
    /// @param type: The Type of Capability being requested
    ///
    /// @return A reference to the Capability or nil if a Capability of given Type is not
    ///         available
    ///
    pub fun getGrantedCapabilityAsRef(_ type: Type): &Capability?

    /// Updates this Handler's parentAddress, occurring whenever a corresponding NFT transfer occurs
    ///
    /// @param newAddress: The Address of the new parent account
    ///
    access(contract) fun updateParentAddress(_ newAddress: Address)

    /// Inserts the given Capability into this Handler's grantedCapabilities mapping.
    ///
    /// @param cap: The Capability being granted
    ///
    access(contract) fun grantCapability(_ cap: Capability) {
        pre {
            !self.grantedCapabilities.containsKey(cap.getType()):
                "Already granted Capability of given type!"
        }
    }

    /// Removes the Capability of given Type from this Handler's grantedCapabilities mapping.
    ///
    /// @param type: The Type of Capability to be removed
    ///
    /// @return the removed Capability or nil if it did not exist
    ///
    access(contract) fun revokeCapabilities(_ types: [Type]): [Type]

    /// Removes all granted Capabilities from this Handler's grantedCapabilities mapping. Helpful when removing
    /// a Handler association from a Collection (AKA unlinking an account) as well as limiting the number of events
    /// emitted compared to revoking one-by-one.
    ///
    /// @return An array containing the types of all Capabilities removed
    ///
    access(contract) fun revokeAllCapabilities(): [Type]

    /// Sets the isActive Bool flag to false
    ///
    access(contract) fun setInactive()
}
```
</code>
</details>

<details>
<summary>LinkedAccountMetadataViews</summary>

```js
/// Metadata views relevant to identifying information about linked accounts
/// designed for use in the standard LinkedAccounts contract
///
pub contract LinkedAccountMetadataViews {

    /// Identifies information that could be used to determine the off-chain
    /// associations of a child account
    ///
    pub struct interface AccountMetadata {
        pub let name: String
        pub let description: String
        pub let creationTimestamp: UFix64
        pub let thumbnail: AnyStruct{MetadataViews.File}
        pub let externalURL: MetadataViews.ExternalURL
    }

    /// Simple metadata struct containing the most basic information about a
    /// linked account
    pub struct AccountInfo : AccountMetadata {
        pub let name: String
        pub let description: String
        pub let creationTimestamp: UFix64
        pub let thumbnail: AnyStruct{MetadataViews.File}
        pub let externalURL: MetadataViews.ExternalURL
    }

    /// A struct enabling LinkedAccount.Handler to maintain implementer defined metadata
    /// resolver in conjunction with the default structs above
    ///
    pub struct interface MetadataResolver {
        pub fun getViews(): [Type]
        pub fun resolveView(_ view: Type): AnyStruct{AccountMetadata}?
    }
}
```
</details>

<details>
<summary>NFT</summary>

```js
/** --- NFT --- */
//
/// Publicly accessible Capability for linked account wrapping resource, protecting the wrapped Capabilities
/// from public access via reference as implemented in LinkedAccount.NFT
///
pub resource interface NFTPublic {
    pub let id: UInt64
    pub fun checkAuthAccountCapability(): Bool
    pub fun checkHandlerCapability(): Bool
    pub fun getChildAccountAddress(): Address
    pub fun getParentAccountAddress(): Address
    pub fun getHandlerPublicRef(): &Handler{HandlerPublic}
}

/// Wrapper for the linked account's metadata, AuthAccount, and Handler Capabilities
/// implemented as an NFT
///
pub resource NFT : NFTPublic, NonFungibleToken.INFT, MetadataViews.Resolver {
    pub let id: UInt64
    access(self) var parentAddress: Address
    access(self) let linkedAccountAddress: Address
    /// The AuthAccount Capability for the linked account this NFT represents
    access(self) var authAccountCapability: Capability<&AuthAccount>
    /// Capability for the relevant Handler
    access(self) var handlerCapability: Capability<&Handler>

    /// Function that returns all the Metadata Views implemented by an NFT & by extension the relevant Handler
    ///
    /// @return An array of Types defining the implemented views. This value will be used by developers to know
    ///         which parameter to pass to the resolveView() method.
    ///
    pub fun getViews(): [Type]

    /// Function that resolves a metadata view for this ChildAccount.
    ///
    /// @param view: The Type of the desired view.
    ///
    /// @return A struct representing the requested view.
    ///
    pub fun resolveView(_ view: Type): AnyStruct?

    /// Get a reference to the child AuthAccount object.
    ///
    pub fun getAuthAcctRef(): &AuthAccount

    /// Returns a reference to the Handler
    ///
    pub fun getHandlerRef(): &Handler

    /// Returns whether AuthAccount Capability link is currently active
    ///
    /// @return True if the link is active, false otherwise
    ///
    pub fun checkAuthAccountCapability(): Bool

    /// Returns whether Handler Capability link is currently active
    ///
    /// @return True if the link is active, false otherwise
    ///
    pub fun checkHandlerCapability(): Bool

    /// Returns the child account address this NFT manages a Capability for
    ///
    /// @return the address of the account this NFT has delegated access to
    ///
    pub fun getChildAccountAddress(): Address

    /// Returns the address on the parent side of the account link
    ///
    /// @return the address of the account that has been given delegated access
    ///
    pub fun getParentAccountAddress(): Address

    /// Returns a reference to the Handler as HandlerPublic
    ///
    /// @return a reference to the Handler as HandlerPublic 
    ///
    pub fun getHandlerPublicRef(): &Handler{HandlerPublic}

    /// Updates this NFT's AuthAccount Capability to another for the same account. Useful in the event the
    /// Capability needs to be retargeted
    ///
    /// @param new: The new AuthAccount Capability, but must be for the same account as the current Capability
    ///
    pub fun updateAuthAccountCapability(_ newCap: Capability<&AuthAccount>) {
        pre {
            newCap.check(): "Problem with provided Capability"
            newCap.borrow()!.address == self.linkedAccountAddress:
                "Provided AuthAccount is not for this NFT's associated account Address!"
        }
    }

    /// Updates this NFT's AuthAccount Capability to another for the same account. Useful in the event the
    /// Capability needs to be retargeted
    ///
    /// @param new: The new AuthAccount Capability, but must be for the same account as the current Capability
    ///
    pub fun updateHandlerCapability(_ newCap: Capability<&Handler>) {
        pre {
            newCap.check(): "Problem with provided Capability"
            newCap.borrow()!.address == self.linkedAccountAddress &&
            newCap.address == self.linkedAccountAddress:
                "Provided AuthAccount is not for this NFT's associated account Address!"
        }
    }

    /// Updates this NFT's parent address & the parent address of the associated Handler
    ///
    /// @param newAddress: The address of the new parent account
    ///
    access(contract) fun updateParentAddress(_ newAddress: Address)
}
```
</details>

## Considered For Inclusion

### Events

> The following proposed events are in addition to the AuthAccount Capability linking event `flow.AccountLinked(address: Address, path: PrivatePath)` implemented in the Cadence [AuthAccount Capability linking API](https://github.com/onflow/flips/pull/53#issuecomment-1452777257) whenever an account is linked via `AuthAccount.linkAccount(PrivatePath)`.

As some members of the community have voiced, using a shared standard contract can present challenges to dApps subscibing to events on the contract. Ultimately, how do they know which events are relevant to their users' accounts?

With this in mind, the following events and values have been proposed, though additional feedback is requested as the hope is these events can be as helpful as possible to dApps relying on this shared standard.

- **Linking Accounts & Removing Linked Accounts** - Whenever an account is added as a child of a parent account, an event is emitted denoting both sides of the link. It may be helpful to include values from the child account's `LinkedAccountMetadataViews.AccountMetadata` metadata struct to better identify the linked account.
    ```js
    pub event AddedLinkedAccount(parent: Address, child: Address, nftID: UInt64)
    pub event UpdatedLinkedAccountParentAddress(previousParent: Address, newParent: Address, child: Address)
    pub event UpdatedAuthAccountCapabilityForLinkedAccount(id: UInt64, parent: Address, child: Address)
    pub event RemovedLinkedAccount(parent: Address, child: Address)
    ```

- **Grant/Revoke Capabilities to/from Child Accounts** - Emitted when a parent account grants/revokes a Capability to/from a linked child account.
    ```js
    pub event LinkedAccountGrantedCapability(parent: Address, child: Address, capabilityType: Type)
    pub event RevokedCapabilitiesFromLinkedAccount(parent: Address, child: Address, capabilityTypes: [Type])
    ```

- **Creation of Standard Resources** - Emitted when a `Collection` and `NFT` are created. Since the Collection creation method is a public contract methods, there isn't any data we can include related to the caller. This events likely won't be very useful for dApps or wallet providers, but could be helpful from a data analysis & user behavior perspective to gain insight into the adoption of this standard. The `MintedNFT` event can provide insight into the linking of an account with this standard.
    ```js
    pub event CollectionCreated()
    pub event MintedNFT(id: UInt64, parent: Address, child: Address)
    ```

### NonFungibleToken & MetadataViews Standards Implementation

Since the first iteration was a similar pattern to the NFT standards, and metadata was already considered, was decided to further implement the standards in this contract, making the wrapping resources `NFT: NonFungibleToken.INFT, MetadataViews.Resolver`, and the `Collection: LinkedAccounts.CollectionPublic NonFungibleToken.Collection, NonFungibleToken.CollectionPublic, MetadataViews.ResolverCollection`. This accomplishes two things:

1. Makes it easy for builders to resolve metadata about a user's linked accounts, at least in regards to the account's association/purpose. 
2. Leverages existing standards & mental models familiar to many on Flow, potentially leveraging existing code within dApps to handle a user's linked accounts.

A significant variance in this proposed iteration compared to most NonFungibleToken Collections is the lack of public `deposit()` functionality. This functionality was limited due to concerns around phishing attack vectors. An attacker could create an account they have access to with metadata resembling a victim's account, and deposit it to the victim's Collection. The victim might then inadvertently then transfer NFTs or funds to that account which the attacker could then siphon. Preventing public deposits avoids this altogether, but this variance from the broader NFT standard makes the NFT implementation worthy as a point of discussion.

With regard to the proposed metadata scheme, the metadata was generalized into `LinkedAccountMetadataViews.AccountMetadata` and an additional optional variable of type `LinkedAccountMetadataViews.MetadataResolver` was added to `Handler` to give builders maximum flexibility. This was done so that implementers could create their own metadata structs and decide how to resolve the metadata as best for their use case while still ensuring a minimum set of metadata to be resolved on a linked account. Whether this pattern

### Storage Iteration Convenience Methods

The most simple use case for this is when dealing with a FT that multiple child accounts hold, or NFTs from the same collection spread among different child accounts. Currently, a dApp would solve this by iterating over each child account's storage either in separate or one large script (the former being more scalable over large number of stored items):

<details>
<summary>Example Account + Storage Iteration Script to get child account balances</summary>

```js
import FungibleToken from "../../contracts/utility/FungibleToken.cdc"
import MetadataViews from "../../contracts/utility/MetadataViews.cdc"
import FungibleTokenMetadataViews from "../../contracts/utility/FungibleTokenMetadataViews.cdc"
import LinkedAccounts from "../../contracts/LinkedAccounts.cdc"

/// Custom struct to easily communicate vault data to a client
pub struct VaultInfo {
    pub let name: String?
    pub let symbol: String?
    pub var balance: UFix64
    pub let description: String?
    pub let externalURL: String?
    pub let logos: MetadataViews.Medias?
    pub let storagePathIdentifier: String
    pub let receiverPathIdentifier: String?
    pub let providerPathIdentifier: String?

    init(
        name: String?,
        symbol: String?,
        balance: UFix64,
        description: String?,
        externalURL: String?,
        logos: MetadataViews.Medias?,
        storagePathIdentifier: String,
        receiverPathIdentifier: String?,
        providerPathIdentifier: String?
    ) {
        self.name = name
        self.symbol = symbol
        self.balance = balance
        self.description = description
        self.externalURL = externalURL
        self.logos = logos
        self.storagePathIdentifier = storagePathIdentifier
        self.receiverPathIdentifier = receiverPathIdentifier
        self.providerPathIdentifier = providerPathIdentifier
    }

    pub fun addBalance(_ addition: UFix64) {
        self.balance = self.balance + addition
    }
}

/// Returns a dictionary of VaultInfo indexed on the Type of Vault
pub fun getAllVaultInfoInAddressStorage(_ address: Address): {Type: VaultInfo} {
    // Get the account
    let account: AuthAccount = getAuthAccount(address)
    // Init for return value
    let balances: {Type: VaultInfo} = {}
    // Assign the type we'll need
    let vaultType: Type = Type<@{FungibleToken.Balance, MetadataViews.Resolver}>()
    let ftViewType: Type= Type<FungibleTokenMetadataViews.FTView>()
    // Iterate over all stored items & get the path if the type is what we're looking for
    account.forEachStored(fun (path: StoragePath, type: Type): Bool {
        if type.isSubtype(of: vaultType) {
            // Get a reference to the vault & its balance
            if let vaultRef = account.borrow<&{FungibleToken.Balance, MetadataViews.Resolver}>(from: path) {
                let balance = vaultRef.balance
                // Attempt to resolve metadata on the vault
                if let ftView = vaultRef.resolveView(ftViewType) as! FungibleTokenMetadataViews.FTView? {
                    // Insert a new info struct if it's the first time we've seen the vault type
                    if !balances.containsKey(type) {
                        let vaultInfo = VaultInfo(
                            name: ftView.ftDisplay?.name ?? vaultRef.getType().identifier,
                            symbol: ftView.ftDisplay?.symbol,
                            balance: balance,
                            description: ftView.ftDisplay?.description,
                            externalURL: ftView.ftDisplay?.externalURL?.url,
                            logos: ftView.ftDisplay?.logos,
                            storagePathIdentifier: path.toString(),
                            receiverPathIdentifier: ftView.ftVaultData?.receiverPath?.toString(),
                            providerPathIdentifier: ftView.ftVaultData?.providerPath?.toString()
                        )
                        balances.insert(key: type, vaultInfo)
                    } else {
                        // Otherwise just update the balance of the vault (unlikely we'll see the same type twice in
                        // the same account, but we want to cover the case)
                        balances[type]!.addBalance(balance)
                    }
                }
            }
        }
        return true
    })
    return balances
}

/// Takes two dictionaries containing VaultInfo structs indexed on the type of vault they represent &
/// returns a single dictionary containg the summed balance of each respective vault type
pub fun merge(_ d1: {Type: VaultInfo}, _ d2: {Type: VaultInfo}): {Type: VaultInfo} {
    for type in d1.keys {
        if d2.containsKey(type) {
            d1[type]!.addBalance(d2[type]!.balance)
        }
    }

    return d1
}

pub fun main(address: Address): {Type: VaultInfo} {
    // Get the balance info for the given address
    var balances: {Type: VaultInfo} = getAllVaultInfoInAddressStorage(address)
    
    /* Iterate over any child accounts */ 
    //
    // Get reference to LinkedAccounts.CollectionPublic if it exists
    if let collectionRef = getAccount(address).getCapability<
            &LinkedAccounts.Collection{LinkedAccounts.CollectionPublic}
        >(
            LinkedAccounts.CollectionPublicPath
        ).borrow() {
        // Iterate over each linked child account in Collection
        for childAddress in collectionRef.getLinkedAccountAddresses() {
            // Ensure all vault type balances are pooled across all addresses
            balances = merge(balances, getAllVaultInfoInAddressStorage(childAddress))
        }
    }
    return balances 
}
 
```
</details>

But we could include this functionality in the contract in some different ways. For example, we could add to `Collection` the following methods:

```js
pub fun getChildBalances(tokenPath: PublicPath): {Address: UFix64}

pub fun withdrawFromChild(tokenPath: PublicPath, amount: UInt64, _ children: Address?): @FungibleToken.Vault

_____________

pub fun getChildNFTIDs(tokenPath: PublicPath): {Address: [UInt64]}

pub fun withdrawChildNFT(tokenPath: PublicPath, tokenID: UInt64, child: Address): @NonFungibleToken.NFT
```

Those convenience methods would allow vaults or collections to easily accessible across all child accounts from one parent at the same time. However, iteration over a large number of accounts and/or unexpected path naming conventions within those accounts might lead to unexpected behavior and encourage reliance on iteration at the script layer anyway.

# Drawbacks

AuthAccount Capability is a powerful tool that can be dangerous if it is not used properly. Standardizing how these Capabilities are managed should not have any negative impact on Flow’s ecosystem. Considering there have already been discussions about enhancing the security of potentially dangerous actions with a [sudo-like transaction](https://forum.onflow.org/t/super-user-account/4088), and increasing the auditability and control of Capabilities with [Capability Controllers (CapCons)](https://github.com/onflow/flow/pull/798/files?short_path=f2770e8#diff-f2770e8e35eaed0f7ffa91d366e50cb08f0e18d363c0c5543774d11d7656c8a9), this Flip does not introduce any new attack vectors into the ecosystem.

## Considerations

### Visibility into All Sub-Account Storage

Queries involving storage iteration over a large number of accounts might encounter memory limits either by iteration over too large a number of addresses, too many items in storage, or some combination of the two. This is noted not as a requirement to solve, but as a design consideration in how we communicate an account’s sub-accounts and whether we also include resource/contract methods to easily query the assets in sub-accounts.

### Sources of Truth

Another consideration is preserving the managing resource and tag as accurate sources of truth for the status of account linking. Just as we rely on keys to determine access to an account, so too should we be able to determine from a parent account if it has a capability for another account. Inversely, from a child account we should be able to determine if another account has delegated authority and the address of that account.

### Auditability and Revocation

If a user wanted to revoke secondary access to one of their linked accounts, they’d have to check two things. First, they’d want to ensure that only the key they custody has access to the linked account. Easily enough, they’d revoke any keys on the account that they do not custody. Second, they’d want to know who else has delegated access via AuthAccount Capabilities. This is not so straightforward, at least not until [Capability Controllers](https://github.com/onflow/flow/pull/798) join the picture.

Ideally, a user could look at the AuthAccount Capability they have and see at minimum if any other parties have been issued a Capability from the same CapabilityPath. If so, they could easily revoke that Capability thereby removing the holding party’s access to the linked account. Alternatively, the user could retarget their Capability to an alternative CapabilityPath and unlink the original, also breaking the secondary party’s access.

Currently, however, a user cannot know who else has been issued an AuthAccount Capability linked at the same path as their held Capability. While a user could unlink any AuthAccount Capabilities at other Capability paths, they can not be guaranteed that no one else has access via the same CapabilityPath as the Capability they hold. If a user wanted to ensure no one else had access, they would unlink the AuthAccount Capability altogether, thereby removing their own access to the child account altogether. 

As such, it’s recommended that users forego revocation in favor of abandoning a child account altogether, at least until CapCons enable more granular control over Capabilities at large.

### Limiting Delayed Attack Vectors

When it comes to accessing a user’s saved AuthAccount Capabilities, it is possible to restrict Capabilities to retrieval by reference - `&AuthAccount` instead of `Capability<&AuthAccount>`. However, in capability-based access, such restrictions on an issued Capability might be considered an anti-pattern.

With that said, signing a malicious transaction today means you are at risk within the scope of that transaction. Signing a malicious transaction in a world of AuthAccount Capabilities means a bad actor could issue themselves a Capability on your account or one of your child accounts to perform their attack at a later time.

One way to prevent this is to make accessing issued AuthAccount Capabilities ephemeral, limiting the scope of the attack to the time scope of the transaction. Another is to rely on events emitted whenever an AuthAccount Capability is retrieved from the `Collection`. Yet another measure would include emitting an event any time an AuthAccount Capability is linked.

```js
// Let's say a parent account is signing a transaction in which a reference to
// a Collection is retrieved
let collectionRef: &LinkedAccounts.Collection = parent.borrow<&LinkedAccounts.Collection>(
		from: LinkedAccounts.CollectionStoragePath
	)!

// In the current prototype definitions of Collection
// we return a reference to an auth account which is ephemeral
// in that it cannot be stored. The worst a malicious transaction could
// do is limited in scope to within the transaction being signed
let childAccountRef: &AuthAccount? = collectionRef.getChildAccountRef(
		address: 0x02
	)

// Alternatively, we could just return the Capability to the AuthAccount.
// With this, a malicious transaction could publish the capability for themselves,
// retrieve it and store it for later use. This expands at least the time scope
// of the attack and is difficult to audit. 
let childAccountCap: Capability<&AuthAccount> = collectionRef.getChildAccountCap(
		address: 0x02
	)
```

Taken together, these measures enable wallet providers to at least notify relevant user’s when any of their accounts trigger an AuthAccount Capability-related event. Such a flow would be similar to the notification you receive from a Web2 identity provider whenever you authorize a new app (e.g. sign in with Google to DapperLabs and Google will let you know you linked your accounts).

### Lack of Ultimate Control

Ideally, this standard would place ultimate authority over the child account in the hands of the parent account. Think of the child account as a joint bank account with the user as the superceding owner - they could issue and revoke capabilities on the joint account, but no one could revoke their access.

Currently, a dApp with shared access to the child account could unlink the AuthAccount Capability removing the parent account’s access to it. If the child account was created by the user and a user-custodied key was added to the child account, the dApp could even revoke the associated keys on the account.

The [SuperAuthAccount proposal](https://forum.onflow.org/t/super-user-account/4088/2?u=gio_on_flow) mentioned earlier removes the possibility of a secondary party revoking key access if they do not also custody a signing private key. Still, if the dApp custodies the private key, the possibility remains that the user can lose access to the child account.

While this custodial risk exists, hybrid custody accounts are by no means intended to store valuable assets and would refer builders to traditional custodial models more suitable for high-value assets. It might be worth informing users that their in-app accounts have shared access, and recommending that the resources they deem valuable be transferred to their wallet-mediated parent accounts.

# **Alternatives Considered**

## Key-Based Child Accounts

For reference, see this implementation of  [key-based child accounts](https://github.com/onflow/sc-eng-gaming/blob/sisyphusSmiling/child-account/contracts/ChildAccount.cdc#L145)

In this construction, account access is delegated by adding a parent account’s PublicKey (1000 weight) to a child account. This gives both the parent and the holder of the originating public key’s paired private key full signing authority on the child account.

Simply using keys to delegate access is the most often suggested alternative to AuthAccount Capabilities for delegated access. While this would technically result in shared access, this approach creates a number of secondary problems without straightforward solutions.

In the scenario that a main account key is added to a linked account, consider the following:

1. What if I want to switch wallets & bring linked accounts with me?
2. What if I want to bring my linked accounts across devices namely mobile?
3. If I need to re-key my main account, do I need to also re-key all linked accounts?

On 1/ - delegated access should be portable across wallets & knowledge of linked accounts should be available to both wallets & dApps. Establishing the association on-chain solves the problem of global knowledge.

On 2/ Again, linked accounts should be available anywhere the user has access to the parent account, meaning cross-device & potentially across different wallets. If we rely on a self-custodied key local to a device, we lose that interoperability.

On 3/ In this key-based approach key, re-keying my main account means I need to re-key all linked accounts. If I have separate keys for each linked account, how do I manage those keys?

This approach, while technically feasible, implies considerations that are more than just technical - with a dependency on wallet providers to implement, how do we make the business case that this is worth implementing?

And if we convince one, that still doesn’t get Flow an ecosystem-wide account linking feature. Implementing at the contract level, standardizing the construct, then telling all wallet providers & dApps to use this thing to support hybrid custody & walletless onboarding is way more impactful.

Even if a key-based approach is decided in favor of account delegation by AuthAccount Capabilities, there remains the issue of representing the association on-chain so the link is portable, meaning a common standard is still required.

## Native Notion of Child Accounts

In this approach, the `Collection` and `Handler` would be a construct native to all AuthAccounts - `AuthAccount.childAccounts` and `AuthAccount.parentAccounts` respectively. The idea is the route to issue delegated AuthAccount Capabilities would be more direct, and all access would be restricted to ephemeral references to prevent storage by unauthorized parties. A byproduct of this is more granular access control and auditability of delegated access, leading us to believe this could be a viable alternative.

However, this approach felt too prescriptive about the use of AuthAccounts and attempts to mitigate security concerns felt incomplete. Additionally, we also wanted to move quickly in prototyping, and doing so at the contract level enabled us to experiment and iterate much faster than if we attempted to introduce a language level construct from the start.

In summary, on the one hand, this feels more secure. On the other hand, if we still allow linking AuthAccount like any other Capability, this is like putting up a fence you can simply walk around. If we choose to continue down this path, we should also reconsider implementation approach to make account linking via this route more direct. In other words, linking AuthAccount Capabilities should only be possible via `AuthAccount.ParentAccounts` and accessible ephemerally via `AuthAccount.ChildAccounts` (or similarly named AuthAccount objects).

![cadence_api_child_accounts](./20230223-auth-account-capability-management-standard-resources/cadence_api_child_accounts.png)

# Best Practices

As this Flip introduces a standard contract and set of resources, best practices around using them are centered around:

1. Creating and funding child accounts
2. Linking existing accounts as a user’s child account
3. Revoking a child account
4. [Seeking input] Granting/revoking a child account Capabilities from the parent account

## Creating and Funding Child Accounts

Aside from implementing onboarding flows & account linking, you'll want to also consider the account funding & custodial pattern appropriate for the dApp you're building. The only one compatible with walletless onboarding is one in which the dApp custodies the child account's key, funds account creation. This can be done any way you'd normally create a self-funded account on Flow.

In general, the funding pattern for account creation will determine to some extent the backend infrastructure needed to support a dApp and the onboarding flow a dApp can support. For example, if you want to to create a service-less client (a totally local dApp without backend infrastructure), you could forego walletless onboarding in favor of a user-funded blockchain-native onboarding to achieve a hybrid custody model. Your dApp maintains the keys to the dApp account to sign on behalf of the user, and the user funds the creation of the the account, linking to their main account on account creation. This would be a **user-funded, dApp custodied** pattern.

Here are the patterns for consideration:

### DApp-Funded, DApp-Custodied

This is the only pattern compatible with walletless onboarding. In this scenario, a backend dApp account funds the creation of a new account and the dApp custodies the key for said account either on the user's device or some backend KMS.

### DApp-Funded, User-Custodied

In this case, the backend dApp account funds account creation, but adds a key to the account which the user custodies. In order for the dApp to act on the user's behalf, it has to be delegated access via AuthAccount Capability which the backend dApp account would maintain in a `Collection`. This means that the new account would have two parent accounts - the user's and the dApp. While not comparatively useful now, once `SuperAuthAccount` is ironed out and implemented, this pattern will be the most secure in that the custodying user will have ultimate authority over the child account. Also note that this and the following patterns are incompatible with walletless onboarding in that the user must have an existing wallet.

### User-Funded, DApp-Custodied

As mentioned above, this pattern unlocks totally service-less architectures - just a local client & smart contracts. An authenticated user signs a transaction creating an account, adding the key provided by the client, and linking the account as a child account. At the end of the transaction, hybrid custody is achieved and the dApp can sign with the custodied key on the user's behalf using the newly created account.

### User-Funded, User-Custodied

While perhaps not useful for most dApps, this pattern may be desirable for advanced users who wish to create a shared access account themselves. The user funds account creation, adding keys they custody, and delegates secondary access to some other account. As covered above in account linking, this can be done via multisig or the publish & claim mechanism.

## Linking existing account as child account

Given the idea of progressive onboarding, we’ll need to add existing accounts as child accounts. This can be done either by a multisig transaction by the parent & child account, or via the `AuthAccount.Inbox` methods `publish()` and `claim()` across two transactions.

The parent can the take the child account’s AuthAccount Capability & call `Collection.addAsChildAccount()` to create the on-chain link between accounts & preserve the child account’s new `Handler` & `AuthAccount` Capabilities.

For more on this process, see [this example](#adding-an-account-as-a-child-account-aka-account-linking).

## Revoking a child account

Revocation in this construct involves removing the `NFT` from the parent’s `Collection`. This can be done via `Collection.removeLinkedAccount()` method. However, this is only one side of revocation.

The other sort of revocation a user might want to accomplish is taking full control of the child account & removing secondary party access. In this case, there are two things we want to check:

1. Key access
2. AuthAccount Capability access

With regards to key access, if a developer account custodied the keys, the user will want to revoke them. In addition, a user will likely want to add a key they can access to their child account. If the user custodied keys for the child account, then this step can be skipped.

With regards to Capabilities on the child account, a user will want to unlink any undesired AuthAccount Capabilities from within their child account.

## Granting a child account Capabilities

This is more of an open question we’re hoping for more input on. On the one hand, CapCons make it much easier to audit and revoke Capabilities. On the other hand, the ability to easily pass Capabilities to a child account via the `Collection` → `Handler` mechanism means the a user has granular control over the Capabilities their child accounts have access to. 

This is especially useful in the case of gaming for instance, where a user may maintain a Capability to their global gamer identity. This resource can be saved and linked at a single path in their main account and a private Capability issued to each of the child accounts used across a number of different game clients. This would allow for full interoperability across all platforms and granular control over each client’s access to the central gamer identity resource. As you can see, the granted capabilities are accessible only by reference in `Handler.getGrantedCapabilityAsRef()` and can be removed from child accounts via `Collection.removeCapability()`.

# **Tutorials and Examples**

Assuming this FLIP is approved/implemented formal tutorials and examples will be provided before launch. Examples provided below are illustrative and for alignment purposes, and may be subject to change.

## Adding an Account as a Child Account (AKA “Account Linking”)
> ℹ️ Can be multisigned transaction by both parties or AuthAccount capability can be linked & published by child account then claimed and linked in the parent accounts Collection

![child_account_add_as_child](./20230223-auth-account-capability-management-standard-resources/child_account_add_as_child.jpg)

> :warning: Note, this diagram and steps below will be updated shortly to reflect the latest iteration of this proposed standard.

In this diagram, the parent account signs a transaction doing the following:

1. Claim the AuthAccount Capability published for them by the child account
2. Gets a reference to their ChildAccountManager and passes the claimed Capability along with child account metadata, `ChildAccountInfo`
3. Creates a ChildAccountTag in the child account if one doesn’t already exist, providing the given `ChildAccountInfo` on creation. Again, this is possible because the parent now has an AuthAccount Capability for the child account.
4. Links the `ChildAccountTag` in as a private Capability in the child account
5. Retrieves the linked `ChildAccountTag` Capability from the child account
6. Creates a `ChildAccountController`, providing the claimed AuthAccount Capability as well as the `ChildAccountTag` Capability

## Using a Child Account’s FlowToken Vault

> :warning: Note, this diagram will be updated shortly to reflect the latest iteration of this proposed standard.

![child_account_use_child_account_vault](./20230223-auth-account-capability-management-standard-resources/child_account_use_child_account_vault.jpg)

```js
import FungibleToken from "../../contracts/utility/FungibleToken.cdc"
import FlowToken from "../../contracts/FlowToken.cdc"
import LinkedAccounts from "../../contracts/LinkedAccounts.cdc"

transaction(fundingChildAddress: Address, withdrawAmount: UFix64) {

    let paymentVault: @FungibleToken.Vault

    prepare(signer: AuthAccount) {
        // Get a reference to the signer's LinkedAccounts.Collection from storage
        let collectionRef: &LinkedAccounts.Collection = signer.borrow<&LinkedAccounts.Collection>(
                from: LinkedAccounts.CollectionStoragePath
            ) ?? panic("Could not borrow reference to LinkedAccounts.Collection in signer's account at expected path!")
        // Borrow a reference to the signer's specified child account
        let childAccount: &AuthAccount = collectionRef.getChildAccountRef(address: fundingChildAddress)
            ?? panic("Signer does not have access to specified account")
        // Get a reference to the child account's FlowToken Vault
        let vaultRef: &TicketToken.Vault = childAccount.borrow<&FlowToken.Vault>(
                from: /storage/flowToken
            ) ?? panic("Could not borrow a reference to the child account's TicketToken Vault at expected path!")
        self.paymentVault <-vaultRef.withdraw(amount: withdrawAmount)
    }

    execute {
      // Do stuff with the vault...(e.g. mint NFT)
    }
}
```

In the above example, an authenticated user signs a transaction that retrieves a Provider on a FlowToken Vault stored in one of their child accounts. This can of course be any denomination Vault, allowing a user to sign a transaction with the parent account and withdraw funds for use in a transaction without needing to first transfer funds to the signing account.

# Compatibility

The standards being discussed in this Flip are additive, and thus do not imply any issues with backwards compatibility.

# User Impact

Our hope is that if/when this Flip is adopted, this feature will be useful for a number of different use cases. We can see child accounts being especially useful for gaming and perhaps even as a new airdrop mechanism, allowing creators to conduct airdrops tied to Web2 identities. As such, we’re hoping to provide as many examples and supporting scripts & transactions we can think of to facilitate progressive onboarding as well as account + storage iteration to get Flow builders started implementing this standard into their projects where they see fit.

# Questions and Discussion Topics

## Verbiage

While the “parent-child” name implies an account hierarchy, it doesn’t necessarily map to the hybrid custody model that this standard works to support. There are number of other ideas for this hierarchical linked account relationship that could serve as better alternatives we’re hoping for input and consensus on. Share your opinion and provide others if you think there’s a better fit.

- Parent-child account
- Main-secondary account
- Principal-proxy account
- Main-shared account
- Core-satellite account
- Primary-partition account
- Private-joint account
- Puppeteer-puppet account
- Hybrid custody account
- Linked Accounts

## Open Questions

- Are there any additional events and/or event data that should be included in this standard?
- Is the limitation of public `deposit()` functionality fly too far away from the NFT standard such that its use is not warranted?
- How will the newly introduced [SuperAuthAccount](https://forum.onflow.org/t/super-user-account/4088/2) feature fit in? Will we want to delegate and store `SuperAuthAccount` in the parent’s managing resource or should parent accounts only preserve AuthAccount? My vote is to give parent accounts the fullest permissions on child accounts so they can add/revoke keys, etc.
- Where do Capability Controllers fit in and can they reduce some of the concerns around auditing and revocation of AuthAccount Capabilities?
- Should ChildAccountTags be allowed to have multiple parent accounts. I believe they should. One use case I can imagine is gaming - I might want all of my game client accounts to have access to each other so I can easily transfer between them and for full interoperability between platforms.
    - `parentAddresses: [Address]`
- Do we want `Collection` to be able to revoke other `Collections`s’ access to a child account? For example, let’s say I have access to a set of child accounts and I want to remove anyone else’s access to those accounts. How can we ensure the user has that ability - AuthAccount CapabilityPath naming conventions, CapCons, in-built functions in `Collection`?
    - This can be helpful in the case that a user delegates child account access to another account - a user’s mobile game client account delegated to their web game client account for instance so assets are visible and accessible between the two without friction.
    - Alternatively, AuthAccount Capabilities on a child account can be handled the same as any Capability - unlink from a designated path and via CapCons in the future.
- Is the proposed metadata in `LinkedAccountMetadataViews` and its use in `Handler` sufficient?
    - Is there any other metadata we would like to ensure is added?
- Linking AuthAccounts is possible outside of the mechanisms defined in this standard. How does a user know who else has secondary access to their child accounts.
    - Example: A user might think they have revoked secondary access to a child account by revoking the originating public key; however, how can the user be assured that no one else has AuthAccount Capabilities. Additionally, how should the user unlink the existing AuthAccount Capabilities without also removing their access which is also delegated via Capability.
    - This is possibly the strongest case for baking this construction into AuthAccounts natively and making linking & Capability access available only through direct routes. Alternatively, child AuthAccounts might be encapsulated by a wrapping API layer that would prevent direct access entirely.

# References

- [Preceding Forum Post](https://forum.onflow.org/t/account-linking-authaccount-capabilities-management/4314)
- [Walletless Onboarding Blog Post](https://flow.com/post/flow-blockchain-mainstream-adoption-easy-onboarding-wallets)
- [Walletless Onboarding Presentation](https://youtu.be/0eYX_S4jUYM)
- [AuthAccount Capability Cadence implementation](https://github.com/onflow/cadence/issues/2151)
- [AuthAccount Capability Flip](https://github.com/onflow/flips/pull/53)
- [Hybrid Custody forum post](https://forum.onflow.org/t/hybrid-custody/4016/8)
- [Gaming + Child Account contract implementation](https://github.com/onflow/sc-eng-gaming/tree/sisyphusSmiling/child-account-auth-acct-cap)
- [Child Account dApp implementation](https://github.com/onflow/flow-games-retro)
- [Super User AuthAccount Forum Post](https://forum.onflow.org/t/super-user-account/4088/2?u=gio_on_flow)