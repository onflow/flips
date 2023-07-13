---
status: draft 
flip: [PR number]
authors: Tarak Ben Youssef (tarak.benyoussef@dapperlabs.com) 
sponsor: 
updated: 2023-07-13 
---

# FLIP [PR number]: Replace the unsafeRandom function by random

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
and unbiasable by all Cadence code prior to the random function call. 

## User Benefit

Cadence uses the prefix `unsafe` to warn developers of the risks of using
the random function. Such risks are addressed by the FVM recent updates
and developers can safely rely on the random function. Removing the prefix
clarifies the possible confusion about the function safety.

## Design Proposal

1. as a first step:
  - rename Cadence's [runtime interface](https://github.com/onflow/cadence/blob/8a128022e0a5171f4c3a173911944a2f43548b98/runtime/interface.go#L107) `UnsafeRandom` to
  `Random`.
  - add a new Cadence function `fun random(): UInt64`, backed
  by a safe implementation in the FVM. `fun unsafeRandom(): UInt64` remains
  available to avoid immediate breaking changes. Note that both functions
  would be backed by the same safe FVM implementation.
2. as a second step, deprecate `fun unsafeRandom(): UInt64` as part of the
stable cadence release (Cadence v1.0).

### Drawbacks

Step 2. introduces a breaking change.

### Alternatives Considered

- `fun random<T: UnsignedInteger>(): T` can provide more flexibility
to developers. Any random output of type `T` can be built using `fun random(): UInt64`.
Adding a new built-in function can still be considered.
- an optional modulus can be added as a input `fun random([modulus; UInt64]): UInt64` to produce
a random number less than `modulus`. In particular, a safe implementation that
provides a uniformly distributed output, suitable for cryptographic applications
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

The step 2 of the #Design-proposal includes a breaking change. 

### User Impact

Please see #Design-proposal.

## Related Issues

As mentioned in #Best-Practices, the current proposal and the new FVM implementation
do not propose solutions for the transaction abortion issue. Solutions to abortion
such as commit-reveal schemes can be proposed in a separate FLIP.

## Questions and Discussion Topics

None