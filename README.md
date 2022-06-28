# FLIPS
FLIPS are Flow Improvement Proposals. Each FLIP describes a protocol change, governance change, or application level standard meant to improve the Flow protocol or ecosystem. Proposals can be submitted by anyone, and should follow the [TDB] style guide for each category.

This repository will hold FLIPS to separate them from the [onflow/flow](https://github.com/onflow/flow/tree/master/flips) repository. This is currently a work in progress, and legacy flips are all in the [flips directory](https://github.com/onflow/flips/tree/main/flips). These flips will be moved into the appropriate sub-directories over time.

## Application
Application FLIPs are standards for applications built on top of FLOW. This could be token standards, contract interface standards, common design patterns that can benefit from social consensus etc. Application standards should not create protocol changes in themselves, and if they rely on any new protocol features that feature should be written up in its own protocol flip.

## Governance
Governance FLIPs are proposals to take governance actions on the network, for instance changing staking rules, admitting new node operators to the whitelist, adjusting fees, or taking various actions with the the service account.

## Protocol
Protocol FLIPs affect the core Flow protocol. This may include items such as: new algorithms which are required for any flow client to work on the network, payload API changes, cryptographic methods, etc.

Cadence changes will currently fall under Protocol flips because they are tightly coupled with the FVM.
