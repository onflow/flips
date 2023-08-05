---
status: proposed
flip: 120
authors: Tarak Ben Youssef (tarak.benyoussef@dapperlabs.com)
sponsor: 
updated: 2023-07-13
---

# FLIP 120: Rename unsafeRandom function to random

## Objective

The purpose is to rename the current `fun unsafeRandom(): UInt64` function to
`fun random(): UInt64`.
This can be done by introducing a new `random` function, and eventually
deprecating `unsafeRandom` (breaking change).

## Motivation

The Flow Virtual Machine (FVM) provides the implementation of `unsafeRandom`
as part of the Flow protocol. The FVM implementation has been using the block hash as
a source of entropy. This source can be manipulated by miners (i.e consensus nodes)
and should not be relied on to derive secure randomness, hence the `unsafe`
suffix in the function name.

FVM is [undergoing changes](https://github.com/onflow/flow-go/pull/4498) that update the source of entropy
to rely on the secure distributed randomness generated within the Flow
protocol by [the random beacon](https://arxiv.org/pdf/2002.07403.pdf) component. The Flow beacon
is designed to generate decentralized, unbiased, unpredictable and verifiable
randomness. Miners have negligible control to bias or predict the beacon
output.

Moreover, FVM is using extra measures to safely extend the secure source of entropy into
randoms in the transaction execution environment:
- FVM uses a unique diversified seed for each transaction execution.
- It uses a crypto-secure pseudo-random generator (PRG) to extend the seed entropy into a
deterministic sequence of randoms.
- FVM does not expose the PRG seed or state to the execution environment.

These measures make the `unsafeRandom` unpredictable to the transaction execution environment
and unbiasable by all transaction code prior to the random function call.

## User Benefit

Cadence uses the prefix `unsafe` to warn developers of the risks of using
the random function. Such risks are addressed by the FVM recent updates
and developers can safely rely on the random function. Removing the prefix
clarifies the possible confusion about the function safety.

## Design Proposal

1. As a first step:
  - rename Cadence's [runtime interface](https://github.com/onflow/cadence/blob/8a128022e0a5171f4c3a173911944a2f43548b98/runtime/interface.go#L107) `UnsafeRandom` to
  `Random`.
  - add a new Cadence function `fun random(): UInt64`, backed
  by a safe implementation in the FVM. `fun unsafeRandom(): UInt64` remains
  available to avoid immediate breaking changes. Note that both functions
  would be backed by the same safe FVM implementation.
2. As a second step, deprecate `fun unsafeRandom(): UInt64` as part of the
stable cadence release (Cadence v1.0).

### Drawbacks

Step (2) in [Design Proposal](#design-proposal) introduces a breaking change.

### Alternatives Considered

- `fun random<T: UnsignedInteger>(): T` can provide more flexibility
to developers. However, any random output of type `T` can be built using `fun random(): UInt64`.
Adding a new built-in function can still be considered.
- An optional modulus can be added as a input `fun random([modulus; UInt64]): UInt64` to produce
a random number less than `modulus`. In particular, a safe implementation that
provides a uniformly distributed output (reducing the modulo bias), suitable for cryptographic applications
could be added. However, such implementation can be built using `fun random(): UInt64`.

### Performance Implications

None

### Dependencies

None

### Engineering Impact

None

### Best Practices

It is important to note that while the new implementation provides
safe randomness that cannot be biased or predicted by the network miners
or by the transaction execution prior to the call (for instance a transaction that calls
`random()` multiple times in order to predict the future random output), developers
should still be mindful about using the function very carefully.

In particular, a transaction sender can atomically abort the transaction execution
and revert all its state changes if they judge the random output is not favorable **after it is
revealed**. The function is not immune to post-selection manipulation where a random number is
sampled and then rejected.
As an example, imagine a transaction
that calls an on-chain casino contract, that rolls a dice to find out if the
transaction sender wins. The transaction can be written so that it triggers an
error if the game outcome is a loss.
Developers writing a similar casino contract should be aware of the transaction
abortion scenario. This limitation is inherent to any smart contract platform that allows transactions to roll back atomically and cannot be solved through secure randomness alone.

### Tutorials and Examples

The `random` function can be used exactly the same way as
`unsafeRandom`.

### Compatibility

The step 2 of the [Design Proposal](#design-proposal) includes a breaking change.

### User Impact

Please see [Design Proposal](#design-proposal) for details on user impact.

## Related Issues

As mentioned in [Best Practices](#best-practices) section, the current proposal and the new FVM implementation
do not propose solutions for the transaction abortion issue. Solutions to abortion
such as commit-reveal schemes can be proposed in a separate FLIP.

## Questions and Discussion Topics

### Randomness in script execution

`executeScriptAtBlock` and `ExecuteScriptAtLatestBlock` are used to execute Cadence read-only code against the execution state at a past sealed block or the latest sealed blocked, respectively. With the entropy being extracted from the protocol state, `executeScriptAtBlock` may require reading the protocol state at a past block, which is not guaranteed to be stored by Flow nodes. Moreover, the FVM implementation of `random` requires a transaction hash which is not possible with script execution (scripts are not executed as part of a transaction). For these reasons, it is not possible to replicate Cadence's `random` behavior in scripts. It is not possible to retrieve the same random numbers when calling `random` as a transaction or as a script. 

We can define an alternative behavior for `random` in scripts. These following options are possible:
 1. panic
 2. return a constant (zero value of the type)
 3. return pseudo-random numbers using a weak source of entropy

Option (1) can cause confusion because valid Cadence code would fail when run as a script. Option (3) can also cause confusion as the returned numbers may look "random" while they are different than the numbers returned by a transaction, and are therefore not useful for users. (2) avoids panicking and may remind users that randomness doesn't work properly on scripts. This FLIP suggests to choose option (2).

### Possible confusion with renaming

- concern: this change may communicate the wrong idea to Cadence users. Although from the protocol perspective randomness is no longer "unsafe", Cadence users will probably not consider it to be such. As far as they are concerned, the randomness (used naively) is still "unsafe" because it is still exploitable by a transaction that reverts to post-select a favorable result. Changing the name of the function to remove the unsafe prefix would suggest to developers that the function is "safe" in the sense that it is immune to this exploit, which it's not. Users can use certain patterns to make their randomness truly safe, but they need to structure their code in a specific way for this to work. The worry is that developers won't realize they need to do this without the built-in warning that a name like unsafeRandom provides. It's preferred to have the name indicate that the user needs to do something additional beyond just calling the function to get a random number; something like revertibleRandom, for example.

- possible answer:
  - Although any transaction with a random call can be reverted, some applications require randomness and have no incentive in reverting the result. Flow-core contracts for instance use randomness without reverting risks. A well-respected dapp also (assigning random NFTs for example) wouldn't want to add a reverting logic to their contract. For these applications, the new exposed randomness is considered safe. Adding `unsafe` would mean there is something wrong with the contract while there isn't.
  - Result abortion is an inherent property of atomic smart-contract platforms, rather than a property of specific functions like randomness. The non-safety comes from the natural language property rather than the randomness itself. We can argue that any logic written with Cadence is revertible, while the prefix `unsafe` or `revertible` is not added for other Cadence functions. Developers should be aware that a state change in a transaction is revertible, not only when it calls randomness (a deterministic game transaction can abort if the player isn't happy with the result for instance).
  - A language should try to provide as-safe-as-possible tools, but it can't guarantee that any program written in that language is totally safe. There is always some responsibility that falls on the language developer. It is possible to write very unsafe contracts and Cadence can't prevent it (using the cryptography functions as an example).


