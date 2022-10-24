# Interaction Template Interface | Fungible Token Transfer Transaction [DRAFT]

| Status        | Draft                                                          |
| :------------ | :------------------------------------------------------------- |
| **FLIP #**    | TBD                                                            |
| **Forum**     |                                                                |
| **Author(s)** | Jeffrey Doyle (jeffrey.doyle@dapperlabs.com)                   |
| **Sponsor**   | Jeffrey Doyle (jeffrey.doyle@dapperlabs.com)                   |
| **Updated**   | 2022-09-13                                                     |

## Abstract

This FLIP establishes the Interaction Template Interface for Fungible Token Transfer. Interaction Templates that conform to and identify with this interface will be deemed Fungible Token Transfer transactions.

## Background

Interaction Template Interface is a data structure established in [FLIP-934](https://github.com/onflow/flips/blob/main/flips/20220503-interaction-templates.md).

## Interaction Template Interface

```javascript
{
    f_type: "InteractionTemplateInterface",
    f_version: "1.0.0",
    id: "",
    data: {
        flip: "FLIP-XXXX",
        title: "Fungible Token Transfer",
        arguments: {
            amount: {
                index: 0,
                type: "UFix64"
            },
            to: {
                index: 1,
                type: "Address"
            }
        }
    }
}
```

