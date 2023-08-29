---
status: proposed
flip: X
authors: Giovanni Sanchez (giovanni.sanchez@dapperlabs.com)
sponsor: Giovanni Sanchez (giovanni.sanchez@dapperlabs.com)
updated: 08-29-2023
---

# FLIP X: Staged Contract Update Mechanism

## Objective

This proposal outlines a mechanism for automated and efficient contract updates to take effect at or beyond a specified
block height with the goal of minimizing recovery time following breaking improvements. Included in this FLIP is a
design enabling contract developers to pre-define a sequence of contract updates across an arbitrary number of contracts
and accounts, and either execute these updates themselves or delegate deployment authority to some trusted party. 

## Motivation

Immediately following planned Cadence improvements on Mainnet (marked by the Cadence 1.0 milestone), many contracts
across the ecosystem will need to be updated. In preparation for this milestone, the Flow community is considering ways
to make this migration as quick and seamless as possible, primarily motivated by the desire to minimize any
user-perceived downtime. With this in mind, it's in the interest of everyone to provide helpful and reliable tools to
empower and support developers to upgrade their contracts.

Surely, some developers will prefer to excecute updates themselves manually. However, others may find it useful and
even preferable to both codify their updated deployment using onchain constructs and additionally delegate update
execution to an automated service run by Flow as a part of the ecosystem-wide post-spork migration.

## User Benefit

As implied above, this contract and update service would allow developers to configure and schedule their update deployment onchain.

## Design Proposal

### Overview

### Implementation Details

#### Interfaces

<details>
<summary>struct ContractUpdate</summary>

```cadence
pub struct ContractUpdate {
    pub let address: Address
    pub let name: String
    pub let code: [UInt8]
}
```
</details>

<details>
<summary>resource Updater</summary>

```cadence
/// Private Capability enabling delegated updates
///
pub resource interface DelegatedUpdater {
    pub fun update(): Bool
}

/// Public interface enabling queries about the Updater
///
pub resource interface UpdaterPublic {
    pub fun getID(): UInt64
    pub fun getBlockUpdateBoundary(): UInt64
    pub fun getContractAccountAddresses(): [Address]
    pub fun getDeployments(): [[ContractUpdate]]
    pub fun getCurrentDeploymentStage(): Int
    pub fun getFailedDeployments(): {Int: [String]}
    pub fun hasBeenUpdated(): Bool
}

/// Resource that enables delayed contract updates to a wrapped account at or beyond a specified block height
///
pub resource Updater : UpdaterPublic, DelegatedUpdater {
    /// Update to occur at or beyond this block height
    // TODO: Consider making this a contract-owned value as it's reflective of the spork height
    access(self) let blockUpdateBoundary: UInt64
    /// Update status for each contract
    access(self) var updateComplete: Bool
    /// Capabilities for contract hosting accounts
    access(self) let accounts: {Address: Capability<&AuthAccount>}
    /// Updates ordered by their deployment sequence and staged by their dependency depth
    /// NOTE: Dev should be careful to validate their dependency tree such that updates are performed from root 
    /// to leaf dependencies
    access(self) let deployments: [[ContractUpdate]]
    /// Current deployment stage
    access(self) var currentDeploymentStage: Int
    /// Contracts whose update failed keyed on their deployment stage
    access(self) let failedDeployments: {Int: [String]}

    /// Executes the next update stabe using Account.Contracts.update__experimental() for all contracts defined in
    /// deployment, returning true if all stages have been attempted and false if stages remain
    ///
    pub fun update(): Bool

    /* --- Public getters --- */
    //
    pub fun getID(): UInt64
    pub fun getBlockUpdateBoundary(): UInt64
    pub fun getContractAccountAddresses(): [Address]
    pub fun getDeployments(): [[ContractUpdate]]
    pub fun getCurrentDeploymentStage(): Int
    pub fun getFailedDeployments(): {Int: [String]}
    pub fun hasBeenUpdated(): Bool
}
```
</details>


<details>
<summary>resource Delegatee</summary>

