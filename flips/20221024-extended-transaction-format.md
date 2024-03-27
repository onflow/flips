# Extended Transaction Format [DRAFT]

| Status        | Draft                                                          |
| :------------ | :------------------------------------------------------------- |
| **FLIP #**    | TBD                                                            |
| **Forum**     | TBD                                                            |
| **Author(s)** | Jeffrey Doyle (jeffrey.doyle@dapperlabs.com)                   |
| **Sponsor**   | Jeffrey Doyle (jeffrey.doyle@dapperlabs.com)                   |
| **Updated**   | 2022-10-24 

## Abstract

This FLIP proposes a change to Cadence, Flow Client Libraries and Wallets on Flow to increase the flexibility of how applications and wallets coordinate on determining the assets and accounts used in a transaction.

## Motivation

Often, transactions today are written with an understanding of where assets are in the accounts used in the transaction. Transactions are also written with a notion of what accounts a user controls, and how their wallet performs actions with that users assets.

As Flow progresses to allow assets to be stored wherever the account owner and wallet may choose, and for the wallet to use their preferred account setup to coordinate actions with the users assets, more flexibility in how transaction developers write their transactions is required.

## Design Proposal

This FLIP proposes that transactions should have any number of `prepare` statements and any number of `post` statements. Each prepare statement would be responsible for assigning values to variables defined in the transaction within the same `role` block.

Each prepare statement would have the ability to initialize transaction variables, but not read their values. Each prepare statement can only assign variables annotated within the same role block as itself. Since prepare statements can mutate execution state, the order of execution of each prepare statement must match the order they are defined in the cadence transaction.

A role would be assigned to a variable or prepare phase by being included in the same role block. Each signer / wallet involved in the transaction is assigned one of the roles defined in the transaction. Each transaction could have any number of roles for it's variables and prepare statements, but each variable and prepare statement can only be assigned to one role. Optionally, each signer / wallet can append a post statement to the transaction with whatever conditions they require.

Each signer / wallet is responsible for producing a prepare statement for their assigned role block using a non-empty set of accounts they control if the prepare statement is not already defined in the transaction. This way, the wallet can choose which accounts they need to use, and how they need to accomplish assigning values for the variables they're responsible for.

For example, a Cadence transaction might look like:

```cadence
// NFT Purchase & Transfer
transaction(nftID: UInt64, amount: UFix64) {

  role seller {
    let nft: @NonFungibleToken.NFT

    // <-- Seller wallet produces a prepare statement that will be inserted here
  }

  role buyer {
    let payment: @FlowToken.Vault
    let receiver: Capability<&{NFT.Receiver}> 

    // <-- Buyer wallet produces a prepare statement that will be inserted here
  }
 
  pre {
  	payment.balance == amount
  }

  execute {
    ...
  }

  post {
    ... // <-- Buyer produces a post statement to check that Buyer received the NFT
  }

  post {
    ... // <-- Seller produces a post statement to check that Seller received 20.0 FLOW
  }
}
```

Here, the transaction developer is declaring that there are two roles in the transaction, "Buyer" and "Seller". The transaction is effectively saying:

> "This transaction requires the Buyer to produce a vault of 20.0 FLOW tokens and a receiver capability to receive the NFT, and the Seller to put up the NFT corresponding to nftID".

Each wallet / signer for the Buyer and Seller would be responsible for producing the prepare statement assigned to them. This way, the transaction developer is able to declare what happens in the transaction without having to know where in the users accounts their assets are, or which Flow accounts are needed to be used to accomplish the transaction.

Each wallet / signer involved in the transaction benefits from having their own isolated prepare statement where they can engage with the accounts they control. This way, wallets do not need to concern themselves with the content of other prepare statements in the transaction where other AuthAccounts are made available, as the AuthAccounts the wallet controls are not impacted by them. 

In the case where the wallet produces the content of their assigned prepare statement, they benefit by not needing to be concerned about a malicious prepare statement written by a 3rd party having access to the AuthAccounts they control.

Flow's Client Libraries would need to be modified to support asking wallets / signers to produce the prepare statements they would like to use for a given transaction. The client libraries would also need to be modified to support declaring the role of each authorizer of the transaction, so the corresponding wallet / signer can understand which role they are acting as.

The Flow transaction data structure and Access Node API must be updated to coordinate which signature(s) for each authorizer correspond to which AuthAccount in each prepare statement of the transaction.

Here is an example of how FCL-JS might be used to execute such a transaction:

```javascript 
const txId = await fcl.mutate({
  cadence: `...`,
  authorizers: [
    appAuthzWallet({ role: “Seller” }), // Authorizer (wallet) for first prepare block (#Seller)
    fcl.currentUser.authz({ role: “Buyer” }) // Authorizer (wallet) for second prepare block (#Buyer)
  ]
})
```

## Considerations / Dependencies

This proposal involves complex changes to multiple areas of Flow, including:
- The Cadence language
- How Flow executes Cadence transactions
- The Flow Transaction data structure 
- Flow Access Node API
- Flow Client Library (FCL) & FCL wallet provider spec.
- Flow Wallets

The benefits of this proposal should be appropriately weighed against the effort required to implement and coordinate this proposals changes among the affected areas and parties.

For wallets, considerable complexity needs to be addressed in how the wallet can interpret a cadence transaction, and understand how to produce the cadence code required to properly assign the variables they are responsible for in their assigned prepare statement.

## Questions and Discussion Topics

- Do viable alternatives to this proposal exist that allow greater flexibility in how applications and wallets coordinate on determining the assets and accounts used in a transaction?
