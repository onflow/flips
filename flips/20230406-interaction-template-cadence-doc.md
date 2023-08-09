# Interaction Template Cadence Doc (v1.0.0)

| Status        | Proposed                                                                        |
| :------------ | :------------------------------------------------------------------------------ |
| **FLIP #**    | TBD                                                                             |
| **Forum**     | TBD                                                                             |
| **Author(s)** | Bjarte Karslen (bjartek@find.xyz), Jeffrey Doyle (jeffrey.doyle@dapperlabs.com) |
| **Sponsor**   | Jeffrey Doyle (jeffrey.doyle@dapperlabs.com)                                    |
| **Updated**   | 2023-04-06                                                                      |

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

Interaction Template Cadence Doc can be included in a Cadence transaction or script, below all transaction/script imports and above the `transaction` or `main` function declaration, with no new line in between.

All Interaction Template Cadence Doc declarations require one empty new line between.

Interaction Template Cadence Doc supports the following declarations:

- The paragraph of Interaction Template Cadence Doc declares the 'title' message. It must be in the language declared by the @lang declaration.

- The second paragraph of Interaction Template Cadence Doc declares the 'description' message. It must be in the language declared by the @lang declaration.

- `@version` declares the version of Interaction Template Cadence Doc to use. The initial version is `1.0.0`.

- `@lang` declares the default BCP-47 language tag for all messages in the Interaction Template Cadence Doc (defaults to 'en-US').

- `@message` declares information about a transaction/script argument

  - `@message [key]` the field `key` declares what key this argument message is for (eg: 'title' or 'description')
  - `@message [key]: [message]` the field `message` declares the human readable message to associate with the argument
  - `@message [translation] [key]: [message]` the optional field `translation` declares a BCP-47 language tag to declare the translation of the message, (default to @lang).

- `@parameter` declares information about a transaction/script argument

  - `@parameter [(optional)key]` the optional field `key` declares what key this argument message is for (eg: 'title' or 'description', default to 'title')
  - `@parameter [(optional)key] [label]` the field `label` declares which transaction/script argument this declaration is for
  - `@parameter [(optional)key] [label]: [message]` the field `message` declares the human readable message to associate with the argument
  - `@parameter [(optional)translation] [key] [label]: [message]` the optional field `translation` declares a BCP-47 language tag to declare the translation of the message, (default to @lang).

- `@balance`

  - `@balance [label]` the field `label` declares the argument label this declaration is for.
  - `@balance [label]: [contract]` the field `contract` declares the contract on the account which defines the type of balance this argument.

- `@translate [...translation]` declares a list of BCP-47 language tags to use translate all messages.

This is an example of Interaction Template Cadence Doc for a transaction:

```cadence
import "FungibleToken"
import "FlowToken"

/**
Transfer Tokens

Transfer tokens from one account to another

@version 1.0.0

@lang en-US

@parameter title amount: Amount
@parameter title to: To
@parameter description amount: The amount of FLOW tokens to send
@parameter description to: The Flow account the tokens will go to

@translate fr-FR cn-CN

@balance amount: FlowToken
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

This is an example of Interaction Template Cadence Doc for a script:

```cadence
import "FungibleToken"
import "FlowToken"

/**
Flow Token Balance

Get account Flow Token balance

@version 1.0.0

@lang en-US

@parameter title address: Address
@parameter description address: Get Flow token balance of Flow account

@translate fr-FR cn-CN

*/
pub fun main(address: Address): UFix64 {
    let account = getAccount(address)
    let vaultRef = account.getCapability(/public/flowTokenBalance).borrow<&FlowToken.Vault{FungibleToken.Balance}>() ?? panic("Failed to borrow Flow Token balance reference")

    return vaultRef.balance
}

```

### As a separate JSON file

Instead of including Interaction Template Cadence Doc directly inside a Cadence transaction or script, it could be included as a separate JSON file.

If there is Interaction Template Cadence Doc both within the Cadence transaction or script, and in a corresponding JSON file as well, the content of the Interaction Template Cadence Doc in the Cadence transaction or script takes precedent.

The Interaction Template Cadence Doc JSON file must be in following format:

- An array of InteractionTemplateCadenceDoc objects
  - Each InteractionTemplateCadenceDoc must include:
    - A `version` field, denoting the version of the InteractionTemplateCadenceDoc object (The initial version is `1.0.0`).
    - A `lang` field, denoting the BCP-47 language tag representing the language of the InteractionTemplateCadenceDoc object
    - A `messages` field, which is an object with:
      - `[key]` key subfields, where `key` is the key of each message, with value being the translation of that message for `lang`
    - A `arguments` field, which is an object with:
      - `[label]` key subfields, where `label` is the label of an argument of the Cadence transaction or script, with value being an object with:
        - `[key]` key subfields, where `key` is the key of each message, with value being the translation of that message for `lang`

This is an example of Interaction Template Cadence Doc for a transaction (as a JSON file):

```json
[
  {
    "version": "1.0.0",
    "lang": "en-US",
    "messages": {
      "title": "Transfer Tokens",
      "description": "Transfer tokens from one account to another"
    },
    "arguments": {
      "to": {
        "title": "The Flow account the tokens will go to"
      },
      "amount": {
        "title": "The amount of FLOW tokens to send"
      }
    }
  },
  {
    "version": "1.0.0",
    "lang": "fr-FR",
    "messages": {
      "title": "Jetons de transfert",
      "description": "Transférer des jetons d'un compte à un autre"
    },
    "arguments": {
      "to": {
        "title": "Le compte Flow auquel les jetons iront"
      },
      "amount": {
        "title": "Le nombre de jetons FLOW à envoyer"
      }
    }
  },
  {
    "version": "1.0.0",
    "lang": "zh-CN",
    "messages": {
      "title": "转移代币",
      "description": "将代币从一个账户转移到另一个账户"
    },
    "arguments": {
      "recipient": {
        "to": "令牌将转到的 Flow 帐户"
      },
      "amount": {
        "title": "要发送的 FLOW 代币数量"
      }
    }
  }
]
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
