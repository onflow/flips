---
Status: Released
Flip: 312
Authors: Vishal Changrani
Sponsor: Vishal Changrani
Updated: 2024-11-27
---

# FLIP 312: Removing Animoca as a multi-signer

## Proposal
Remove Animoca from the service account committee (multi-signer) as they no longer wish to continue to be signer.

_The removal is strictly because Animoca no longer wishes to be one of the multi-signer and for no other reason. Animoca continues to be a valued member of the Flow community._

## Implementation

The key index for Animoca is [10](https://github.com/onflow/service-account/blob/main/flow.json#L45-L49) on the service account and [4](https://github.com/onflow/service-account/blob/main/flow-staking.json#L57-L62) on the staking account.

The key will be removed by executing the following transaction:

### Transaction for service account

```
transaction {
    prepare(signer: auth(RevokeKey) &Account) {
        signer.keys.revoke(keyIndex: 10)
    }
}
```

### Transaction for staking account

```
transaction {
    prepare(signer: auth(RevokeKey) &Account) {
        signer.keys.revoke(keyIndex: 4)
    }
}
```

## Links
- [FLIP Tracker Issue](https://github.com/onflow/flips/issues/312)
- [Forum post](https://forum.flow.com/t/flip-312-removing-animoca-as-a-multi-signer/6844)
- [Transaction 1](https://www.flowscan.io/tx/3969c4e8172afe32866c40dd16dbe3e591de42d336db32739c376d0351044434)
- [Transaction 2](https://www.flowscan.io/tx/c1b06e5a3a52c4bc526aac42189fe12ae8a7983752fe40d5560d670cb7ca06f1)
