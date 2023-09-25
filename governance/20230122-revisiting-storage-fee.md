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
3. minimum account balance however would increase 10 times such that minimum balance required is 0.01 FLOW
4. 90kb bonus storage would be given to each account, such that storage_available = 90kb + storage_capacity per the pricing function tabulated below



|                                                      **Current Prices**                                                     |                                                    **Proposed Prices**                                                   |
|:---------------------------------------------------------------------------------------------------------------------------:|:------------------------------------------------------------------------------------------------------------------------:|
| 100 MB per 1 reserved FLOW <br>10 MB per 0.1 reserved FLOW<br>1 MB per 0.01 reserved FLOW<br>100 kB per 0.001 reserved FLOW | 100 MB (+90kB bonus) per 100 reserved FLOW<br>10 MB (+90kB bonus) per 10 reserved FLOW<br>1 MB (+90kB bonus) per 1 reserved FLOW<br>10 kB (+90kB bonus) per 0.01 reserved FLOW |

Note that if the proposal is accepted and implemented, all account balance requirements will be calculated based on the prices recommended above and the byte-size of data stored by the account. Here are a few examples to explain how the minimum balance requirement will be calculated - 

•	0.01 FLOW provides 10kB storage capacity (at the new price rate) and a 90kB bonus, thus giving the account a storage capacity of 100kB. In other words, an account that stores < 100kB will be required to maintain a balance of 0.01 FLOW.

•	0.1 FLOW provides 100kB storage capacity (at the new price rate) and a 90kB bonus, thus a total storage capacity of 190kB.

•	0.11 FLOW provides 110kB storage capacity and a 90kB bonus, thus a total storage capacity of 200kB.

•	1 FLOW will provide 1MB storage capacity and a 90kB bonus, and so on.


Note that users would require a minimum account balance of 0.01 FLOW as part of the account-creation transaction (which is 10 times than today’s requirement), providing them with up to 100kB of storage capacity (due to the 90kB bonus); this is equivalent to what new accounts get today. As the minimum account balance corresponds to account-creation transactions, this change makes the network more robust to sybil attacks.

With the proposed change in the pricing, Flow would continue to remain ~20x and 140x cheaper than NEAR protocol’s and Solana’s storage prices. Nevertheless, with this increase, Flow would **make deliberate storage-exhaustion attacks more costly, reduce token liquidity to drive broader network benefits, and allow for a sustainable and regulated growth of state.**

Transitioning to the new storage pricing structure would be a multi-phase process requiring a change-management approach that minimizes user-impact, supports developer experience, and promotes network scalability. With community consultations prior to any protocol changes, the Flow core R&D team would implement the changes only after deliberate consideration, communication, testing, and impact-analysis.

## Final Word

Flow's core team is working towards **strengthening FLOW token economics**, and one of the ways to achieve this would be to **price FLOW’s use-cases for the value it generates** for the community vis-a-vis other networks. This proposal is meant to serve as a starting point for community discussions on how potential state scaling problem and storage-exhaustion attacks can be addressed economically with a revised storage pricing structure in the short-term. **The community is invited to share feedback on this post and participate in defining storage price changes** to help elevate Flow’s value capture and ultimately generate improved experience for developers and users on the platform.

## FAQs

1. **Why am I proposing this change?**
    
    The proposed change aims to incentivize judicious data storage, safeguard against state exhaustion, and enhance Flow's security by increasing attack costs.
    
2. **Why am I prioritizing fee increase among other ideas floating in the community?**
    
    There are many thoughts and considerations from the community members on revising various aspects of Flow’s storage framework. I believe that those must be published as FLIPs, discussed and voted upon. In view of the research and engineering costs behind some mature solutions however, I am proposing that a linear pricing based on byte-size be upheld and an increase in the price per data-unit be deliberated.
    
3. **If the proposal is passed, what might happen to an old account that has only 0.001 FLOW? Do accounts with low balances need additional deposits to meet the new minimum FLOW balance?**
    
    Yes, accounts falling below the minimum would need to make an additional deposit to meet the required balance; in this case an additional deposit of 0.099 FLOW would be required. Accounts exceeding storage limits per the revised fee structure would result in failed transactions, just like it happens right now.
    
