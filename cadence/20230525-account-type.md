---
status: proposed
flip: 92
authors: Bastian Müller (bastian@dapperlabs.com)
sponsor: Bastian Müller (bastian@dapperlabs.com)
updated: 2023-05-25
---

# FLIP 92: Account Type

## Objective

The goal of this proposal is to improve how Cadence provides access to accounts.
The proposed API aims to make account access simpler and safer,
by enabling and encouraging the principle of least authority (PoLA) for account access.

## Motivation

In Cadence, access to accounts is provided through the built-in types `AuthAccount` and `PublicAccount`:
`AuthAccount` provides _write_ access to an account,
whereas `PublicAccount` only provides _read_ access.

Any program may request read access to an account using the built-in function `getAccount`.
Scripts may request write access to an account using the built-in function `getAuthAccount`.

A transaction declaration may request write access to zero or more accounts,
through the parameter list of a `transaction` declaration's `prepare` block.

For example, a transaction declaration which requests access to two accounts may be written as:

```cadence
transaction {
    prepare(signer1: AuthAccount, signer2: AuthAccount) {
        /* ... */
    }
}
```

The account API has evolved a lot since it was originally introduced, and several new features were added.
This makes the `AuthAccount` type very powerful today:
Having access to the account grants access to **everything** in the account: storage, contracts, keys, capabilities, etc.

Often, programs do not need full access to an account, and at the same time,
it is not possible for developers to only request reduced access.
For example, if a transaction transfers tokens, it only needs access to the account's storage,
and does not need, nor should have the possibility, to manage contracts, keys, etc.

