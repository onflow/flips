---
status: proposed
flip: GOV-5
authors: Kshitij Chaudhary (kshitij.chaudhary@dapperlabs.com)
sponsors: Kshitij Chaudhary (kshitij.chaudhary@dapperlabs.com)
updated: 2023-01-22
---

# FLIP GOV-5: Revisiting Flow storage minimum account balance

This FLIP is being discussed [here](https://forum.onflow.org/t/flip-66-revisiting-flow-storage-minimum-account-balance/4241).

## Introduction

Flow incentivizes mindful use of on-chain storage through economic means.
Depending on the amount of data that an account holds, there is a minimum FLOW token balance that the account must maintain.
[To bootstrap an account](https://developers.flow.com/learn/concepts/storage#storage-parameters) today for instance, users are required to have a minimum deposit of 0.001 FLOW as part of the account-creation transaction, which corresponds to an initial storage capacity of 100kB.
While Flow storage pricing was kept low in its early days to promote network trial-ability without financial concerns, as the chain matures and usage-patterns evolve, it is important to revisit the pricing from time to time.
If storing data is priced too low, the platform could potentially become vulnerable to storage exhaustion attacks or make rapid state growth unsustainable.
This FLIP aims at gathering community feedback on revisiting storage pricing such that it reduces attack surfaces and promotes sustainable state growth.

Importantly, storage pricing on Flow is not intended in any way to limit state growth that emanates from massive network growth, higher resource utilization and rising developer and user activity.
It is only aimed at avoiding “state bloating” - a wasteful uncontrolled capture of storage capacity, and to prevent storage exhaustion attacks.
Additionally, as storage fees are established not as “fees” but rather a minimum balance check, to avoid confusion, we use the term “storage pricing” to allude to the minimum balance requirement.

## Problem Statement

***Storage pricing must prevent storage-exhaustion attacks, allow for massive state growth, and steer mass adoption and scalability.***

While storage price introduced some form of payment structure to modify behavior, today, the price itself is too low to incentivize developers to make rational data-storage decisions. As storage prices today are a constant multiplier of data-size and not based on data-staleness (frequency of access), once users store data they have little incentives to delete the stale data that continues to be stored in the state forever. Additionally, low minimum balance requirements could allow malicious actors to constantly send write-heavy transactions and bloat the state at a low cost. Today, Flow protocol has no policy to prevent such an act. To economically deter such deliberate or accidental storage-exhaustion attacks, minimum balance requirements need to grow multifold.

With over 5 billion internet users today and Flow’s focus on mainstream adoption, storage on Flow must also be prepared for significant scale. A billion accounts on Flow storing an average of at least 1 MB of data each would require 1 Petabyte of storage in the future. While it is imperative to eventually move current state from memory to disc to allow for such large state future, it is also crucial to incentivize users to economize on storage space and make conscientious decisions about occupying the available storage capacity.

## Proposed Solution

### Objective

**This FLIP primarily focuses on economic solutions, i.e. storage price alterations** **to incentivize users to thoughtfully make data-storage decisions** **and disincentivize any storage-exhaustion attacks.** Effectively, storage pricing must prevent ‘*tragedy of the commons’* i.e. over-utilization of finite network resources to the extent of exploitation, and allow for a sustainable and regulated state growth.

### Proposal

Different L1 blockchains have implemented varying structures to deal with potential state bloating issues, but generally have priced storage much higher than Flow. Solana’s [rent model](https://docs.solana.com/storage_rent_economics) requires a minimum balance at the rate of ~7 SOL per MB of data (US$168 for 1MB storage, at US$24/SOL exchange rate), else users are required to pay rent for storing data on-chain at the end of every epoch. At the prescribed rate, **Solana’s storage price is about 14,000x more expensive than Flow** (US$0.012 for 1MB storage at US$1.2/FLOW exchange rate). NEAR protocol employs a [storage-staking mechanism](https://docs.near.org/concepts/storage/storage-staking#the-million-cheap-data-additions-attack) that requires accounts owning smart contracts to stake (lock) tokens according to the amount of data stored in that smart contract. For 100KB of storage occupied, the protocol requires 1 NEAR token to be staked, or ~$2.6 for 100KB storage at $2.6/NEAR exchange rate, making **NEAR ~2,100x more expensive than Flow** (0.001 FLOW per 100KB requirement).

Given that the state of Flow needs to be stored only across a handful of execution nodes (7 currently), Flow is less challenged by potential state growth challenges and could arguably choose a lower storage price compared to competitors like Solana (3400+ validators) and NEAR (100 validators).

**Thus, it is proposed that**

1. storage pricing function would remain a constant multiple of the byte-size of the data to be stored on-chain
2. pricing (storage capacity per reserved Flow) would increase 100 times, as tabulated below.
3. minimum account balance however would increase 10 times such that minimum storage capacity reduces from 100kB to 10kB and the minimum balance required changes from 0.001 FLOW providing 100kB storage capacity, to 0.01 FLOW providing 10kB capacity (at a 100x multiple of the current pricing).

|                                                      **Current Prices**                                                     |                                                    **Proposed Prices**                                                   |
|:---------------------------------------------------------------------------------------------------------------------------:|:------------------------------------------------------------------------------------------------------------------------:|
| 100 MB per 1 reserved FLOW <br>10 MB per 0.1 reserved FLOW<br>1 MB per 0.01 reserved FLOW<br>100 kB per 0.001 reserved FLOW | 100 MB per 100 reserved FLOW<br>10 MB per 10 reserved FLOW<br>1 MB per 1 reserved FLOW<br>10 kB per 0.01 reserved FLOW |

If the proposal is accepted and implemented, users would require a minimum account balance of 0.01 FLOW as part of the account-creation transaction, providing them with up to 10kB of storage capacity, which is what >95% of all accounts use today. As the minimum account balance corresponds to account-creation transactions, this change makes the network more robust to sybil attacks. If the storage used by an account exceeds 10kB, account balance requirements will be calculated based on the prices recommended above and the byte-size of data stored by the account.

With the proposed change in the pricing, Flow would continue to remain ~20x and 140x cheaper than NEAR protocol’s and Solana’s storage prices. Nevertheless, with this increase, Flow would **make deliberate storage-exhaustion attacks more costly, reduce token liquidity to drive broader network benefits, and allow for a sustainable and regulated growth of state.**

Transitioning to the new storage pricing structure would be a multi-phase process requiring a change-management approach that minimizes user-impact, supports developer experience, and promotes network scalability. With community consultations prior to any protocol changes, the Flow core R&D team would implement the changes only after deliberate consideration, communication, testing, and impact-analysis.

## Final Word

Flow's core team is working towards **strengthening FLOW token economics**, and one of the ways to achieve this would be to **price FLOW’s use-cases for the value it generates** for the community vis-a-vis other networks. This proposal is meant to serve as a starting point for community discussions on how potential state scaling problem and storage-exhaustion attacks can be addressed economically with a revised storage pricing structure in the short-term. **The community is invited to share feedback on this post and participate in defining storage price changes** to help elevate Flow’s value capture and ultimately generate improved experience for developers and users on the platform.

## Resources

[Flow Storage Parameters](https://developers.flow.com/learn/concepts/storage#storage-parameters)

[Solana Storage Rent Economics](https://docs.solana.com/storage_rent_economics)

[NEAR Storage Staking](https://docs.near.org/concepts/storage/storage-staking#the-million-cheap-data-additions-attack)
