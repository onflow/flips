---
Status: Proposed
Flip: 326
Authors: Vishal Changrani
Sponsor: Vishal Changrani
Updated: 2025-04-12
---

# FLIP 326: Removing Ichi as a multi-signer

## Proposal

As part of maintaining an effective and decentralized multi-signing process, this FLIP proposes the removal of Ichi from the service committee.
The signer has not participated in more than 10 consecutive multi-signing meetings, which is in violation of the participation [policy](https://github.com/onflow/service-account/pull/370/files#diff-b335630551682c19a781afebcf4d07bf978fb1f8ac04c6bf87428ed5106870f5R18) outlined in the service account repository.

> This proposal is intended to uphold the accountability and continuity of the committee and is not a reflection on the individual’s contributions to the Flow ecosystem. Ichi remains a valued member of the Flow community, and we appreciate their past involvement and continued support.

## Implementation

The key index for Ichi is [9](https://github.com/onflow/service-account/blob/main/flow.json#L25-L34) on the service account and [3](https://github.com/onflow/service-account/blob/main/flow-staking.json#L25-L37) on the staking account.

The key will be removed by executing the following transaction:

### Transaction for service account

```
transaction {
    prepare(signer: auth(RevokeKey) &Account) {
        signer.keys.revoke(keyIndex: 9)
    }
}
```

### Transaction for staking account

```
transaction {
    prepare(signer: auth(RevokeKey) &Account) {
        signer.keys.revoke(keyIndex: 3)
    }
}
```

## Links
- [FLIP Tracker Issue](https://github.com/onflow/flips/issues/326)