```cadence
/// Public interface for Delegatee
///
pub resource interface DelegateePublic {
    pub fun check(id: UInt64): Bool?
    pub fun getUpdaterIDs(): [UInt64]
    pub fun delegate(updaterCap: Capability<&Updater{DelegatedUpdater, UpdaterPublic}>)
    pub fun removeAsUpdater(updaterCap: Capability<&Updater{DelegatedUpdater, UpdaterPublic}>)
}

/// Resource that executed delegated updates
///
pub resource Delegatee : DelegateePublic {
    // TODO: Block Height - All DelegatedUpdaters must be updated at or beyond this block height
    // access(self) let blockUpdateBoundary: UInt64
    /// Track all delegated updaters
    // TODO: If we support staged updates, we'll want visibility into the number of stages and progress through all
    //      maybe removing after stages have been complete or failed
    access(self) let delegatedUpdaters: {UInt64: Capability<&Updater{DelegatedUpdater, UpdaterPublic}>}

    /// Checks if the specified DelegatedUpdater Capability is contained and valid
    ///
    pub fun check(id: UInt64): Bool?
    /// Returns the IDs of the delegated updaters 
    ///
    pub fun getUpdaterIDs(): [UInt64]
    /// Allows for the delegation of updates to a contract
    ///
    pub fun delegate(updaterCap: Capability<&Updater{DelegatedUpdater, UpdaterPublic}>)
    /// Enables Updaters to remove their delegation
    ///
    pub fun removeAsUpdater(updaterCap: Capability<&Updater{DelegatedUpdater, UpdaterPublic}>)
    /// Executes update on the specified Updater
    ///
    // TODO: Consider removing Capabilities once we get signal that the Updater has been completed
    pub fun update(updaterIDs: [UInt64]): [UInt64]
    /// Enables admin removal of a DelegatedUpdater Capability
    ///
    pub fun removeDelegatedUpdater(id: UInt64)
}
```
</details>


#### Events

```cadence
pub event UpdaterCreated(updaterUUID: UInt64, blockUpdateBoundary: UInt64)
pub event UpdaterUpdated(
    updaterUUID: UInt64,
    updaterAddress: Address?,
    blockUpdateBoundary: UInt64,
    updatedAddresses: [Address],
    updatedContracts: [String],
    failedAddresses: [Address],
    failedContracts: [String],
    updateComplete: Bool
)
pub event UpdaterDelegationChanged(updaterUUID: UInt64, updaterAddress: Address?, delegated: Bool)
```

#### Helper Methods

```cadence
/// Returns the Address of the Delegatee associated with this contract
///
pub fun getContractDelegateeAddress(): Address

/// Helper method that returns the ordered array reflecting sequenced and staged deployments, with each contract
/// update represented by a ContractUpdate struct.
///
/// NOTE: deploymentConfig is ordered, and the order is used to determine both the order of the contracts in each
/// deployment and the order of the deployments themselves. Each entry in the inner array must be exactly one
/// key-value pair, where the key is the address of the associated contract name and code.
///
pub fun getDeploymentFromConfig(_ deploymentConfig: [[{Address: {String: String}}]]): [[ContractUpdate]]

/// Returns a new Updater resource
///
pub fun createNewUpdater(
    blockUpdateBoundary: UInt64,
    accounts: [Capability<&AuthAccount>],
    deployments: [[ContractUpdate]]
): @Updater

/// Creates a new Delegatee resource enabling caller to self-host their Delegatee
///
pub fun createNewDelegatee(): @Delegatee
```

#### Note on Update API

- Design is heavily dependent on the existence of an alternative update API
    - Problem being that failed updates prevent iteration, and lack of iteration would significantly impact the final contract design and higher-level architecture
        - e.g. Required to submit one update per transaction compared to hundreds - at least one order of magnitude difference
            - That many transactions would also require more complex signing architecture due to the requisite number of proposal keys
- Issue currently open for `tryUpdate()`, but not finalized
    - Ideally this method would 

#### Managing Dependencies

#### 

### Considerations

Update executions via `Delegatee` can't account for the global dependency graph since there is no way to enforce it executes all updates, and so depend on the `Updater`'s creator to sequence the contained deployment appropriately given the contracts to be updated. This means that developers will still want to validate their contracts are both individually Stable Cadence compatible & updatable as well as sequenced correctly within the `Updater` resource.

Another consideration is 

### Drawbacks

- All contracts would need to be either a) core contracts or b) owned by the updater

### Considered Alternatives

### Performance Implications

### Best Practices

### Examples

### Compatibility

### User Impact

## Related Issues

- [Issue: Staged Contract Update](https://github.com/onflow/cadence/issues/2019)
- [Tracker: Cadence 1.0](https://github.com/onflow/cadence/issues/2642)
- [Issue: Add `tryUpdate()` method](https://github.com/onflow/cadence/issues/2700)
- [Issue: Flow CLI Contract to hex util](https://github.com/onflow/flow-cli/issues/1157)

## Prior Art

- [Proof of concept implementation](https://github.com/sisyphusSmiling/contract-updater)

## Questions & Discussion Topics

- What are the transaction limits and how do they affect `Updater` setup and execution of updates for large numbers of accounts?
    - Benchmarks would be helpful here
- What other use cases would this mechanism be useful for?
- How many community developers would use this service if offered?
    - We wouldn't make its use obligatory, of course, but would the community find this useful for their own Dev Ops post-spork?