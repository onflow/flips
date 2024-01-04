---
status: approved 
flip: 163
authors: Bjarte Karslen (bjartek@find.xyz), Jeffrey Doyle (jeffrey.doyle@dapperlabs.com), Tom Haile (tom.haile@dapperlabs.com)
sponsor: Jeffrey Doyle (jeffrey.doyle@dapperlabs.com)    
updated: 2023-04-06
---

# FLIP 163: Interaction Template Cadence Doc (v1.0.0)

## Objective

This FLIP proposes a new standard for how Cadence transaction and script developers can declare static metadata about a transaction or script, in a machine parsable way, with the goal of this metadata to aid in the generation of a corresponding Interaction Template for the transaction or script.

This standard is to be called "Interaction Template Cadence Doc".

## Motivation

In order to increase adoption of Interaction Templates across the Flow ecosystem, the creation of developer friendly tools and processes such as this standard for the generation of Interaction Templates is required.

## User Benefit

Developers benefit from Interaction Template Cadence Doc by having an easy way to declare static metadata for their transactions and scripts. This allows them to store static metadata alongside the Cadence code for the transaction or script in the Interaction Template Cadence Doc comment. This way, developers can iterate on their transactions and scripts, and only need to declare their metadata once while being able to change it as required or desired.

With the creation of tooling for generating Interaction Templates through parsing Interaction Template Cadence Doc, automated ways for creating Interaction Templates can be created; such as, Flow CLI integration for generating an Interaction Template from a cadence transaction or script file that has Interaction Template Cadence Doc, or a Flow CLI file watcher mechanism which autogenerates a transaction or scripts corresponding Interaction Template every time the transaction or script with Interaction Template Cadence Doc changes.

## Design Proposal

### Inside a Cadence transaction or script

Interaction Template Cadence Doc can be included in a Cadence transaction or script, below all transaction/script imports and above the `transaction` or `main` function declaration, with no new line in between. This documentation format leverages existing Cadence support and parsing of pragma.

Interaction Template Cadence Doc supports the following format:

- `#interaction()` declares the Interaction Template Cadence Doc pragma. The metadata will be enclosed within.

- `version` the version of FLIX, needed to understand the structure to be generated. 

- `title` the title of the template, needs to be in the language declared by `language`

- `language` declares the default BCP-47 language tag for all messages in the Interaction Template Cadence Doc (defaults to 'en-US').

`#interaction-param-<name>` pragma includes information about a transaction/script parameters, contains same as interaction metadata for the interaction. 
- `title` the title of the field a human readable short message to associate with the parameter
- `description` optional description of the field a human readable message to associate with the parameter
- `language` declares the default BCP-47 language tag for all messages in the Interaction Template Cadence Doc (defaults to 'en-US').

This is an example of Interaction Template Cadence Doc for a transaction:

```cadence
import "FungibleToken"
import "FlowToken"

#interaction(
    version: "1.1.0",
    title: "Transfer Tokens",
    description: "Transfer tokens from one account to another",
    language: "en-US",
)

#interaction-param-amount(
    title: "Amount", 
    description: "Amount of Flow to transfer"
    language: "en-US",
)

#interaciton-param-to(
    title: "Reciever", 
    description: "Destination address to receive Flow Tokens"
    language: "en-US",
)

transaction(amount: UFix64, to: Address) {

  let vault: @FungibleToken.Vault

  prepare(signer: AuthAccount) {
    self.vault <- signer
      .borrow<&{FungibleToken.Provider}>(from: /storage/flowTokenVault)!
      .withdraw(amount: amount)
  }

  execute {
    getAccount(to)
      .getCapability(/public/flowTokenReceiver)!
      .borrow<&{FungibleToken.Receiver}>()!
      .deposit(from: <-self.vault)
  }
}
```

This is an example of Interaction Template Cadence Doc for a script:

```cadence
import "FungibleToken"
import "FlowToken"
#interaction(
    version: "1.1.0",
    title: "Flow Token Balance",
    description: "Get account Flow Token balance",
    language: "en-US",
)

#interaction-param-address(
    title: "Address", 
    description: "Get Flow token balance of Flow account",
    language: "en-US",
)

pub fun main(address: Address): UFix64 {
    let account = getAccount(address)
    let vaultRef = account.getCapability(/public/flowTokenBalance).borrow<&FlowToken.Vault{FungibleToken.Balance}>() ?? panic("Failed to borrow Flow Token balance reference")

    return vaultRef.balance
}

```

### Drawbacks

- Since Interaction Template Cadence Doc lives inside the transaction/script, it unnecessarily persists information on the blockchain that is not relevant to the execution of the transaction itself.

### Dependencies

- Flow Interaction Templates
  - Any breaking changes to Interaction Templates must have a corresponding update to Interaction Template Cadence Doc to accommodate them.

### Engineering Impact

- Interaction Template Cadence Doc will be maintained by those working on Interaction Templates.

### Best Practices

- Interaction Template Cadence Doc will need to be added to tutorials and documentation provided to developers so they can use it when creating transactions and scripts on Flow.

### Compatibility

- Interaction Template Cadence Doc will need to maintain compatibility with the latest specification of the Flow Interaction Template data structure.

## Prior Art

- Interaction Template Cadence Doc is built to resemble best practices from Javadoc and JSDoc.

## Questions and Discussion Topics

- Are there other viable formats for declaring static metadata for a transaction or script?
