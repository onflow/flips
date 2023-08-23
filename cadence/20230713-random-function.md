---
status: proposed
flip: 120
authors: Tarak Ben Youssef (tarak.benyoussef@dapperlabs.com)
sponsor: 
updated: 2023-07-13
---

# FLIP 120: Rename unsafeRandom function to random

## Objective

The purpose of this FLIP is to:
- Rename the current `fun unsafeRandom(): UInt64` function to
`fun random(): UInt64`.
This can be done by introducing a new `random` function, and eventually
deprecating `unsafeRandom` (breaking change).
- Expand the current function to a more safe and convenient `fun random<T: UnsignedInteger>([modulo: T]): T`,
where `UnsignedInteger` covers all Cadence's fixed-size unsigned integer types, and `modulo` is an optional upper-bound argument.

## Motivation

### Function Rename

The Flow Virtual Machine (FVM) provides the implementation of `unsafeRandom` as part of the Flow protocol. 
The FVM implementation has been using the block hash as a source of entropy. 
This source can be manipulated by miners (i.e consensus nodes)
and should not be relied on to derive secure randomness, 
hence the `unsafe` suffix in the function name.

FVM is [undergoing changes](https://github.com/onflow/flow-go/pull/4498) that update the source of entropy
to rely on the secure distributed randomness generated within the Flow
protocol by [the random beacon](https://arxiv.org/pdf/2002.07403.pdf) component. 
The Flow beacon is designed to generate decentralized, unbiased, unpredictable and verifiable
randomness. 
Miners have negligible control to bias or predict the beacon
output.

Moreover, FVM is using extra measures to safely extend the secure source of entropy into
randoms in the transaction execution environment:
- FVM uses a unique diversified seed for each transaction execution.
- It uses a crypto-secure pseudo-random generator (PRG) to extend the seed entropy into a
deterministic sequence of randoms.
- FVM does not expose the PRG seed or state to the execution environment.

These measures make the `unsafeRandom` unpredictable to the transaction execution environment
and unbiasable by all transaction code prior to the random function call.

### function generalized header

Many applications require a random number less than an upper-bound `N` rather than a random number without constraints. For example, sampling a random element from an array requires picking a random index less than the array size. `N` is commonly called the modulo. In security-sensitive applications, it is important to maintain a uniform distribution of the random output. Returning the remainder of the division of a 64-bits number by `N` (using the modulo operation `%`) is known to result in a biased distribution where smaller outputs are more likely to be sampled than larger ones. This is known as the "modulo bias". There are safe solutions to avoid the modulo bias such as rejection sampling and large modulo reduction. Although these solutions can be implemented purely in Cadence, it is safer to provide the secure functions and abstract the complexity away from developers. This also avoids using unsafe methods. The FLIP suggests to add an optional unsigned-integer argument `N` to the `random` function. If `N` is provided, the returned random is uniformly sampled strictly less than `N`. The function errors if `N` is equal to `0`. If `N` is not provided, the returned output has no constraints.

A more convenient way of using `random` is to cover all fixed-size unsigned integer types (`UInt8`, `UInt16`, `UInt32`, `UInt64`, `UInt128`, `UInt256`, `Word8`, `Word16`, `Word32`, `Word64`).
The type applies to the optional argument `modulo` as well as the returned value. 
This would abstract the complexity of generating randoms of different types using 64-bits values as a building block. 
The new suggested function signature is therefore `fun random<T: UnsignedInteger>([modulo: T]): T`, 
where `T` can be any type from the above list.
Note that `UInt` is a variable-size type and is not supported by the function.

## User Benefit

Cadence uses the prefix `unsafe` to warn developers of the risks of using
the random function. 
Such risks are addressed by the FVM recent updates
and developers can safely rely on the random function. 
Removing the prefix clarifies the possible confusion about the function safety.

The generalized function signature offers safe and more flexible ways to use randomness. 
Without such change, developers are required to implement extra logic and take the risk of making mistakes.

## Design Proposal

1. As a first step:
  - Update Cadence's [runtime interface](https://github.com/onflow/cadence/blob/8a128022e0a5171f4c3a173911944a2f43548b98/runtime/interface.go#L107) `UnsafeRandom() (uint64, error)` to `ReadRandom(byte[]) error`.
  - Add a new Cadence function `fun random<T: UnsignedInteger>([modulo: T]): T`, backed by a safe FVM implementation. 
    `fun unsafeRandom(): UInt64` remains available to avoid immediate breaking changes. 
    Note that both functions will be backed by the same safe FVM implementation.
2. As a second step, deprecate `fun unsafeRandom(): UInt64` as part of the
Stable Cadence release (aka Cadence v1.0).

### Drawbacks

Step (2) in [Design Proposal](#design-proposal) introduces a breaking change.

### Alternatives Considered

The generalized function signature could be omitted from the proposal because it is possible to implement `fun random<T: UnsignedInteger>([modulus; T]): T` purely on Cadence using `fun random(): UInt64`. 
However, this requires developers to be familiar with safe low-level implementations and it may result in bugs and vulnerabilities. 
It is safer to have these tools provided natively by Cadence.

### Performance Implications

The `random` function using the optional modulo argument `N` is obviously slower than when `N` is not provided. The extra cost is required to provide the safety and uniformity of the output distribution.

### Dependencies

None

### Engineering Impact

The Cadence repository needs to implement the generalized function signature (optional argument and extra types).

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

```
let r1 = random<UInt8>()                    // r1 is of type `UInt8`
let r2 = random<Word16>()                   // r2 is of type `Word16`
let r3 = random<UInt128>(3)                 // r3 is of type `UInt128` and is strictly less than 3
let r4 = random<UInt128>(1 << 100)          // r4 is of type `UInt128` and is of at most 100 bits
let r5 = random<Word64>(Word64(r1))         // r5 is of type `Word64` and is strictly less than `r1`
let r6 = random<UInt64>(0)                  // panics
```

### Compatibility

The step 2 of the [Design Proposal](#design-proposal) includes a breaking change.

### User Impact

Please see [Design Proposal](#design-proposal) for details on user impact.

## Related Issues

As mentioned in [Best Practices](#best-practices) section, the current proposal and the new FVM implementation
do not propose solutions for the transaction abortion issue. Solutions to abortion
such as commit-reveal schemes can be proposed in a [separate FLIP](https://github.com/onflow/flips/pull/123).

## Questions and Discussion Topics

### Randomness in script execution

`executeScriptAtBlock` and `ExecuteScriptAtLatestBlock` are used to execute Cadence read-only code against the execution state at a past sealed block or the latest sealed blocked, respectively. 
The FVM implementation of `random` uses the transaction hash to diversify the random sequence per transaction. This does not add new entropy but prevents generating the same randoms for different transactions. For this reason, it is not possible to replicate Cadence's `random` behavior in scripts.
The FLIP suggests to use the same source of randomness for scripts as for transactions, and to diversify the random sequence per script using the hash of the script code.
Although scripts use the same randomness source as transactions to derive randoms, it is not possible to retrieve the same random numbers when calling `random` as a transaction or as a script, the additional diversifier being different (transaction hash vs script hash).


### Possible confusion with renaming

- concern: this change may communicate the wrong idea to Cadence users. Although from the protocol perspective randomness is no longer "unsafe", Cadence users will probably not consider it to be such. As far as they are concerned, the randomness (used naively) is still "unsafe" because it is still exploitable by a transaction that reverts to post-select a favorable result. Changing the name of the function to remove the unsafe prefix would suggest to developers that the function is "safe" in the sense that it is immune to this exploit, which it's not. Users can use certain patterns to make their randomness truly safe, but they need to structure their code in a specific way for this to work. The worry is that developers won't realize they need to do this without the built-in warning that a name like `unsafeRandom` provides. It's preferred to have the name indicate that the user needs to do something additional beyond just calling the function to get a random number; something like `revertibleRandom`, for example.

- possible answer:
  - Although any transaction with a random call can be reverted, some applications require randomness and have no incentive in reverting the result. Flow-core contracts for instance use randomness without reverting risks. A well-respected dapp also (assigning random NFTs for example) wouldn't want to add a reverting logic to their contract. For these applications, the new exposed randomness is considered safe. Adding `unsafe` would mean there is something wrong with the contract while there isn't.
  - Result abortion is an inherent property of atomic smart-contract platforms, rather than a property of specific functions like randomness. The non-safety comes from the natural language property rather than the randomness itself. We can argue that any logic written with Cadence is revertible, while the prefix `unsafe` or `revertible` is not added for other Cadence functions. Developers should be aware that a state change in a transaction is revertible, not only when it calls randomness (a deterministic game transaction can abort if the player isn't happy with the result for instance).
  - A language should try to provide as-safe-as-possible tools, but it can't guarantee that any program written in that language is totally safe. There is always some responsibility that falls on the language developer. It is possible to write very unsafe contracts and Cadence can't prevent it (using the cryptography functions as an example).


