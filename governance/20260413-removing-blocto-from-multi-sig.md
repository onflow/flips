---
Status: Released
Flip: 365
Authors: Vishal Changrani
Sponsor: Vishal Changrani
Updated: 2026-04-22
---

# FLIP 365: Removing Blocto as a multi-signer

## Proposal
Remove Blocto from the service account committee (multi-signer) as they are no longer an active member of the Flow community.

## Implementation

The key index for Blocto is 7 on the service account and 1 on the staking account.

The key will be removed by executing the following transaction:

### Transaction for service account

```
transaction {
    prepare(signer: auth(RevokeKey) &Account) {
        signer.keys.revoke(keyIndex: 7)
    }
}
```

### Transaction for staking account

```
transaction {
    prepare(signer: auth(RevokeKey) &Account) {
        signer.keys.revoke(keyIndex: 1)
    }
}
```

## Links
- [FLIP Tracker Issue](https://github.com/onflow/flips/issues/365)

## Implemented
- Transaction for service account - https://www.flowscan.io/tx/42b9c46973536d3ed63ccae0a5db7787a8606935bdf149e912999203f4a779ad
- Transaction for staking account - https://www.flowscan.io/tx/fdbb81f16bbb4ee05e72a9dec40ddb24d5ce0d332078302b76b7e726e9f6bdeb
