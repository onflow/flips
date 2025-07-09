---
status: Released
flip: 321
authors: Layne Lafrance 
sponsor: Vishal Changrani 
updated: 2025-04-15
---

# FLIP 321: Update service account signers

## Objective

The service account would benefit from additional redundancy. In order to achieve that redundancy the governance committee would like to add me as an additional external signer. This involves removing my keys from the existing setup and adding a new key to the service account. This will ensure more signers are available to review and sign for service account multi-sig operations. 

## Motivation

Redundancy in decentralized systems is a critical component of providing reliable security and service even in times of potential network degradation. This change will improve signer redundancy. 

Users will not be impacted by this change. It will potentially speed up future service account signings to have more signers available. 

## User Benefit

The entire Flow ecosystem benefits from reliable, timely governance. The more the governance committee can do to keep things running smoothly and on-schedule - via more signers available to sign key changes at the right time - the easier things run overall. 

This would also represent further community representation in Flow's governance structures. 

## Design Proposal

### Transaction to Add New Key (+ revoke old)

https://github.com/onflow/flow-core-contracts/blob/master/transactions/accounts/add_key.cdc

https://github.com/onflow/flow-core-contracts/blob/master/transactions/accounts/revoke_key.cdc

### Drawbacks

None - Because multiple signers already provide redundancy on my existing key, the addition of this new key only makes more signers available to us with no additional costs or changes to the existing setup. 

### Alternatives Considered

Any other community members who feel strongly about Flow's governance and have an observable track record contributing to the ecosystem should also be considered as additional signers. 

### Performance Implications

This change will not impact protocol performance. 
The only measurable improvement would be reduced time to compete multi-signer signings when required. 

### Dependencies

* Dependencies: does this proposal add any new dependencies to Flow?

No 

* Dependent projects: are there other areas of Flow or things that use Flow 
(Access API, Wallets, SDKs, etc.) that this affects? 

Service Account 

* How have you identified these dependencies and are you sure they are complete? 
If there are dependencies, how are you managing those changes?

@vishalchangrani will be verifying with SRE to confirm no other dependencies exist for the existing keys to be removed. 

### Engineering Impact

* Do you expect changes to binary size / build time / test times?

No
 
* Who will maintain this code? Is this code in its own buildable unit? 
Can this code be tested in its own? 

The transaction to revoke key has been tested.
Revoking a key on the service account has been successfully done in the past.

* Is visibility suitably restricted to only a small API surface for others to use?

No - the change here is to the service account and will be visible to all users.

### Best Practices

* Does this proposal change best practices for some aspect of using/developing Flow? 
How will these changes be communicated/enforced?

N/A

### Tutorials and Examples

N/A 

### Compatibility

* Does the design conform to the backwards & forwards compatibility [requirements](../docs/compatibility.md)?

Yes

* How will this proposal interact with other parts of the Flow Ecosystem?
    - How will it work with FCL?
    - How will it work with the Emulator?
    - How will it work with existing Flow SDKs?

This proposal will not change how these services interact with the service account. 

### User Impact

* What are the user-facing changes? How will this feature be rolled out?

None 

## Related Issues

* What related issues do you consider out of scope for this proposal, but could be addressed independently in the future?

Adding more signers to the service account which is always an option and something that should be considered following consistent contributions from community members. 

## Prior Art

https://github.com/onflow/flips/issues/312
https://github.com/onflow/flips/issues/310 

## Questions and Discussion

Please bring any questions / concerns to this issue

## Transactions

Following are the transactions submitted by the service committee to implement this FLIP and FLIP 326,

Service Account: https://www.flowscan.io/tx/02841acc58dd74fb57ee4eb89ef14ccce19bc120684f64df15bcc806b978eddd
Staking Account: https://www.flowscan.io/tx/5fc1215ed2d8be18c46de837d64999b5f85898b8845bdcce271e382d49f99020