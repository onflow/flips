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