4. **Can users unlock their accounts and add more FLOW?**
    
    Accounts below the minimum cannot perform any actions, even top-ups. Only third parties can top up such accounts. No edits, transactions, or account restoration can occur until the balance is increased. A full rollout plan will therefore be created in conjunction with developers to minimize disruptions.
    
5. **Can users swap USDC to FLOW on platforms like [Increment.fi](http://increment.fi/) to add FLOW to their accounts?**
    
    No, once again - users cannot self-top their accounts. Only third-party payers can add FLOW to accounts below the minimum.
    
6. **Who bears the increased costs - the user or the dApp?**
    
    Increased account creation costs would be borne by those paying for new accounts today, aka the dApps, exchanges, wallets, etc. Before the change goes into effect however, Flow Foundation would work with partners to determine the resources and support needed to ensure a smooth transition.
    
7. **How straightforward is the account replenishment process? Can dApps/wallets independently manage it?**
    
    The ease of the account replenishment procedure can vary among partners, based on their familiarity and background in developing on Flow. However, the Flow Foundation is dedicated to collaborating with partners to identify the necessary resources and assistance required for a seamless transition, aiming to mitigate any disruption for end users. Moreover, ample advance notice regarding this alteration will be given; it will not be an overnight switch.
    
8. **Could an increase in failed transactions pose a problem?**
    
    I am aware that this is a breaking change, and might result in specific transactions failing, particularly if dApps do not handle the process of account replenishment. However, this challenge will dissipate as dApps top-up the accounts with the required balances. Additionally, note that this challenge is not primarily linked to the alteration in storage fees, but rather pertains to an existing vulnerabilities within dApps - transactions can fail due to a variety of reasons even today, irrespective of these changes. Thus, as long as all parties exceed the minimum requirement, the occurrence of transaction failures should not increase or cause additional problems.
    
9. **Why am I not proposing an increase in the transaction fee first?** 
    
    The aim is to align storage fee changes with transaction fee adjustments, with a proposal for transaction fee alterations in the discussion stage ([FLIP link](https://forum.onflow.org/t/flip-74-revisiting-flow-transaction-fee/4497)).
    
10. **What happens when FLOW price goes up or down? Does this cause affordability challenges for dApps and wallets?**
    
    If FLOW market price changes substantially, another proposal adjustment may be initiated at that time. Based on the markets and exogenous factors, the [Tokenomics Working Group](https://github.com/onflow/twg) must continuously analyze the community’s experience and the inflection points that could make adoption challenging for folks, and accordingly propose changes.
    
11. **Could this change lead to greater security threats?**
    
    In fact, it's quite the opposite. As detailed in the FLIP document, the higher storage fees will serve as a deterrent against storage exhaustion attacks. The 100x fee would help economically deter any deliberate or inadvertent attacks, making them 100x less attractive or financially viable.
    
12. **What happens if projects fill random accounts with NFTs and exhaust space? Who bears the cost?**
    
    Transactions fail in this case, and no one pays for it.
    
13. **If passed, how soon will this be implemented?**
    
    The FLIP is actively under discussion and is anticipated to be put up for a community vote sometime around September/ October 2023. Following the voting process and approval of the community, the implementation of this change could take approximately 2-3 months.
    
14. **Can users completely destruct accounts and reclaim locked FLOW?**
    
    Today, users have the capability to erase data and unlock FLOW tokens, except that they cannot erase their account to retrieve the minimum balance. At present, with a mere 0.001 FLOW minimum balance, this seems reasonable. However, I understand that if the minimum account balance were to increase 100x, having the capability to deactivate and recover funds would become more desirable and I urge the community to take up the capability to recover storage space as a separate FLIP.
    
15. **How will users know if they have enough balance. In other words, how can users differentiate between total balance and withdraw-able balance?**
    
    The “availableBalance” is often obfuscated from UI by developers but can be obtained through Cadence on Flow CLI. UI updates for Flow port and dApps would help demonstrate “availableBalance” as the primary information.

## Resources

[Flow Storage Parameters](https://developers.flow.com/learn/concepts/storage#storage-parameters)

[Solana Storage Rent Economics](https://docs.solana.com/storage_rent_economics)

[NEAR Storage Staking](https://docs.near.org/concepts/storage/storage-staking#the-million-cheap-data-additions-attack)