Cadence has recently gained a powerful new language feature which allows specifying access control in a declarative manner:
[entitlements](https://github.com/onflow/flips/blob/main/cadence/20221214-auth-remodel.md),
which can be used to improve this situation.

In addition, [FLIP 89 is proposing further improvements to access control](https://github.com/onflow/flips/pull/89).

The changes that are proposed in this FLIP affect all Cadence programs which work with accounts, mostly transactions.

## User Benefit

Users will benefit from the improved safety that this proposal results in:
Cadence developers will be able to be more precise about the account access their programs require,
which will in turn give users confidence that transactions they sign will only have limited access,
just enough for the transaction to work, and not more.
User agents, like wallets, will be able to analyze transactions and communicate to users
what kind of account access the user will grant to the application.

## Design Proposal

The proposal consists of several changes to the existing account API.

### Refactor `AuthAccount` and `PublicAccount` into a Single `Account` Type

The account types `AuthAccount` and `PublicAccount` are refactored into a single `Account` type.

The functionality of both types intentionally overlaps:
The `PublicAccount` type only contains/exposes the read-only functionality,
whereas the `AuthAccount` type contains/exposes both read and write functionality.

When the two types were introduced,
having separate types was the only means to restrict access to a set of functionality.

Today, references and [entitlements](https://github.com/onflow/flips/blob/main/cadence/20221214-auth-remodel.md)
can be used to implement the access control pattern.

The full type definition of the new `Account` type can be found in the separate subsection below.

### Add Entitlements and Entitlement Mappings for `Account` Type

Several fine-grained entitlements for individual operations (nested functions)
and coarse-grained entitlements for groups of operations (nested types) are introduced.

This allows developers to request access to specific account operations.

For example, the fine-grained entitlement `AddContract` is required to deploy a new contract,
i.e. to call the `contracts.add` function, and the coarse-grained entitlement `Contracts` grants
access to all contract management operations, e.g. to `contracts.update`.
Similar entitlements are added for other individual management operations and categories of operations,
e.g. the fine-grained `AddKey` entitlement is added for allowing to add a key,
and the coarse-grained `Keys` entitlement is added to allow any key management operation.

Entitlement mappings are introduced to propagate entitlements to the account to nested types.

For example, a reference to an account with the fine-grained `AddContract`
or coarse-grained `Contracts` entitlement is propagated to the nested `Account.Contracts` type:

```cadence
transaction {
    prepare(signer: auth(AddContract) &Account) {
        let contracts: auth(AddContract) &Account = signer.contracts
        contracts.add(/* ... omitted ... */)
    }
}
```

As the field `Account.Contracts` has the access modifier `access(AccountMapping)`,
the `AccountMapping` entitlement map is used when accessing the field.
As `AccountMapping` includes the built-in identity mapping,
the entitlement of `AddContract` on the `Account` is mapped as-is to `AddContract`,
and the `add` function, which requires entitlement `AddContract`, can be called.

For a full list of entitlements and entitlement mappings,
see the full type definition of `Account` below.

This proposal depends on an upcoming FLIP which will propose the idea of _identity mapping_:
A common use-case for entitlement mappings is to propagate entitlements from the outer to nested types as-is.
For example, given a reference `account: auth(AddContract) &Account`,
an access of `account.contracts` should also result in `auth(AddContract) &Account`, instead of just `&Account`.

This can already be expressed today,
by adding entitlement mapping entries like `AddContract -> AddContract` for each entitlement.
However, this is quite verbose, especially when many fine-grained entitlements are provided and should be granted transitively.

### Expose Account Access Through References

Instead of exposing the account API directly through an owned `Account` type,
the API is exposed through a reference type, `&Account`.

This makes the semantics clearer and avoids confusion:
The types already internally have reference semantics,
i.e. multiple copies refer to the same account,
copying an `AuthAccount` or `PublicAccount` value does not create a new account.

As a result, `PublicAccount` is equivalent to `&Account`, an unauthorized reference,
i.e., one that does not have any entitlements, and therefore can only access read-only functionality.

Further, `AuthAccount` is equivalent to `auth(Storage, Contracts, Keys, Inbox, Capabilities) &Account`,
an authorized reference which grants all entitlements, i.e., and therefore can access all functionality
(read and write operations).

This has a nice side-effect: More powerful references are longer and a warning sign.
The minimal type `&Account` is the easiest to write and only grants read access,
and developers have to go out of their way to define more powerful references.

This allows developers to request specific access in a transaction.
For example, a transaction which deploys a new contract can use the type `auth(AddContract) &Account`
in the parameter list of the `prepare` block of the transaction.
This ensures that the transaction is only allowed to deploy a contract,
and is not able to perform any other write operations on the account,
e.g. it cannot transfer an object, add a new key, etc.

Contracts have an implicit field named `account` which grants the contract full access to the account the contract is deployed to.
The field of the type is changed from `AuthAccount` to `auth(Storage, Contracts, Keys, Inbox, Capabilities) &Account`.
A future FLIP could maybe allow contracts to override the type of the field,
which would allow developers to restrict the access a contract has to the account.

This proposal depends on [FLIP 89](https://github.com/onflow/flips/pull/89),
which proposes that field access on references returns a reference.
For example, given a reference `account: &Account`,
accessing the field `contracts` results in another reference `&Account.Contracts`
(instead of the field's type, just `Account.Contracts`).

Account capabilities already leverage account references:
The type of an account capability is `Capability<&AuthAccount>`.
Just like for the non-reference type `AuthAccount`,
the equivalent of the reference type `&AuthAccount`
will be `auth(Storage, Contracts, Keys, Inbox, Capabilities) &Account`.

### Refactor Storage-Related Functionality into a Nested Type

Over time, the functionality exposed through the `AuthAccount` type has grown significantly.
Today, the type grants management access to several parts of the account, such as storage, keys, contracts, capabilities, etc.
Originally, all functionality was exposed directly in the type itself,
which resulted in a large number of unrelated fields and functions.

Previous FLIPs addressed this by grouping related functionality into nested sub-types.
For example, the key and contract management APIs were grouped into the `Keys` and `Contracts` sub-types.
New functionality, such as the capability inbox and capability controller APIs, were added immediately using this pattern,
i.e. as the `Inbox` and `Capabilities` sub-types.

What remains in the account types are the storage-related functions and fields,
such as `save`, `load`, `storageUsed`, and `storageCapacity`.
Those storage-related functions and fields are refactored into a new sub-type `Storage`,
and exposed through a new `storage` field.

### Remove the `#allowAccountLinking` Pragma

When Account Linking was added to Cadence,
the `#allowAccountLinking` pragma was temporarily added to allow user agents (e.g. wallets)
to detect that a transaction requests the ability to create a new account link.

This proposal obsoletes the pragma – the same functionality,
statically requesting/granting access to creating an account capability,
can now be expressed through the type system.

For example, a transaction can only issue a new account capability controller
if the transaction requests the fine-grained `IssueAccountCapabilityController`
or coarse-grained `Capabilities` entitlement.

See the [Examples](#Examples) section to see how a transaction which issues an account capability controller
would look like.

### Full `Account` Type Definition

```cadence
access(all)
struct Account {

    /// The address of the account.
    access(all)
    let address: Address

    /// The FLOW balance of the default vault of this account.
    access(all)
    let balance: UFix64

    /// The FLOW balance of the default vault of this account that is available to be moved.
    access(all)
    let availableBalance: UFix64

    /// The storage of the account.
    access(AccountMapping)
    let storage: Account.Storage

    /// The contracts deployed to the account.
    access(AccountMapping)
    let contracts: Account.Contracts

    /// The keys assigned to the account.
    access(AccountMapping)
    let keys: Account.Keys

    /// The inbox allows bootstrapping (sending and receiving) capabilities.
    access(AccountMapping)
    let inbox: Account.Inbox

    /// The capabilities of the account.
    access(AccountMapping)
    let capabilities: Account.Capabilities

    access(all)
    struct Storage {
        /// The current amount of storage used by the account in bytes.
        access(all)
        let used: UInt64

        /// The storage capacity of the account in bytes.
        access(all)
        let capacity: UInt64

        /// All public paths of this account.
        access(all)
        let publicPaths: [PublicPath]

        /// All storage paths of this account.
        access(all)
        let storagePaths: [StoragePath]

        /// Saves the given object into the account's storage at the given path.
        ///
        /// Resources are moved into storage, and structures are copied.
        ///
        /// If there is already an object stored under the given path, the program aborts.
        ///
        /// The path must be a storage path, i.e., only the domain `storage` is allowed.
        access(SaveValue)
        fun save<T: Storable>(_ value: T, to: StoragePath)

        /// Reads the type of an object from the account's storage which is stored under the given path,
        /// or nil if no object is stored under the given path.
        ///
        /// If there is an object stored, the type of the object is returned without modifying the stored object.
        ///
        /// The path must be a storage path, i.e., only the domain `storage` is allowed.
        access(all)
        view fun type(at path: StoragePath): Type?

        /// Loads an object from the account's storage which is stored under the given path,
        /// or nil if no object is stored under the given path.
        ///
        /// If there is an object stored,
        /// the stored resource or structure is moved out of storage and returned as an optional.
        ///
        /// When the function returns, the storage no longer contains an object under the given path.
        ///
        /// The given type must be a supertype of the type of the loaded object.
        /// If it is not, the function panics.
        ///
        /// The given type must not necessarily be exactly the same as the type of the loaded object.
        ///
        /// The path must be a storage path, i.e., only the domain `storage` is allowed.
        access(LoadValue)
        fun load<T: Storable>(from: StoragePath): T?

        /// Returns a copy of a structure stored in account storage under the given path,
        /// without removing it from storage,
        /// or nil if no object is stored under the given path.
        ///
        /// If there is a structure stored, it is copied.
        /// The structure stays stored in storage after the function returns.
        ///
        /// The given type must be a supertype of the type of the copied structure.
        /// If it is not, the function panics.
        ///
        /// The given type must not necessarily be exactly the same as the type of the copied structure.
        ///
        /// The path must be a storage path, i.e., only the domain `storage` is allowed.
        access(all)
        view fun copy<T: AnyStruct>(from: StoragePath): T?

        /// Returns true if the object in account storage under the given path satisfies the given type,
        /// i.e. could be borrowed using the given type.
        ///
        /// The given type must not necessarily be exactly the same as the type of the     borrowed object.
        ///
        /// The path must be a storage path, i.e., only the domain `storage` is allowed.
        access(all)
        view fun check<T: Any>(from: StoragePath): Bool

        /// Returns a reference to an object in storage without removing it from storage.
        ///
        /// If no object is stored under the given path, the function returns nil.
        /// If there is an object stored, a reference is returned as an optional,
        /// provided it can be borrowed using the given type.
        /// If the stored object cannot be borrowed using the given type, the function panics.
        ///
        /// The given type must not necessarily be exactly the same as the type of the borrowed object.
        ///
        /// The path must be a storage path, i.e., only the domain `storage` is allowed
        access(BorrowValue)
        view fun borrow<T: &Any>(from: StoragePath): T?

        /// Iterate over all the public paths of an account,
        /// passing each path and type in turn to the provided callback function.
        ///
        /// The callback function takes two arguments:
        ///   1. The path of the stored object
        ///   2. The runtime type of that object
        ///
        /// Iteration is stopped early if the callback function returns `false`.
        ///
        /// The order of iteration is undefined.
        ///
        /// If an object is stored under a new public path,
        /// or an existing object is removed from a public path,
        /// then the callback must stop iteration by returning false.
        /// Otherwise, iteration aborts.
        ///
        access(all)
        fun forEachPublic(_ function: fun(PublicPath, Type): Bool)

        /// Iterate over all the stored paths of an account,
        /// passing each path and type in turn to the provided callback function.
        ///
        /// The callback function takes two arguments:
        ///   1. The path of the stored object
        ///   2. The runtime type of that object
        ///
        /// Iteration is stopped early if the callback function returns `false`.
        ///
        /// If an object is stored under a new storage path,
        /// or an existing object is removed from a storage path,
        /// then the callback must stop iteration by returning false.
        /// Otherwise, iteration aborts.
        access(all)
        fun forEachStored(_ function: fun (StoragePath, Type): Bool)
    }

    access(all)
    struct Contracts {

        /// The names of all contracts deployed in the account.
        access(all)
        let names: [String]

        /// Adds the given contract to the account.
        ///
        /// The `code` parameter is the UTF-8 encoded representation of the source code.
        /// The code must contain exactly one contract or contract interface,
        /// which must have the same name as the `name` parameter.
        ///
        /// All additional arguments that are given are passed further to the initializer
        /// of the contract that is being deployed.
        ///
        /// The function fails if a contract/contract interface with the given name already exists in the account,
        /// if the given code does not declare exactly one contract or contract interface,
        /// or if the given name does not match the name of the contract/contract interface declaration in the code.
        ///
        /// Returns the deployed contract.
        access(AddContract)
        fun add(
            name: String,
            code: [UInt8]
        ): DeployedContract

        /// Updates the code for the contract/contract interface in the account.
        ///
        /// The `code` parameter is the UTF-8 encoded representation of the source code.
        /// The code must contain exactly one contract or contract interface,
        /// which must have the same name as the `name` parameter.
        ///
        /// Does **not** run the initializer of the contract/contract interface again.
        /// The contract instance in the world state stays as is.
        ///
        /// Fails if no contract/contract interface with the given name exists in the account,
        /// if the given code does not declare exactly one contract or contract interface,
        /// or if the given name does not match the name of the contract/contract interface declaration in the code.
        ///
        /// Returns the deployed contract for the updated contract.
        access(UpdateContract)
        fun update(name: String, code: [UInt8]): DeployedContract

        /// Returns the deployed contract for the contract/contract interface with the given name in the account, if any.
        ///
        /// Returns nil if no contract/contract interface with the given name exists in the account.
        access(all)
        view fun get(name: String): DeployedContract?

        /// Removes the contract/contract interface from the account which has the given name, if any.
        ///
        /// Returns the removed deployed contract, if any.
        ///
        /// Returns nil if no contract/contract interface with the given name exists in the account.
        access(RemoveContract)
        fun remove(name: String): DeployedContract?

        /// Returns a reference of the given type to the contract with the given name in the account, if any.
        ///
        /// Returns nil if no contract with the given name exists in the account,
        /// or if the contract does not conform to the given type.
        access(all)
        view fun borrow<T: &Any>(name: String): T?
    }

    access(all)
    struct Keys {

        /// Adds a new key with the given hashing algorithm and a weight.
        ///
        /// Returns the added key.
        access(AddKey)
        fun add(
            publicKey: PublicKey,
            hashAlgorithm: HashAlgorithm,
            weight: UFix64
        ): AccountKey

        /// Returns the key at the given index, if it exists, or nil otherwise.
        ///
        /// Revoked keys are always returned, but they have `isRevoked` field set to true.
        access(all)
        view fun get(keyIndex: Int): AccountKey?

        /// Marks the key at the given index revoked, but does not delete it.
        ///
        /// Returns the revoked key if it exists, or nil otherwise.
        access(RevokeKey)
        fun revoke(keyIndex: Int): AccountKey?

        /// Iterate over all unrevoked keys in this account,
        /// passing each key in turn to the provided function.
        ///
        /// Iteration is stopped early if the function returns `false`.
        ///
        /// The order of iteration is undefined.
        access(all)
        fun forEach(_ function: fun(AccountKey): Bool)

        /// The total number of unrevoked keys in this account.
        access(all)
        let count: UInt64
    }

    access(all)
    struct Inbox {

        /// Publishes a new Capability under the given name,
        /// to be claimed by the specified recipient.
        access(PublishInboxCapability)
        fun publish(_ value: Capability, name: String, recipient: Address)

        /// Unpublishes a Capability previously published by this account.
        ///
        /// Returns `nil` if no Capability is published under the given name.
        ///
        /// Errors if the Capability under that name does not match the provided type.
        access(UnpublishInboxCapability)
        fun unpublish<T: &Any>(_ name: String): Capability<T>?

        /// Claims a Capability previously published by the specified provider.
        ///
        /// Returns `nil` if no Capability is published under the given name,
        /// or if this account is not its intended recipient.
        ///
        /// Errors if the Capability under that name does not match the provided type.
        access(ClaimInboxCapability)
        fun claim<T: &Any>(_ name: String, provider: Address): Capability<T>?
    }

    access(all)
    struct Capabilities {

        /// The storage capabilities of the account.
        access(CapabilitiesMapping)
        let storage: Account.StorageCapabilities

        /// The account capabilities of the account.
        access(CapabilitiesMapping)
        let account: Account.AccountCapabilities

        /// Returns the capability at the given public path.
        /// Returns nil if the capability does not exist,
        /// or if the given type is not a supertype of the capability's borrow type.
        access(all)
        view fun get<T: &Any>(_ path: PublicPath): Capability<T>?

        /// Borrows the capability at the given public path.
        /// Returns nil if the capability does not exist, or cannot be borrowed using the given type.
        /// The function is equivalent to `get(path)?.borrow()`.
        access(all)
        view fun borrow<T: &Any>(_ path: PublicPath): T?

        /// Publish the capability at the given public path.
        ///
        /// If there is already a capability published under the given path, the program aborts.
        ///
        /// The path must be a public path, i.e., only the domain `public` is allowed.
        access(PublishCapability)
        fun publish(_ capability: Capability, at: PublicPath)

        /// Unpublish the capability published at the given path.
        ///
        /// Returns the capability if one was published at the path.
        /// Returns nil if no capability was published at the path.
        access(UnpublishCapability)
        fun unpublish(_ path: PublicPath): Capability?
    }

    access(all)
    struct StorageCapabilities {

        /// Get the storage capability controller for the capability with the specified ID.
        ///
        /// Returns nil if the ID does not reference an existing storage capability.
        access(GetStorageCapabilityController)
        view fun getController(byCapabilityID: UInt64): &StorageCapabilityController?

        /// Get all storage capability controllers for capabilities that target this storage path
        access(GetStorageCapabilityController)
        view fun getControllers(forPath: StoragePath): [&StorageCapabilityController]

        /// Iterate over all storage capability controllers for capabilities that target this storage path,
        /// passing a reference to each controller to the provided callback function.
        ///
        /// Iteration is stopped early if the callback function returns `false`.
        ///
        /// If a new storage capability controller is issued for the path,
        /// an existing storage capability controller for the path is deleted,
        /// or a storage capability controller is retargeted from or to the path,
        /// then the callback must stop iteration by returning false.
        /// Otherwise, iteration aborts.
        access(GetStorageCapabilityController)
        fun forEachController(
            forPath: StoragePath,
            _ function: fun(&StorageCapabilityController): Bool
        )

        /// Issue/create a new storage capability.
        access(IssueStorageCapabilityController)
        fun issue<T: &Any>(_ path: StoragePath): Capability<T>
    }

    access(all)
    struct AccountCapabilities {
        /// Get capability controller for capability with the specified ID.
        ///
        /// Returns nil if the ID does not reference an existing account capability.
        access(GetAccountCapabilityController)
        view fun getController(byCapabilityID: UInt64): &AccountCapabilityController?

        /// Get all capability controllers for all account capabilities.
        access(GetAccountCapabilityController)
        view fun getControllers(): [&AccountCapabilityController]

        /// Iterate over all account capability controllers for all account capabilities,
        /// passing a reference to each controller to the provided callback function.
        ///
        /// Iteration is stopped early if the callback function returns `false`.
        ///
        /// If a new account capability controller is issued for the account,
        /// or an existing account capability controller for the account is deleted,
        /// then the callback must stop iteration by returning false.
        /// Otherwise, iteration aborts.
        access(GetAccountCapabilityController)
        fun forEachController(_ function: fun(&AccountCapabilityController): Bool)

        /// Issue/create a new account capability.
        access(IssueAccountCapabilityController)
        fun issue<T: &Account{}>(): Capability<T>
    }
}

/* Storage entitlements */

entitlement Storage

entitlement SaveValue
entitlement LoadValue
entitlement BorrowValue

/* Contract entitlements */

entitlement Contracts

entitlement AddContract
entitlement UpdateContract
entitlement RemoveContract

/* Key entitlements */

entitlement Keys

entitlement AddKey
entitlement RevokeKey

/* Inbox entitlements */

entitlement Inbox

entitlement PublishInboxCapability
entitlement UnpublishInboxCapability
entitlement ClaimInboxCapability

/* Capability entitlements */

entitlement Capabilities

entitlement StorageCapabilities
entitlement AccountCapabilities

entitlement PublishCapability
entitlement UnpublishCapability

entitlement GetStorageCapabilityController
entitlement IssueStorageCapabilityController

entitlement GetAccountCapabilityController
entitlement IssueAccountCapabilityController

/* Entitlement mappings */

entitlement mapping AccountMapping {
    /*
    Identity is a built-in mapping which implicitly maps E -> E, for any entitlement E.
    A FLIP will propose this feature separately.
    */
    include Identity

    Storage -> SaveValue
    Storage -> LoadValue
    Storage -> BorrowValue

    Contracts -> AddContract
    Contracts -> UpdateContract
    Contracts -> RemoveContract

    Keys -> AddKey
    Keys -> RevokeKey

    Inbox -> PublishInboxCapability
    Inbox -> UnpublishInboxCapability
    Inbox -> ClaimInboxCapability

    Capabilities -> StorageCapabilities
    Capabilities -> AccountCapabilities
}

entitlement mapping CapabilitiesMapping {
    include Identity

    StorageCapabilities -> GetStorageCapabilityController
    StorageCapabilities -> IssueStorageCapabilityController

    AccountCapabilities -> GetAccountCapabilityController
    AccountCapabilities -> IssueAccountCapabilityController
}
```

### Adjust `getAccount`

The function `getAccount` allows accessing the public portion of an account
in any context, in transactions and scripts.

The function currently returns a `PublicAccount` value, and has the signature:

```cadence
fun getAccount(_ address: Address): PublicAccount
```

The return type of the function is changed from `PublicAccount`
to the equivalent `&Account`,
i.e., the function is changed to have the following signature:

```cadence
fun getAccount(_ address: Address): &Account
```

### Adjust `getAuthAccount`

The function `getAuthAccount` allows accessing the authorized portion of an account in a script.

The function currently returns an `AuthAccount` value, and has the signature:

```cadence
fun getAuthAccount(_ address: Address): AuthAccount
```

The return type of the function is changed from `AuthAccount` to a type parameter,
i.e., the function is changed to have the following signature:

```cadence
fun getAuthAccount<T: &Account>(_ address: Address): T
```

This allows "summoning" any account type needed.
For example, to get access to the storage of account at address 0x1:

```cadence
let ref = getAuthAccount<auth(Storage) &Account>(0x1)
```

### Adjust `account` Field of Contracts

All contracts have a a pre-declared field `account`, which currently has the type `AuthAccount`.

The field's type is changed to `auth(Storage, Contracts, Keys, Inbox, Capabilities) &Account`.

### Replace `AuthAccount` constructor with `Account` constructor

The constructor function `fun AuthAccount(payer: AuthAccount): AuthAccount` creates a new account,
given an account which is charged the cost for the creation.

This function is replaced with the new function
`fun Account(payer: auth(BorrowValue | Storage) &Account): auth(Storage, Contracts, Keys, Inbox, Capabilities) &Account`.
Like before, it creates a new account, but now returns a fully-entitled reference to the new account.
Like before, an account must pay for the creation,
but given that the charging only requires borrowing a [fungible token vault](https://github.com/onflow/flow-ft) from storage,
only those entitlements are necessary.

### Drawbacks

This proposal depends on several language features: references, entitlements, entitlement mappings, field access on references, etc.
As the majority of Cadence programs interact with accounts, the majority of developers will use this new account API.

Usage of the API will likely be unsurprising to developers,
but understanding why and how the API works will require an understanding of those language features.

### Alternatives Considered

None

### Performance Implications

None

### Engineering Impact

The effort required to implement the proposed changes is about a month of work.
In addition, existing stored data must be migrated, which will require additional work to implement a storage migration.

### Best Practices

Cadence programs, especially transactions, should follow the principle of least authority (PoLA):
They should only request the least amount of entitlements for an account
that are necessary to perform the intended operation on the account.

Fine-grained account entitlements should be preferred over coarse-grained account entitlements.

Fully entitled account references should be avoided, as they are very powerful,
and unlikely to be necessary for the majority of use-cases.

### Examples

#### Deploying a Contract

Today, a transaction which deploys a contract is likely written as:

```cadence
transaction {
    prepare(signer: AuthAccount) {
        signer.contracts.add(name: "MyContract", code: [/* code */])
    }
}
```

With this FLIP implemented, the same transaction can now be written as follows:

```cadence
transaction {
    prepare(signer: auth(AddContract) &Account) {
        signer.contracts.add(name: "MyContract", code: [/* code */])
    }
}
```

Note the change in the parameter list of the `prepare` block:
Instead of requesting access to the whole account, only the `AddContract` entitlement is requested,
which means that the `contracts.add` function is available,
while other operations on the account, like adding a key (`keys.add`), are unavailable.

### Linking an Account Capability / Issuing an Account Capability Controller

Today, a transaction which using the linking-based capability API is likely written as:

```cadence
#allowAccountLinking

transaction {
    prepare(signer: AuthAccount) {
        signer.linkAccount(/private/account)
    }
}
```

With the new
[Capability Controllers API](https://github.com/onflow/flips/blob/main/cadence/20220203-capability-controllers.md)
it is likely written as:

```cadence
transaction {
    prepare(signer: AuthAccount) {
        signer.capabilities.account.issue<&AuthAccount>()
    }
}
```

With this FLIP implemented, the same transaction can now be written as follows:

```cadence
transaction {
    prepare(signer: auth(IssueAccountCapabilityController) &Account) {
        signer.capabilities.account.issue<&Account>()
    }
}
```

Note the change in the parameter list of the `prepare` block:
Instead of requesting access to the whole account, and annotating the transaction with a special pragma,
only the `IssueAccountCapabilityController` entitlement is requested,
which means that the `capabilities.account.issue` function is available,
while other operations on the account, like adding a key (`keys.add`), are unavailable.

In addition, note how the `issue` function takes a type parameter.
Currently, the only possible type is `&AuthAccount`,
which allows the capability to perform all operations on the account.
With this FLIP, the type is any subtype of `&Account`.
That means by "default", the capability is not authorized to perform any operations on the account,
and the issuer may choose to grant certain coarse or fine-grained entitlements as needed.

For example, it is possible to issue a new account capability which is able to access the account's storage,
but not perform other operations, like adding keys or contracts:

```cadence
signer.capabilities.account.issue<auth(Storage) &Account>()
```

### Compatibility

The proposal is not backwards-compatible, it is a breaking change.

Existing programs which use the account API, i.e. the `AuthAccount` and `PublicAccount`,
will need to be migrated to the new `Account` type.

Existing stored data will be migrated automatically:
- The type `PublicAccount` will be migrated to `&Account`.
- The type `AuthAccount` will be migrated to `auth(Storage, Contracts, Keys, Inbox, Capabilities) &Account`.

To provide better backwards compatibility and reduce the amount of broken programs,
`PublicAccount` and `AuthAccount` could stay in the language as type aliases for their respective equivalents.
However, this is intentionally *not* proposed:
Not providing aliases forces developers to update their programs and encourages them to follow PoLA,
ultimately resulting in a better, and especially safer experience for users.

### User Impact

The majority of Cadence developers will be impacted by the change and will need to update their programs,
especially transactions.
The required changes should be minimal:
Statements can very often stay as-is,
the majority of changes will be for type annotations, e.g. in transactions.
That will require developers to determine what set of entitlements need to be requested.
In return, safety for users will improve,
as they will be able to better understand what operations may be performed by a transaction they sign.

## Related Issues

- [FLIP 89](https://github.com/onflow/flips/pull/89)
- [FLIP 86](https://github.com/onflow/flips/pull/86)

## Prior Art

None

## Questions and Discussion Topics

### Naming of entitlements

It is still unclear how entitlements should be named.

This proposal, along with [FLIP 86](https://github.com/onflow/flips/pull/86),
is the first case where built-in entitlements are added to Cadence.
As such, the FLIP is setting a precedent for the naming of entitlements in general.

For now, entitlements are named in verb form, in upper camel-case.
That can be easily changed.
