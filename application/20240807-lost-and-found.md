---
status: proposed
flip: 281
authors: Navid TehraniFar (navid.tehranifar@flowfoundation.org)
sponsor: Greg Santos (greg.santos@flowfoundation.org)
updated: 2024-08-07
---

# FLIP 281: Integrating `LostAndFound` with Flow Port and Flow Wallet for Token Transfers

## Objective

When a Flow user wants to send a token to another account, the recipient's account must be initialized to receive that type and have sufficient storage. If these conditions are not met, the token transfer transaction will fail.

To address this issue, an intermediate contract can store the tokens until the recipient is ready to claim them. Austin Kline (@austinkline) from Flowty has developed a solution based on this concept called [LostAndFound](https://github.com/Flowtyio/lost-and-found). The sender can transfer tokens to the `LostAndFound` contract, and the recipient can claim them when they are online.

This FLIP proposes integrating the `LostAndFound` contract into the Flow Port and Flow Wallet products to mitigate account initialization and storage errors without altering the Flow protocol or Cadence.

## Motivation

Token transfers fail if the recipient's account is not initialized or lacks sufficient storage. This situation requires the recipient to initialize their account before the transaction can complete, negatively impacting the user experience. Our goal is to enhance the token transfer process by providing a solution that allows tokens to be stored until the recipient is ready to claim.

## User Benefit

Integrating `LostAndFound` with Flow Port and Flow Wallet ensures that token transfers will not fail due to the recipient's account not being initialized. Senders can transfer tokens to the `LostAndFound` contract, and recipients can claim them when they are online.

## Design Proposal

We propose integrating the `LostAndFound` contract, already [deployed](https://contractbrowser.com/A.473d6a2c37eab5be.LostAndFound) on mainnet, with Flow Port and Flow Wallet. The contract uses a type-based ticketing system with support for fee estimation. More details are available in the project's [repository](https://github.com/Flowtyio/lost-and-found).

#### Sending Tokens

When the user enters the recipient's address on Flow Port or Flow Wallet, a Cadence script will be executed that checks if the receiving side has initialized their account to receive that type and has enough storage. The result could be one of the following:

- **Initialized with sufficient storage**

    The transaction proceeds as usual.

- **Not initialized** or **Initialized but lacks storage**

    The amount of needed storage and required FLOW tokens are calculated and displayed to the sender. Upon confirmation, the tokens and required FLOW for storage will be deposited into the `LostAndFound` contract. The receiver can choose to claim the tokens when they are online.

#### Receiving Tokens

When a user logs in to Flow Port or Flow Wallet the following will happen:

1. A script will be executed to check if the user has any unclaimed tokens in the `LostAndFound` contract.

2. If the user has unclaimed tokens, they will be shown in the new "inbox" section. In this screen, the user can:

    - View a list of unclaimed tokens and the associated info.

    - Redeem one or multiple number of tokens. The user will be prompted to approve the transaction to claim the token(s). FLOW tokens held for storage will be transferred to the sender.

    - Discard one or multiple number of tokens. The user will be prompted to approve the transaction. The tokens will be destroyed and the sender will receive the FLOW locked for storage.

### Drawbacks

- This solution addresses token transfer issues at the application level and does not resolve the problem at the protocol level.

- Different platforms can choose to deploy their own `LostAndFound` contract, which may lead to fragmentation.

### Alternatives Considered

#### Using child accounts

Instead of a contract, the sender could create a child account for the recipient and transfer the tokens there. The recipient can later claim the account. However, the minimum storage balance cannot be claimed by the recipient as accounts cannot be destroyed on Flow. Additionally, the minimium storage balance might increase in the future, thus making this solution more costly for the sender and recipient.

#### Solving at Protocol Level

Several protocol-level solutions could address this issue, but they would require changes to the Flow protocol and/or Cadence. This FLIP avoids these changes by providing an application-level fix.

### Performance Implications

All the tokens are stored in a single universal `LostAndFound` contract. This can cause performance issues if the contract is used by a large number of users.

### Dependencies

This FLIP introduces the `LostAndFound` contract as a dependancy for the Flow Port and Flow Wallet products.

Flowty will continue to own the `LostAndFound` contract.

### Best Practices

This FLIP introduces a new best practice especially for wallet providers: always check if the recipient has initialized their account and has enough storage before sending tokens. If not, deposit the token(s) into `LostAndFound`. Documentation and guides will be updated accordingly, and other applications and wallets can adopt the necessary scripts and transactions.

### Examples

The `LostAndFound` [repository](https://github.com/Flowtyio/lost-and-found) includes examples for using the contract:

#### Scripts

* Estimating storage fee: [estimate_deposit_nft.cdc](https://github.com/Flowtyio/lost-and-found/blob/main/scripts/lost-and-found/estimate_deposit_nft.cdc)

* Fetching all unclaimed tokens info: [borrow_all_tickets.cdc](https://github.com/Flowtyio/lost-and-found/blob/main/scripts/lost-and-found/borrow_all_tickets.cdc)

#### Transactions

* Depositing a token: [try_send_multiple_example_nft.cdc](https://github.com/Flowtyio/lost-and-found/blob/main/transactions/example-nft/try_send_multiple_example_nft.cdc)

* Redeeming tokens based on type: [redeem_example_nft_all.cdc](https://github.com/Flowtyio/lost-and-found/blob/main/transactions/example-nft/redeem_example_nft_all.cdc)

### Compatibility

No compatibility issues are expected.

### User Impact

This will be released as a new feature in Flow Port and Flow Wallet.

## Related Issues

Supporting token transfers to uninitialized accounts natively is considered out of scope for this FLIP. This FLIP is an application-level solution for token transfer issues.

## Prior Art

Storage and account initialization are unique concepts of Flow.

## Questions and Discussion

1. Is the design outlined here sufficient to solve the problem? Are there better ways to address token transfer errors?

2. Does the `LostAndFound` contract meet the requirements outlined here?

3. Is the contract secure enough?

4. What are the performance implications of storing all tokens in a single contract?

5. How can users trust contract owners that there will be no rogue updates to the contract code that could affect locked assets?