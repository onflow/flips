# Interaction Template CadenceDoc (v1.0.0)

| Status        | Proposed                                                                        |
| :------------ | :------------------------------------------------------------------------------ |
| **FLIP #**    | TBD                                                                             |
| **Forum**     | TBD                                                                             |
| **Author(s)** | Bjarte Karslen (bjartek@find.xyz), Jeffrey Doyle (jeffrey.doyle@dapperlabs.com) |
| **Sponsor**   | Jeffrey Doyle (jeffrey.doyle@dapperlabs.com)                                    |
| **Updated**   | 2023-04-06                                                                      |

## Objective

This FLIP proposes a new standard for how Cadence transaction and script developers can declare static metadata about their transactions or scripts, in a machine parsable way, with the goal of this metadata to aid in the generation of a corresponding Interaction Template for the transaction or script.

This standard is to be called "CadenceDoc"

## Motivation

In order to further adoption of Interaction Templates across the Flow ecosystem, the creation of developer friendly tools and processes such as this standard for the generation of Interaction Templates is required.

## User Benefit

Developers benefit from CadenceDoc by having an easy way to declare static metadata for their transactions and scripts. This allows them to store the metadata alongside the Cadence code for the transaction or script in a CadenceDoc comment. This way, developers can iterate on their transactions and scripts, and only need to declare their metadata once, and change it as required or desired.

With the creation of tooling for generating Interaction Templates by parsing CadenceDoc, automated ways for creating Interaction Templates can be created; such as, Flow CLI integration for generating an Interaction Template from a cadence transaction or script file that has CadenceDoc, or a file watcher mechanism which autogenerates a transaction or scripts corresponding Interaction Template every time the transaction or script with CadenceDoc changes.

## Design Proposal

CadenceDoc is included in a Cadence transaction or script, below all transaction/script imports and above the `transaction` or `main` function declaration, with no new line in between.

All CadenceDoc declarations require one empty new line between.

CadenceDoc supports the following declarations:

- The paragraph of CadenceDoc declares the 'title' message. It must be in the language declared by the @lang declaration.

- The second paragraph of CadenceDoc declares the 'description' message. It must be in the language declared by the @lang declaration.

- `@version` declares the version of CadenceDoc to use. The initial version is `1.0.0`.

- `@lang` declares the default BCP-47 language tag for all messages in the CadenceDoc (defaults to 'en-US').

- `@message` declares information about a transaction/script argument

  - `@message [key]` the field `key` declares what key this argument message is for (eg: 'title' or 'description')
  - `@message [key]: [message]` the field `message` declares the human readable message to associate with the argument
  - `@message [translation] [key]: [message]` the optional field `translation` declares a BCP-47 language tag to declare the translation of the message, (default to @lang).

- `@argument` declares information about a transaction/script argument

  - `@argument [(optional)key]` the optional field `key` declares what key this argument message is for (eg: 'title' or 'description', default to 'title')
  - `@argument [(optional)key] [label]` the field `label` declares which transaction/script argument this declaration is for
  - `@argument [(optional)key] [label]: [message]` the field `message` declares the human readable message to associate with the argument
  - `@argument [(optional)translation] [key] [label]: [message]` the optional field `translation` declares a BCP-47 language tag to declare the translation of the message, (default to @lang).

- `@balance`

  - `@balance [label]` the field `label` declares the argument label this declaration is for.
  - `@balance [label]: [placeholder].[contract]` the field `placeholder` declares the address placeholder, the field `contract` declares the contract on the account represented by `placeholder` which defines the type of balance this argument is for (eg: 0xFungibleTokenAddress.).

- `@translate [...translation]` declares a list of BCP-47 language tags to use translate all messages.

This is an example of CadenceDoc for a transaction:

```cadence
import FungibleToken from 0xFUNGIBLETOKENADDRESS
import FlowToken from 0xFLOWTOKENADDRESS

/**
Transfer Tokens

Transfer tokens from one account to another

@version 1.0.0

@lang en-US

@param title amount: Amount
@param title to: To
@param description amount: The amount of FLOW tokens to send
@param description to: The Flow account the tokens will go to

@translate fr-FR cn-CN

@balance amount: 0xFLOWTOKENADDRESS.FlowToken
*/
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

### Drawbacks

- Since CadenceDoc lives inside the transaction/script, it unnecessarily persists information on the blockchain that is not relevant to the execution of the transaction itself.

### Alternatives Considered

- Alternatively, there could exist a separate file which such interaction metadata could be included that sits outside the

### Dependencies

- Flow Interaction Templates
  - Any breaking changes to Interaction Templates must have a corresponding update to CadenceDoc to accommodate them

### Engineering Impact

- CadenceDoc will be maintained by those working on Interaction Templates

### Best Practices

- CadenceDoc will need to be added to tutorials and documentation provided to developers so they can use it when creating transactions and scripts on Flow.

### Compatibility

- CadenceDoc will need to maintain compatibility with the latest specification of the Flow Interaction Template data structure.

## Prior Art

- CadenceDoc is built to resemble best practices from Javadoc and JSDoc.

## Questions and Discussion Topics

- Are there other viable formats for declaring static metadata for a transaction or script?
