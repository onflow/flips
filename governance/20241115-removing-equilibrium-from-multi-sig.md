---
Status: Released
Flip: 310
Authors: Vishal Changrani
Sponsor: Vishal Changrani
Updated: 2024-11-27
---

# FLIP 310: Removing Equilibrium as a multi-signer

## Proposal
Remove Equilibrium from the service account committee (multi-signer) as they no longer wish to continue to be signer.

_The removal is strictly because Equilibrium no longer wishes to be one of the multi-signer and for no other reason. Equilibrium continues to be a valued member of the Flow community._

## Implementation

The key index for equilibrium is [8](https://github.com/onflow/service-account/blob/main/flow.json#L25-L29) on the service account and [2](https://github.com/onflow/service-account/blob/main/flow-staking.json#L25-L30) on the staking account.

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
- [Transaction 1](https://www.flowscan.io/tx/3969c4e8172afe32866c40dd16dbe3e591de42d336db32739c376d0351044434)
- [Transaction 2](https://www.flowscan.io/tx/c1b06e5a3a52c4bc526aac42189fe12ae8a7983752fe40d5560d670cb7ca06f1)