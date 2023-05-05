---
status: Proposed 
flip: 72
title: AuthAccount Capability Management
forum: https://forum.onflow.org/t/account-linking-authaccount-capabilities-management/4314
authors: Giovanni Sanchez (giovanni.sanchez@dapperlabs.com), Austin Kline (austin@flowty.io), Deniz Edincik (deniz@edincik.com)
editor: Jerome Pimmel (jerome.pimmel@dapperlabs.com)
updated: 2023-02-23
---

# AuthAccount Capability Management

<details>
<summary>Table of Contents</summary>

- [Context](#context)
- [Objective](#objective)
    - [Non-goals](#non-goals)
    - [Goals](#essential)
- [Ongoing Work](#ongoing-work)
    - [Previous Work](#previous-work)
    - [Proposal Updates](#proposal-updates)
- [Motivation](#motivation)
- [FAQ](#faq)
- [User Benefit](#user-benefit)
- [Design Proposal](#design-proposal)
    - [Bird's-Eye View](#birds-eye-view)
    - [Lower-Level Details](#lower-level-details)
        - [HybridCustody](#hybridcustody)
        - [CapabilityFactory](#capabilityfactory)
        - [CapabilityFilter](#capabilityfilter)
        - [CapabilityProxy](#capabilityproxy)
    - [Example Implementation](#example-implementation)
    - [Considered For Inclusion](#considered-for-inclusion)
        - [Events](#events)
        - [MetadataViews Standards Implementation](#metadataviews-standards-implementation)
        - [Storage Iteration Convenience Methods](#storage-iteration-convenience-methods)
        - [Discoverability](#discoverability)
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

## Goals

This FLIP proposes a standard for the creation and management of child accounts to support  progressive onboarding models. The standard defines the resource representing the parent-child relationship between accounts, identifies an account as a parent and/or child, and related management functions including:

- Child account creation
- Child AuthAccount capability management
- Viewing existing child accounts
- Adding an account as a child account
- Revoking hybrid custody approval/access granted to a child account
- Identifying an account’s child accounts
- Identifying an account’s parent account(s)
- Implement useful events builders can rely on

# Ongoing Work

Community-led efforts to collaboratively iterate to a final solution that works for all parties end-to-end can be found in [flowtyio/restricted-child-accounts](https://github.com/Flowtyio/restricted-child-account). This FLIP has been updated to reflect the latest design raised by [@austinkline](https://github.com/austinkline) in [this forum post](https://forum.onflow.org/t/hybrid-custody-restrictions-on-linked-accounts/4561) to address a number of concerns with the original design put forth in this FLIP.

## Previous Work

Below are contract and dApp implementations that were used as sandboxes for on-chain gaming, native attachments, progressive onboarding and eventually culminated in the development of this FLIP:

- [Simplified linked accounts Cadence](https://github.com/onflow/linked-accounts)
- [Linked accounts Cadence in context](https://github.com/onflow/sc-eng-gaming/tree/sisyphusSmiling/child-account-auth-acct-cap)
- [Walletless onboarding dApp example](https://github.com/onflow/walletless-arcade-example)
- [v1 Testnet contract deployment](https://f.dnz.dev/0x1b655847a90e644a/ChildAccount)
- [v2 Testnet contract deployment](https://f.dnz.dev/0x1b655847a90e644a/LinkedAccounts)

## Proposal Updates

- **05.05.23**: Major design change covering new HybridCustody + supporting contracts, introducing the ability for app developers to define and encapsulate retrictions on the access given to parent accounts.

# Motivation

With Hybrid Custody as an ecosystem-wide feature, we can imagine that a user can have any number of child accounts tied to their main account - one for each dApp they’re using. They will need to maintain AuthAccount capabilities for each, add existing accounts as child accounts, use child account capabilities and easily manage assets across their sub-accounts without the need to first transfer assets to their main account.

For example, a user might have a child account associated with a game dApp containing an NFT and in-game FungibleTokens. They should be able to sign into an NFT marketplace and view the NFTs across all of their linked accounts. Then, without needing to first transfer the NFT to the signing account, list that NFT for sale or purchase an NFT with the FungibleTokens in their child account, signing as the parent account. Similarly, if an offer is submitted on an NFT in a child account, a wallet provider managing the parent or dApp in which the parent accounts is authenticated (i.e. a marketplace) should be able to notify the owner and allow them to accept the offer, again signing as the parent account.

To put it simply, this standard sets down primitives through which well-known web2 account-to-application authorization schemes can be modeled in our decentralized context.

Accomplishing this vision successfully - success here meaning building a secure and universally useful construct that serves as solid ground truth while satisfying the aforementioned objectives - can further distinguish Flow among other chains as a platform for builders to create totally new dApps not possible elsewhere.

# FAQ

- I’m concerned my main account can get adopted by another Flow account, how is that prevented?
    - Issuing Capabilities on an AuthAccount pose the same sort of vulnerability vector as adding keys onto an account. Since these features are in a similar class - that of delegated authority - the community has worked to introduce a new [“Super User Account” feature](https://forum.onflow.org/t/super-user-account/4088), similar to `sudo` in Linux systems. This means issuing a Capability on your AuthAccount, as well as adding keys, deploying/deleting contracts, and other potentially dangerous operations, can only occur in transactions for which you have given explicit sudo-like permission for. Note that the concept is not the focus of this Flip, and is still in discussion meaning the exact construct will likely evolve.
- Why would I want someone else to have access to my account?
    - To clarify, the design isn’t for you to give a dApp access to your main account. The dApp creates an account it maintains access to that it uses to interact with Flow. Then, when you’re ready to access the dApp’s account more directly, the dApp delegates access to your main account, thereby linking its account as a child of your main account. This lets the dApp act on your behalf for only the assets in the child account. Everything in your main account remains only in your control, while allowing you to act on the assets in the child account without the need for the dApp to mediate.
- As a user, why would I want a separate account I share access with? Doesn’t that put all of my assets at custodial risk?
    - Again, only the child account shares access with another party, meaning your main account is safe from custodial risk. In fact, partitioning assets across accounts in this way enhances security over a model that requires all transactions be signed by your main account. A user can keep all of their more valuable assets in their main account, out of reach without a user-signed transaction, while keeping less valuable dApp assets in a shared account for ease of use.
- As an application developer, won't I expose myself to undo risk by giving a user access on an account I custody?
    - The newly proposed design introduces the ability to restrict delegated access. This means that you can set the rules on what a user can access via the delegation you grant them, thereby setting their scope as you define it. For example, want users to be able to access an NFT Collection in your app-custodied account? That can be easily configured!

# User Benefit

A standard resource managing all child account capabilities allows for a unified user experience, enabling any dApp in the ecosystem to connect a user’s primary wallet and get global context into the entirety of the user’s accounts and assets that would otherwise be fragmented and unknown. For the end user, this means:

- Greater organization of their app accounts and assets across them
- Reduced risk of forgetting about assets in secondary accounts
- Reduced custodial risk to assets in their main account due to partitioned access between parent and child accounts
- Fewer transfers needed to engage with assets in child accounts as connected dApps can leverage the standard child account managing resource to utilize capabilities across multiple linked accounts within a single signed transaction
- Unified asset balance views across all of their accounts (e.g. Alice has 3 accounts with TokenA Vaults, but only needs to know the sum of all their balances; Alice has 3 accounts with NFTs but is only interested in viewing her total collection overview on a marketplace profile page)

And for builders:

- Clear expectations on where child account capabilities will reside and how to engage with them
- The ability to create self-custodial dApps leveraging web2 onboarding flows with the added flexibility of delegating shared control of an app account to a user in the future
- Newly included ability to define and set restrictions on the Capabilities a parent account has access to on an account your app custodies
- Easily tap into an ecosystem-wide differentiating feature not possible anywhere else in the industry

# Design Proposal

## Bird's-Eye View

![restricted_child_accounts_high_level](./20230223-auth-account-capability-management-standard-resources/restricted_child_accounts_high_level.png)

At a high level, a user will have delegated access on an app-custodied account - access mediated by resources which encapsulate developer-defined and instantiated rules, regulating user access on the account to that which the developer has allowed.

The notion of restricting access emerged out of a need to conform with existing custodial infrastructure in a manner that reduces regulatory risk for developers, thereby reducing the burden of implementation. Prior iterations did not adequately account for the issues created for developers and custodian services by sharing unrestricted access on app accounts.

As we'll see below, the promise of real ownership enabled by Hybrid Custody is still realized in that a user can access assets in their child accounts; however, this is now possible without outsized burden and risk on the implementing applications. Additionally, this new approach accounts for a path to "promotion", allowing for users to take control of their shared access accounts while preserving developer safety and expectations.

## Lower-Level Details

![restricted_child_accounts_low_level](./20230223-auth-account-capability-management-standard-resources/restricted_child_accounts_low_level.png)
*This diagram depicts the composition of resources and Capabilities involved in this design*

Taking a look at the [current implementation](https://github.com/Flowtyio/restricted-child-account), you'll find the following constructs:

### HybridCustody

- `Manager` - A resource associated with a parent account that will allow its owner to store and access Capabilities which mediate access on child accounts. These can be Capabilities with restrictions (`accounts`) or without restrictions (`ownedAccounts`), mapping to Capabilities on `ProxyAccount` and `ChildAccount` respectively. Enables the addition of delegated Capabilities, retrieval of Capabilities in child accounts, and querying a `Manager`'s child accounts. A manager can also set a filter on the Capabilities it can be granted from a child account which can be helpful if a user is managing child accounts from a custodial wallet that wants to prevent access on specific Capabilities (e.g. `FungibleToken.Vault`).
- `ProxyAccount` - This resource lives in a child account and maintains a Capability on a `ChildAccount` along with other Capabilities that regulate Capability retrieval from said `ChildAccount`. This resource also serves as a pointer to its related "parent" account who holds a Capability on it from a `Manager` resource.
- `ChildAccount` - This resource also lives in a child account, encapsulating an AuthAccount Capability on the account in which it's stored. It also maintains state on the parent accounts that have delegated access, its current owner, as well as functionality that enables the owner to revoke parent access and grant account ownership by delegating a Capability on itself to said owner.

### CapabilityFactory

- `Manager` - Motivated by the need to retrieve typed Capabilities from child accounts, this resource maintains `Factory` structs which retrieve Capabilities that can be cast by the caller.

### CapabilityFilter

- `DenylistFilter` - Types defined in this filter are those that should not be accessible. This and all following `Filter` implementations link Capabilities that are stored in a `ProxyAccount` as a, well filter, on the Capability Types accessible from a child account.
- `AllowlistFilter` - Types defined in this filter are those which are accessible
- `AllowAllFilter` - Can be counted on to allow all types of capabilities

### CapabilityProxy

- `Proxy` - Tasked with storing and retrieving arbitrary Capabilities and intended to be segmented similar to current public & public models for Capability access via `GetterPublic` and `GetterPrivate` interfaces respectively. This resource in stored in a child account and a Capability on it is maintained by a `ProxyAccount`.

## Example Implementation

![restricted_child_accounts_overview](./20230223-auth-account-capability-management-standard-resources/restricted_child_accounts_overview.png)
*Above you'll find the composition of resources and Capabilities along with relevant interfaces*

The constructs listed above have been prototyped and interface references are available for reference below. Development is ongoing and up-to-date implementation can be found in [this repo](https://github.com/Flowtyio/restricted-child-account).

<details>
<summary>Manager</summary>

```cadence
/// Entry point for a parent to borrow its child account and obtain capabilities or
/// perform other actions on the child account
pub resource interface ManagerPrivate {
    pub fun borrowAccount(addr: Address): &{AccountPrivate, AccountPublic}?
    pub fun removeChild(addr: Address)
    pub fun removeOwned(addr: Address)
    pub fun borrowOwnedAccount(addr: Address): &{Account, ChildAccountPublic, ChildAccountPrivate}?
    pub fun setManagerCapabilityFilter(cap: Capability<&{CapabilityFilter.Filter}>?, childAddress: Address)
}

/// Functions anyone can call on a manager to get information about an account such as
/// What child accounts it has
pub resource interface ManagerPublic {
    pub fun borrowAccountPublic(addr: Address): &{AccountPublic}?
    pub fun getChildAddresses(): [Address]
    pub fun getOwnedAddresses(): [Address]
}


/// A resource for an account which fills the Parent role of the Child-Parent account
/// management Model. A Manager can redeem or remove child accounts, and obtain any capabilities
/// exposed by the child account to them.
/// TODO: Implement MetadataViews.Resolver and MetadataViews.ResolverCollection
pub resource Manager: ManagerPrivate, ManagerPublic {
    pub let accounts: {Address: Capability<&{AccountPrivate, AccountPublic}>}
    pub let ownedAccounts: {Address: Capability<&{Account, ChildAccountPublic, ChildAccountPrivate}>}
    /// An optional filter to gate what capabilities are permitted to be returned from a proxy account
    /// For example, Dapper Wallet parent account's should not be able to retrieve any FungibleToken Provider capabilities.
    pub let filter: Capability<&{CapabilityFilter.Filter}>?

    pub fun addAccount(_ cap: Capability<&{AccountPrivate, AccountPublic}>) {
        pre {
            self.accounts[cap.address] == nil: "There is already a child account with this address"
        }
    }
    pub fun setManagerCapabilityFilter(cap: Capability<&{CapabilityFilter.Filter}>?, childAddress: Address)
    pub fun removeChild(addr: Address)
    pub fun addOwnedAccount(_ cap: Capability<&{Account, ChildAccountPublic, ChildAccountPrivate}>) {
        pre {
            self.ownedAccounts[cap.address] == nil: "There is already a child account with this address"
        }
    }
    pub fun getAddresses(): [Address]
    pub fun borrowAccount(addr: Address): &{AccountPrivate, AccountPublic}?
    pub fun borrowAccountPublic(addr: Address): &{AccountPublic}?
    pub fun borrowOwnedAccount(addr: Address): &{Account, ChildAccountPublic, ChildAccountPrivate}?
    pub fun removeOwned(addr: Address)
    pub fun giveOwnerShip(addr: Address, to: Address)
    pub fun getChildAddresses(): [Address]
    pub fun getOwnedAddresses(): [Address]
}
```
</details>

<details>
<summary>ProxyAccount</summary>
<code>

```cadence
/// Public methods exposed on a proxy account resource. ChildAccountPublic will share
/// some methods here, but isn't necessarily the same
pub resource interface AccountPublic {
    pub fun getPublicCapability(path: PublicPath, type: Type): Capability?
    pub fun getPublicCapFromProxy(type: Type): Capability?
    pub fun getAddress(): Address
}

/// Methods only accessible to the designated parent of a ProxyAccount
pub resource interface AccountPrivate {
    pub fun getCapability(path: CapabilityPath, type: Type): Capability? {
        post {
            result == nil || [true, nil].contains(self.getManagerCapabilityFilter()?.allowed(cap: result!)): "Capability is not allowed by this account's Parent"
        }
    }
    pub fun getPublicCapability(path: PublicPath, type: Type): Capability?
    pub fun getManagerCapabilityFilter():  &{CapabilityFilter.Filter}?
    pub fun getPrivateCapFromProxy(type: Type): Capability? {
        post {
            result == nil || [true, nil].contains(self.getManagerCapabilityFilter()?.allowed(cap: result!)): "Capability is not allowed by this account's Parent"
        }
    }
    pub fun getPublicCapFromProxy(type: Type): Capability?
    access(contract) fun redeemedCallback(_ addr: Address)
    access(contract) fun setManagerCapabilityFilter(_ managerCapabilityFilter: Capability<&{CapabilityFilter.Filter}>?)
}

/// The ProxyAccount resource sits between a child account and a parent and is stored on the same account as the child account.
/// Once created, a private capability to the proxy account is shared with the intended parent. The parent account
/// will accept this proxy capability into its own manager resource and use it to interact with the child account.
/// Because the ProxyAccount resource exists on the child account itself, whoever owns the child account will be able to manage all
/// ProxyAccount resources it shares, without worrying about whether the upstream parent can do anything to prevent it.
pub resource ProxyAccount: AccountPrivate, AccountPublic {
    access(self) let childCap: Capability<&{BorrowableAccount, ChildAccountPublic}>

    // The CapabilityFactory Manager is a ProxyAccount's way of limiting what types can be asked for
    // by its parent account. The CapabilityFactory returns Capabilities which can be
    // casted to their appropriate types once obtained, but only if the child account has configured their 
    // factory to allow it. For instance, a ProxyAccout might choose to expose NonFungibleToken.Provider, but not
    // FungibleToken.Provider
    pub var factory: Capability<&CapabilityFactory.Manager{CapabilityFactory.Getter}>

    // The CapabilityFilter is a restriction put at the front of obtaining any non-public Capability.
    // Some wallets might want to give access to NonFungibleToken.Provider, but only to **some** of the collections it
    // manages, not all of them.
    pub var filter: Capability<&{CapabilityFilter.Filter}>

    // The CapabilityProxy is a way to share one-off capabilities by the child account. These capabilities can be public OR private
    // and are separate from the factory which returns a capability at a given path as a certain type. When using the CapabilityProxy,
    // you do not have the ability to specify which path a capability came from. For instance, Dapper Wallet might choose to expose
    // a Capability to their Full TopShot collection, but only to the path that the collection exists in.
    pub let proxy: Capability<&CapabilityProxy.Proxy{CapabilityProxy.GetterPublic, CapabilityProxy.GetterPrivate}>

    // managerCapabilityFilter is a component optionally given to a proxy account when a manager redeems it. If this filter
    // is not nil, any Capability returned through the `getCapability` function checks that the manager allows access first.
    access(self) var managerCapabilityFilter: Capability<&{CapabilityFilter.Filter}>?

    pub let parent: Address

    pub fun getAddress(): Address
    access(contract) fun redeemedCallback(_ addr: Address)
    access(contract) fun setManagerCapabilityFilter(_ managerCapabilityFilter: Capability<&{CapabilityFilter.Filter}>?)
    pub fun setCapabilityFactory(_ cap: Capability<&CapabilityFactory.Manager{CapabilityFactory.Getter}>)
    pub fun setCapabilityFilter(_ cap: Capability<&{CapabilityFilter.Filter}>)
    // The main function to a child account's capabilities from a parent account. When a PrivatePath type is used, 
    // the CapabilityFilter will be borrowed and the Capability being returned will be checked against it to 
    // ensure that borrowing is permitted
    pub fun getCapability(path: CapabilityPath, type: Type): Capability?
    pub fun getPrivateCapFromProxy(type: Type): Capability?
    pub fun getPublicCapFromProxy(type: Type): Capability?
    pub fun getPublicCapability(path: PublicPath, type: Type): Capability?
    pub fun getManagerCapabilityFilter():  &{CapabilityFilter.Filter}?
    access(contract) fun setRedeemed(_ addr: Address)
    pub fun borrowCapabilityProxy(): &CapabilityProxy.Proxy?
}
```
</code>
</details>

<details>
<summary>ChildAccount</summary>

```cadence
/*
ChildAccount
A resource which sits on the account it manages to make it easier for apps to configure the behavior they want to permit.
A ChildAccount can be used to create ProxyAccount resources and share them publish them to other addresses.

The ChildAccount can also be used to pass ownership of an account off to another address, or to relinquish ownership entirely,
marking the account as owned by no one. Note that even if there isn't an owner, the parent accounts would still exist, allowing
a form of Hybrid Custody which has no true owner over an account, but shared partial ownership.
*/
pub resource ChildAccount: Account, BorrowableAccount, ChildAccountPublic, ChildAccountPrivate {
    priv var acct: Capability<&AuthAccount>

    pub let parents: {Address: Bool}
    pub var acctOwner: Address?
    pub var relinquishedOwnership: Bool

    access(contract) fun setRedeemed(_ addr: Address) {
        pre {
            self.parents[addr] != nil: "address is not waiting to be redeemed"
        }
    }

    /*
    publishToParent
    A helper method to make it easier to manage what parents an account has configured.
    The steps to sharing this ChildAccount with a new parent are:

    1. Create a new CapabilityProxy for the ProxyAccount resource being created. We make a new one here because
        CapabilityProxy types are meant to be shared explicitly. Making one shared base-line of capabilities might
        introuce an unforseen behavior where an app accidentally shared something to all accounts when it only meant to go
        to one of them. It is better for parent accounts to have less access than they might have anticipated, than for a child
        to have given out access it did not intend to.
    2. Create a new Capability<&{BorrowableAccount}> which has its own unique path for the parent to share this child account with.
        We make new ones each time so that you can revoke access from one parent, without destroying them all. A new Link is made each time
        based on the address being shared to allow this fine-grained control, but it is all managed by the ChildAccount resource itself.
    3. A new @ProxyAccount resource is created and saved, using the CapabilityProxy made in step one, and our CapabilityFactory and CapabilityFilter
        Capabilities. Once saved, public and private links are configured for the ProxyAccount.
    4. Publish the newly made private link to the designated parent's inbox for them to claim on their @Manager resource.
    */
    pub fun publishToParent(
        parentAddress: Address,
        factory: Capability<&CapabilityFactory.Manager{CapabilityFactory.Getter}>,
        filter: Capability<&{CapabilityFilter.Filter}>
    )
    pub fun borrowAccount(): &AuthAccount
    pub fun getParentsAddresses(): [Address]
    pub fun isChildOf(_ addr: Address): Bool
    // returns nil if the given address is not a parent, false if the parent
    // has not redeemed the child account yet, and true if they have
    pub fun getRedeemedStatus(addr: Address): Bool?
    pub fun getParentStatuses(): {Address: Bool}
    /*
    removeParent
    Unlinks all paths configured when publishing an account, and destroy's the @ProxyAccount resource
    configured for the provided parent address. Once done, the parent will not have any valid capabilities
    with which to access the child account.
    */
    pub fun removeParent(parent: Address): Bool
    pub fun getAddress(): Address
    pub fun getOwner(): Address?
    pub fun giveOwnership(to: Address)
    // Revokes all keys on an account, unlinks all currently active AuthAccount capabilities, then makes a new one and replaces the
    // @ChildAccount's underlying AuthAccount Capability with the new one to ensure that all parent accounts can still operate normally.
    // Unless this method is executed via the giveOwnership function, this will leave an account **without** an owner.
    // USE WITH EXTREME CAUTION.
    pub fun seal()
    pub fun borrowProxyAccount(parent: Address): &ProxyAccount?
    pub fun setCapabilityFactoryForParent(parent: Address, cap: Capability<&CapabilityFactory.Manager{CapabilityFactory.Getter}>)
    pub fun borrowCapabilityProxyForParent(parent: Address): &CapabilityProxy.Proxy?
    pub fun addCapabilityToProxy(parent: Address, _ cap: Capability, isPublic: Bool)
    pub fun removeCapabilityFromProxy(parent: Address, _ cap: Capability)
}
```
</details>

<details>
<summary>Proxy</summary>

```cadence
pub resource interface GetterPrivate {
    pub fun getPrivateCapability(_ type: Type): Capability? {
        post {
            result == nil || type.isSubtype(of: result.getType()): "incorrect returned capability type"
        }
    }

    pub fun findFirstPrivateType(_ type: Type): Type?
    pub fun getAllPrivate(): [Capability]
}

pub resource interface GetterPublic {
    pub fun getPublicCapability(_ type: Type): Capability? {
        post {
            result == nil || type.isSubtype(of: result.getType()): "incorrect returned capability type "
        }
    }

    pub fun findFirstPublicType(_ type: Type): Type?
    pub fun getAllPublic(): [Capability]
}

pub resource Proxy: GetterPublic, GetterPrivate {
    access(self) let privateCapabilities: {Type: Capability}
    access(self) let publicCapabilities: {Type: Capability}

    // ------ Begin Getter methods
    pub fun getPublicCapability(_ type: Type): Capability?
    pub fun getPrivateCapability(_ type: Type): Capability?
    pub fun getAllPublic(): [Capability] 
    pub fun getAllPrivate(): [Capability]
    pub fun findFirstPublicType(_ type: Type): Type?
    pub fun findFirstPrivateType(_ type: Type): Type?
    // ------- End Getter methods
    pub fun addCapability(cap: Capability, isPublic: Bool)
    pub fun removeCapability(cap: Capability)
}
```
</details>

</details>

<details>
<summary>Filters</summary>

```cadence
pub resource interface Filter {
    pub fun allowed(cap: Capability): Bool
    pub fun getDetails(): AnyStruct
}

pub resource DenylistFilter: Filter {
    // deniedTypes
    // Represents the underlying types which should not ever be 
    // returned by a RestrictedChildAccount. The filter will borrow
    // a requested capability, and make sure that the type it gets back is not
    // in the list of denied types
    access(self) let deniedTypes: {Type: Bool}

    pub fun addType(_ type: Type)
    pub fun removeType(_ type: Type)
    pub fun allowed(cap: Capability): Bool
    pub fun getDetails(): AnyStruct
}

pub resource AllowlistFilter: Filter {
    // allowedTypes
    // Represents the set of underlying types which are allowed to be 
    // returned by a RestrictedChildAccount. The filter will borrow
    // a requested capability, and make sure that the type it gets back is
    // in the list of allowed types
    access(self) let allowedTypes: {Type: Bool}

    pub fun addType(_ type: Type)
    pub fun removeType(_ type: Type)
    pub fun allowed(cap: Capability): Bool
    pub fun getDetails(): AnyStruct
}

// AllowAllFilter is a passthrough, all requested capabilities are allowed
pub resource AllowAllFilter: Filter {
    pub fun allowed(cap: Capability): Bool
    pub fun getDetails(): AnyStruct
}
```
</details>

<details>
<summary>Factory</summary>

```cadence
pub struct interface Factory {
    pub fun getCapability(acct: &AuthAccount, path: CapabilityPath): Capability
}

pub resource interface Getter {
    pub fun getFactory(_ t: Type): {CapabilityFactory.Factory}?
}

pub resource Manager: Getter {
    pub let factories: {Type: {CapabilityFactory.Factory}}

    pub fun addFactory(_ t: Type, _ f: {CapabilityFactory.Factory})
    pub fun getFactory(_ t: Type): {CapabilityFactory.Factory}?
}
```
</details>

## Considered For Inclusion

### Events

> The following proposed events are in addition to the AuthAccount Capability linking event `flow.AccountLinked(address: Address, path: PrivatePath)` implemented in the Cadence [AuthAccount Capability linking API](https://github.com/onflow/flips/pull/53#issuecomment-1452777257) whenever an account is linked via `AuthAccount.linkAccount(PrivatePath)`.

As some members of the community have voiced, using a shared standard contract can present challenges to dApps subscibing to events on the contract. Ultimately, how do they know which events are relevant to their users' accounts?

With this in mind, the following events and values have been proposed, though additional feedback is requested as the hope is these events can be as helpful as possible to dApps relying on this shared standard.

- **Linking Accounts & Removing Linked Accounts** - Whenever an account is added as a child of a parent account - IOW when a Capability is added/removed to/from a `Manager` - an event is emitted denoting both sides of the link. Other events include coverage for actions involved in readying a child account to be claimed by a `Manager` resource.
    ```cadence
    pub event ProxyAccountPublished(childAcctID: UInt64, proxyAcctID: UInt64, capProxyID: UInt64, factoryID: UInt64, filterID: UInt64, filterType: Type, child: Address, pendingParent: Address)
    pub event ChildAccountRedeemed(id: UInt64, child: Address, parent: Address)
    pub event RemovedParent(id: UInt64, child: Address, parent: Address)
    /* proxy: Manager.accounts | !proxy: Manager.ownedAccounts | active: added | !active: removed */
    pub event AccountUpdated(id: UInt64?, child: Address, parent: Address, proxy: Bool, active: Bool)
    ```

- **Grant/Revoke Capabilities from Child Accounts** - Emitted when access to a Capability has been granted from a child account.
    ```cadence
    /* CapabilityProxy - isPublic: Proxy.publicCapabilities | !isPublic: Proxy.privateCapabilities | active: added | !active: removed */
    pub event ProxyUpdated(id: UInt64, capabilityType: Type, isPublic: Bool, active: Bool)
    /* CapabilityFilter - active: added | !active: removed */
    pub event FilterUpdated(id: UInt64, filterType: Type, type: Type, active: Bool)
    ```

- **Ownership Actions** - A `ChildAccount` resource has the ability to delegate ownership to another account, providing unrestricted access to the account via its encapsulated AuthAccount Capability. This is an important action and deserves relevant events.
    ```cadence
    pub event OwnershipGranted(id: UInt64, child: Address, owner: Address)
    pub event AccountSealed(id: UInt64, address: Address, parents: [Address])
    ```

- **Creation of Standard Resources** - Emitted when a `Manager`, `ChildAccount` and `Proxy` are created. Since the `Manager` creation method is a public contract methods, there isn't any data we can include related to the caller. However, this event could be helpful from a data analysis & user behavior perspective allowing us to gain insight into the adoption of this standard.
    ```cadence
    /* HybridCustody */
    pub event CreatedManager(id: UInt64)
    pub event CreatedChildAccount(id: UInt64, child: Address)
    /* CapabilityProxy */
    pub event ProxyCreated(id: UInt64)
    ```

### MetadataViews Standards Implementation

While not yet implemented, `MetadataViews` interfaces lend themselves well to the need for identifying the purposes of child accounts. For example, a user would want to identify which child account is associated with their MonsterMaker game and which is their social media app account, etc. Leveraging these existing metadata interfaces makes it easy for builders to resolve metadata about a user's linked accounts, at least in regards to the account's association/purpose.

Implementation of the standard is easy enough, but we'll want to consider guidance on source of truth as developer-defined metadata presents a phishing vector. An attacker could create an account with standard resources and attached metadata resembling a victim's account, then publish it for the victim to claim. The victim might then inadvertently then claim the account transfer NFTs or funds to that account which the attacker could then siphon. Requiring user signature to add accounts to their `Manager` helps prevent this, but building out user notifications on `ProxyAccountPublished` presents at minimum a spam vector that could lead to user action exposing them to phishing attacks. A combination of on- and off-chain pet names have been raised a potential construct to minimize the risk associated with these sorts of attack vectors.

With regard to the metadata scheme, this deserves further discussion, and [a PR](https://github.com/Flowtyio/restricted-child-account/pull/22) was recently submitted as a touchpoint of dicusssion for [the issue](https://github.com/Flowtyio/restricted-child-account/issues/18).

### Storage Iteration Convenience Methods

To support developers integrate with Hybrid Custody accounts, it will be essential that we provide example scripts (if not in-built helper methods). Inspecting balances across all of a user's associated accounts is one common use case.

Note that this script doesn't check if the a user has access to the Vaults in those accounts which will be an important consideration in production environments

The most simple use case for this is when dealing with a FT that multiple child accounts hold, or NFTs from the same collection spread among different child accounts. Currently, a dApp would solve this by iterating over each child account's storage either in separate or one large script (the former being more scalable over large number of stored items):

<details>
<summary>Example Account + Storage Iteration Script to get child account balances</summary>

```cadence
import "FungibleToken"
import "MetadataViews"
import "FungibleTokenMetadataViews"
import "HybridCustody"

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
    // Get reference to HybridCustody.Manager if it exists
    if let managerRef = getAccount(address)
        .getCapability<&{HybridCustody.ManagerPublic}>(
            HybridCustody.ManagerPublicPath
        ).borrow() {
        // Iterate over each linked child account in Collection
        for childAddress in managerRef.getChildAddresses() {
            // Ensure all vault type balances are pooled across all addresses
            balances = merge(balances, getAllVaultInfoInAddressStorage(childAddress))
        }
        for ownedAddress in managerRef.getOwnedAddresses() {
            // Ensure all vault type balances are pooled across all addresses
            balances = merge(balances, getAllVaultInfoInAddressStorage(ownedAddress))
        }
    }
    return balances 
}
 
```
</details>

But we could include this functionality in the contract in some different ways. For example, we could add to `Manager` the following methods:

```cadence
pub fun getChildNFTIDs(publicPath: PublicPath, collectionType: Type): {Address: [UInt64]}
pub fun withdrawChildNFT(tokenPath: PublicPath, tokenID: UInt64, child: Address): @NonFungibleToken.NFT
```

Those convenience methods would allow vaults or collections to easily accessible across all child accounts from one parent at the same time. However, iteration over a large number of accounts and/or unexpected path naming conventions within those accounts might lead to unexpected behavior and encourage reliance on iteration at the script layer anyway.

## Discoverability

The notion of discoverability has come up a number of times throughout conversations, and can generally be described as the process of understanding the purpose of, contents in, and access allowed on a particular child account.

Metadata views and resolution helps answer the question of the purpose of a child account. Scripts can help us discover the contents of a given accounts.

The remaining question - what access is allowed on an account via `ProxyAccount` Capability - has not been settled, but is in discussion in [this issue](https://github.com/Flowtyio/restricted-child-account/issues/19). The idea is to include helper methods within a `ProxyAccount` that will enable easy querying on the accessibility a said resource and Capability can facilitate on a child account.

# Drawbacks

AuthAccount Capability is a powerful tool that can be dangerous if it is not used properly. Standardizing how these Capabilities are managed should not have any negative impact on Flow’s ecosystem. Considering there have already been discussions about enhancing the security of potentially dangerous actions with [entitlements](https://github.com/onflow/flips/pull/54) enabled AuthAccounts, the introduction of an account linking transaction pragma (`#allowAccountLinking`), and increasing the auditability and control of Capabilities with [Capability Controllers (CapCons)](https://github.com/onflow/flow/pull/798/files?short_path=f2770e8#diff-f2770e8e35eaed0f7ffa91d366e50cb08f0e18d363c0c5543774d11d7656c8a9), this Flip does not introduce any new attack vectors into the ecosystem.

## Considerations

### Visibility into All Sub-Account Storage

Queries involving storage iteration over a large number of accounts might encounter memory limits either by iteration over too large a number of addresses, too many items in storage, or some combination of the two. This is noted not as a requirement to solve, but as a design consideration in how we communicate an account’s sub-accounts and whether we also include resource/contract methods to easily query the assets in sub-accounts.

### Sources of Truth

Another consideration is preserving the `Manager`, `ProxyAccount` and `ChildAccount` state values as accurate sources of truth for the status of a particular account link. Just as we rely on keys to determine access to an account, so too should we be able to determine from a parent account if it has a capability for another account. Inversely, from a child account we should be able to determine if another account has delegated authority and the address of that account.

### Auditability and Revocation

The proposed design does as well as possible to accurately reflect linking state and identify parent(s) and child parties; however, auditability on delegated access is limited by the nature of Capabilities as they exist today. 

From a user's perspective, even when an account is "owned" & "sealed" - that is, all keys have been revoked and AuthAccount Capailities are linked to a single, deterministically generated path - it should be treated as though a secondary party has access. This is because a malicious agent could link an AuthAccount Capability at the deterministally generated path and issue themselves said Capability in the same transactions ownership is granted. Counter-measures against such a case only go so far, at least until [Capability Controllers](https://github.com/onflow/flow/pull/798) enter the picture.

As such, and also because the current design ultimately puts control & custody in the hands of app developers, it’s recommended that a user remove their own delegated access on an account over revoking an application's access on an account.

### Limiting Delayed Attack Vectors

When it comes to accessing a user’s saved AuthAccount Capabilities, it is possible to restrict Capabilities to retrieval by reference - `&AuthAccount` instead of `Capability<&AuthAccount>`. However, in capability-based access, such restrictions on an issued Capability might be considered an anti-pattern.

With that said, signing a malicious transaction today means you are at risk within the scope of that transaction. Signing a malicious transaction in a world of AuthAccount Capabilities means a bad actor could issue themselves a Capability on your account or one of your child accounts to perform their attack at a later time.

One way to prevent this is to make accessing issued AuthAccount Capabilities ephemeral, limiting the scope of the attack to the time scope of the transaction. Another is to rely on events emitted whenever an owned account's AuthAccount Capability is retrieved from the `ChildAccount`. Yet another measure would include emitting an event any time an AuthAccount Capability is linked.

```cadence
// Let's say a parent account is signing a transaction in which a reference to
// a ChildAccount is retrieved
let ownedAccountRef = parent.borrow<&HybridCustody.Manager>(
		from: HybridCustody.ManagerStoragePath
	)!.borrowOwnedAccount(addr: Address)!

// In the current prototype definitions of ChildAccount
// we return a reference to an auth account which is ephemeral
// in that it cannot be stored. The worst a malicious transaction could
// do is limited in scope to within the transaction being signed
let accountRef: &AuthAccount = ownedAccountRef.borrowAccount()

// Alternatively, we could just return the Capability to the AuthAccount.
// With this, a malicious transaction could publish the capability for themselves,
// retrieve it and store it for later use. This expands at least the time scope
// of the attack and is difficult to audit. 
let accountCap: Capability<&AuthAccount> = ownedAccountRef.getAccountCapability(address: 0x02)
```

Taken together, these measures enable wallet providers to at least notify relevant user’s when any of their accounts trigger an AuthAccount Capability-related event. Such a flow would be similar to the notification you receive from a Web2 identity provider whenever you authorize a new app (e.g. sign in with Google to DapperLabs and Google will let you know you linked your accounts).

### Lack of Ultimate Control

Initially, the idea of Hybrid Custody was to give user's ultimate control over their app accounts. However, over time it became evident this model would not be reasonable for application developers to implement.

The response to the concerns was to place custody and control of Hybrid Custody accounts in the hands of the implementing developers. This means the current proposal is interoperable with existing custodial and funding infrastructure. The tradeoff here is that, at least initially, users have secondary access on these accounts - an application could cut off their access at any time, or another party could be given ownership of the account, superceding the user's access rights.

What this design foregoes in user sovereignty and property rights it makes up for in developer ergonomics and feasibility. A balance is necessary here for the sake of developer ergonomics and adoption of the this feature, and it's believed that this user-restricted access model strikes an appropriate balance.

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

As this FLIP introduces a standard contract and set of resources, best practices around using them are centered around:

1. Creating and funding child accounts
2. Linking existing accounts as a user’s child account
3. Removing a child account
4. **TODO**: Granting ownership of a child account to a parent

## Creating and Funding Child Accounts

Aside from implementing onboarding flows & account linking, you'll want to also consider the account funding & custodial pattern appropriate for the dApp you're building. The only pattern compatible with walletless onboarding is one in which the dApp custodies the child account's key and funds account creation, and thus is the only pattern considered in this FLIP. While there are technically alternative configurations of the resources outlined in this proposal that would allow for alternative custodial patterns, they will only be explored insofar as they lend insight into security risks.

Under the proposed design, account creation and funding can be done any way you'd normally create an account on Flow. Alternatively, there are custodial services that have plans to build APIs which developers can leverage to easily implement Walletless Onboarding and eventually Hybrid Custody into their applications.

## Linking existing account as child account

Given the idea of progressive onboarding, we’ll need to add existing accounts as child accounts. This can be done either by a multisig transaction by the parent & child account, or via the `AuthAccount.Inbox` methods `publish()` and `claim()` across two transactions.

The parent can the take the child account’s Capability & call `Manager.addAccount()` to create the on-chain link between accounts & preserve the child account’s `ProxyAccount` Capability.

For more on this process, see [this walkthrough](#adding-an-account-as-a-child-account-aka-account-linking).

## Revoking a child account

Revocation in this construct involves removing the `ProxyAccount` Capability from the parent’s `Manager`. This can be done via `Manager.removeChild()` method. However, this is only one side of revocation.

## Granting ownership

The other sort of revocation is one in which the current owner or custodian of a delegator account grants ownership to the requesting parent account. Note that a parent account does not have authority to initiate this process on their own, and must rely on the application to both enable and execute this process which may even force the application to surrender its access on the account entirely. This process involves the custodial party calling `ChildAccount.giveOwnership()` which revokes:

1. Key access
2. AuthAccount Capability access - at least any not at a deterministically generated path

With regards to key access, if a developer account custodied the keys, they will be revoked. The granted owner could add a key once they have claimed ownership of the account.

With regards to Capabilities on the child account, the `ChildAccount.seal()` method unlinks any AuthAccount Capabilities and relinks a Capability at a path generated at runtime.

The motivation for total revocation when granting ownership is due to the potential regulatory risks involved with sharing unrestricted access on an account with an unknown party. 

Many custodial applications have gone through great pains to prevent users from having access to fungible tokens, and arbitrarily delegating total access on custodied accounts presents not only technical complexity, but could also imply legal obligations that developers are not prepared or do not wish to deal with. As such, it's recommended that if developers do decide to grant full ownership over accounts to users, they concurrently relinquish application access on those accounts.

## Granting access to child account Capabilities

It's clear that access to typed Capabilities is a requirement for this standard to be useful. `CapabilityFactory` constructs enable retrieval of castable Capabilities while `CapabilityProxy` interfaces and resources enable developers to share arbitrary Capabilities with a parent account. Additionally, `CapabilityFilter` resources enable developers to set rules on the Capabilitites a parent account can access.

As an example, a developer likely wouldn't want to include access to a child account's `FlowToken.Vault` in an `AllowlistFilter`. So one option would be to enumerate the types allowed in an `AllowlistFilter` and include a `Factory` in a `CapabilityFactory.Manager` that returns the desired typed Capabilities, such as an NFT `Collection` related to the app they're building. That way, a user could access the `Collection` from their main account, but the developer can be assured that any $FLOW used to fund the child account's storage remains safe from linked parent accounts.

On the other end, some wallets - where `Manager` resources are likely to be stored - may not want to be granted certain classes of Capabilities. To aid with this case, a `Filter` can be added to the `Manager`, preventing access on unwanted Capabilities. As an example, a custodial wallet may want to avoid access to fungible tokens, and so adds a `DenylistFilter` which prevents access on such Capabilities.

# **Tutorials and Examples**

Assuming this FLIP is approved/implemented formal tutorials and examples will be provided before launch. Examples provided below are illustrative and for alignment purposes, and may be subject to change.

## Adding an Account as a Child Account (AKA “Account Linking”)
> ℹ️ Can be multisigned transaction by both parties or resources & Capabilities can be configured, linked & published by child account then claimed in the parent accounts `Manager`

When delegating access, the delegator ("child") account needs to be configured with the necessary resources & Capabilities to both provide and regulate access according to the rules the developer sets.

1. Save `CapabilityFactory.Manager` and link as `{CapabilityFactory.Getter}` in public
    - Add any `Factory` structs to return typed Capabilities a delegatee ("parent") will need access to
1. Save `CapabilityFilter.Allowlist` (or other `Filter`) and link as `{CapabilityFilter.Filter}` in public
    - Add any Types a delegatee ("parent") will need access to
1. Save `HybridCustody.ChildAccount` and link as `{BorrowableAccount, ChildAccountPublic, ChildAccountPrivate}` in private and `{ChildAccountPublic}` in public
1. Get reference to `HybridCustody.ChildAccount` and call `publishToParent()`, providing public Capabilities to previously configured `{Filter}` and `{Getter}`
    - This creates and saves a `CapabilityProxy.Proxy` specific to the parent, linking as `{GetterPrivate}` and `{GetterPublic}` in private and public respectively
    - Also in scope, a `ProxyAccount` is created, linking as `{AccountPrivate, AccountPublic}` in private and publishing this Capability for the parent to claim

Once published, a user need to accept the published Capability and add it to their `Manager` resource

1. Save `HybridCustody.Manager` and link as `{HybridCustody.ManagerPrivate, HybridCustody.ManagerPublic}` in private and `{HybridCustody.ManagerPublic}` in public
1. Getting a reference to `Manager`, claim the published Capability on the created `ProxyAccount`
    - This Capability is then inserted into the `Manager.accounts` mapping, indexed on the delegator account's `Address`

## Using a Child Account’s NFT Collection

```cadence
import "NonFungibleToken"
import "MetadataViews"
import "ExampleNFT"
import "HybridCustody"

transaction(childAddress: Address, withdrawID: UInt64) {

    let providerRef: &ExampleNFT.Collection{NonFungibleToken.Provider}

    prepare(signer: AuthAccount) {
        // Get a reference to the signer's HybridCustody.Manager from storage
        let managerRef: &HybridCustody.Manager = signer.borrow<&HybridCustody.Manager>(
                from: HybridCustody.ManagerStoragePath
            ) ?? panic("Could not borrow reference to HybridCustody.Manager!")
        // Borrow a reference to the signer's specified ProxyAccount
        let proxyAccountRef: &{HybridCustody.AccountPrivate, HybridCustody.AccountPublic} = managerRef.borrowAccount(addr: childAddress)
            ?? panic("Signer does not have access to specified ProxyAccount")
        let collectionData = ExampleNFT.resolveView(Type<MetadataViews.NFTCollectionData>())! as! MetadataViews.NFTCollectionData
        // Get a Capability
        let cap = proxyAccountRef.getCapability(
                path: collectionData.providerPath, type: Type<&{NonFungibleToken.Provider}>()
            ) ?? panic("Could not borrow a reference to the child account's ExampleNFT Provider!")
        // Cast the Capability as the one we want and borrow
        let providerCap = cap as! Capability<&{NonFungibleToken.Provider}>
        self.providerRef = providerCap.borrow() ?? panic("Returned invalid Provider Capability!")
    }

    execute {
      // Do stuff with the Provider...(e.g. withdraw NFT)
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
- Hybrid custody account
- Linked Accounts

## Open Questions

- [ ] How will entitlements factor in to contract implementation and upgradability?
- [ ] What metadata should be included with the introduction of this standard, and who should have authority to assign & mutate said metadata among all relevant resources?
- [ ] Potentially overlapping with metadata, what discoverability methods should be included to reveal a user's access rights on an account?
- [X] ~~Are there any additional events and/or event data that should be included in this standard?~~
    - A: See [PR](https://github.com/Flowtyio/restricted-child-account/pull/20)
- [X] ~~Is the limitation of public `deposit()` functionality fly too far away from the NFT standard such that its use is not warranted?~~
    - A: We are no longer implementing the NFT standard in this contract, and have made additions to `Manager` private
- [X] ~~Where do Capability Controllers fit in and can they reduce some of the concerns around auditing and revocation of AuthAccount Capabilities?~~
    - A: Any Capability Controller access is done in function scope and would be an upgradable change. If anything, the feature should enhance security guarantees without much refactor needed.
- [ ] ~~Should child accounts be allowed to have multiple parent accounts?~~
    - A: Yes and they will be mediated by a single owning `ChildAccount` managing `ProxyAccount` access for each parent
- [X] ~~~Linking AuthAccounts is possible outside of the mechanisms defined in this standard. How does a user know who else has secondary access to their child accounts.~~
    - A: Until CapCons, this is not technically possible and will need to be addressed via communication and education
- [X] ~~~Do we want `Manager` to be able to revoke other `Manager`s’ access to a child account? For example, let’s say I have access to a set of child accounts and I want to remove anyone else’s access to those accounts - should I be able to and how would I do that if so?~~
    - A: In the current implementation, only the custodian and the parent "owner" of a `ChildAccount` can remove other parents' access

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
- [Hybrid Custody & Restrictions on LinkedAccounts Forum Post](https://forum.onflow.org/t/hybrid-custody-restrictions-on-linked-accounts/4561/)