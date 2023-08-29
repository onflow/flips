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

As implied above, this contract and update service would allow developers to configure their update deployment onchain

## Design Proposal

### Overview

### Implementation Details

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

- 

### Drawbacks

- All contracts would need to be either a) core contracts or b) owned by the updater

### Considered Alternatives

### Performance Implications

### Best Practices

### Examples

### Compatibility

### User Impact

## Related Issues

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