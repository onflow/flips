---
Status: Proposed
Flip: 310
Authors: Vishal Changrani
Sponsor: Vishal Changrani
Updated: 2024-11-15
---

# FLIP 310: Removing Equilibrium as a multi-signer

## Proposal
Remove Equilibrium from the service account committee (multi-signer) as they no longer wish to continue to be signer.

_The removal is strictly because Equilibrium no longer wishes to be one of the multi-signer and for no other reason. Equilibrium continues to be a valued member of the Flow community._

## Implementation

The key index for equilibrium is 8 on the service account and 2 on the staking account.

The key will be removed by executing the following transaction:

### Transaction for service account

```
transaction {
    prepare(signer: auth(RevokeKey) &Account) {
        signer.keys.revoke(keyIndex: 8)
    }
}
```

### Transaction for staking account

```
transaction {
    prepare(signer: auth(RevokeKey) &Account) {
        signer.keys.revoke(keyIndex: 2)
    }
}
```

## Links
- [FLIP Tracker Issue](https://github.com/onflow/flips/issues/310)
- [Forum post](https://forum.flow.com/t/flip-310-removing-equilibrium-as-a-multi-signer/6782)
