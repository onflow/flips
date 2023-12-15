---
status: implemented 
flip: 217
authors: Daniel Sainati (daniel.sainati@dapperlabs.com)
sponsor: Daniel Sainati (daniel.sainati@dapperlabs.com)
updated: 2023-10-24 
---

# FLIP 217: Attachment + Entitlements

## Objective

This FLIP proposes to remove the ability for attachments to `require` entitlements to their `base`, 
to remove a foot-gun from the language, and replace it with a per-function entitlement inference for
the `base` and `self` variables in attachments.

## Motivation

Currently, attachments present a major foot-gun with respect to their interaction with entitlements.
In particular, it is possible to authorize an attachment to some entitlement, and then use that to obtain access
to the attachment's base value. While this is safe in and of itself, if a value with attachments is transferred 
to another user, the attachments on that value are transferred as well and may contain attachments with entitlements
that the user is not expecting.  
Consider the following example, in which an attachment is used to backdoor a resource:

```cadence
// standard code
access(all) entitlement Withdraw
access(all) entitlement Deposit

access(all) resource Vault {

  access(all) var balance: UFix64

  init(balance: UFix64) {
    self.balance = balance
  }

  access(Withdraw) fun withdraw(amount: UFix64): @Vault {
    self.balance = self.balance - amount
    return <-create Vault(balance: amount)
  }

  access(Deposit) fun deposit(from: @Vault) {
    self.balance = self.balance + from.balance
    from.balance = 0.0
    destroy from
  }

}

// Victim code starts - designed without thinking about attachments
access(all) resource Company {

    access(self) var vault: @Vault

    init(incorporationEquity: @Vault) {

        pre {
            incorporationEquity.balance >= 100.0
        }

        self.vault <- incorporationEquity
    }

    access(all) fun getDepositOnlyReference(): auth(Deposit) &Vault {
        return &self.vault as auth(Deposit) &Vault
    }
}
// Victim code ends

access(all) attachment Backdoor for Vault {

  require entitlement Withdraw

  access(all) fun sneakyWithdraw(amount: UFix64): @Vault {
    return <- base.withdraw(amount: amount)
  }

}

access(all) fun main(): UFix64 {

   let backdooredVault <- attach Backdoor() to (<- create Vault(balance: 100.0)) with (Withdraw)
   let c <- create Company(incorporationEquity: <- backdooredVault)
   var ref = c.getDepositOnlyReference()
   
   // The following would fail because we only have a Deposit entitlement
   // var x <- ref.withdraw(amount: ref.balance)
   // Luckily we have backdoored the underlying vault so we can do this:
   var x <- ref[Backdoor]!.sneakyWithdraw(amount: ref.balance)
   var amountOfFundsStolen = x.balance

   destroy x
   destroy c

   return amountOfFundsStolen
} 
```

In this code sample, a resource that wraps a vault is vulnerable to backdooring via an attachment. 
A malicious attachment author, as in this example, could write an attachment that requires the ability to `Withdraw` from its base `Vault`, 
attach it to this `Vault`, and then provide that input `Vault` to someone else's resource without that other person knowing that the `Vault` 
is compromised. The only way for the author of the `Company` resource in this example to defend against this attack is for them to "sanitize" their
`Vault` inputs by enforcing that they have no attachments on them, which is unwieldly and would impose an extremely defensive coding style that is
detrimental to composability on chain. 

## User Benefit

This would enable contract authors to write their contracts in a more natural, intuitive style without compromising the safety of their code. 

## Design Proposal

This proposes two main changes to how attachments and entitlements interact. The first changes how access is determined for attachment functions, while 
the second changes how authorization is decided for attachments on access from references.

1) Removing the `require entitlement` syntax from `attachment` declarations, along with the ability to declare `attachment`s with mapped access. Instead, 
all attachments will be declared with `access(all)` access on their declaration, and the entitlements of the `self` and `base` references in each function will be
determined by the entitlements used in the access of that function. So, for example, the `Backdoor` attachment above would fail if written as:

    ```
    access(all) attachment Backdoor for Vault {
        access(all) fun sneakyWithdraw(amount: UFix64): @Vault {
            return <- base.withdraw(amount: amount)
        }
    }
    ```

    Here, `sneakyWithdraw` withdraw would not type check because it is declared as `access(all)`, meaning that `self` and `base` are unentitled in its body. Conversely, if
    the definition were written as

    ```
    access(all) attachment Backdoor for Vault {
        access(Withdraw) fun sneakyWithdraw(amount: UFix64): @Vault {
            return <- base.withdraw(amount: amount)
        }
    }
    ```

    This definition would type check, as since `sneakyWithdraw` has `Withdraw` access, this means that `self` and `base` are entitled to `Withdraw` in its body.
    This brings us to the second proposed change:

2) Attachment access always propagates entitlements from its base as if through the `Identity` map. Equivalently, `v[A]` always produces a reference to `A` that is entitled to the same entitlements as `v`. This means that in the above example, the definition of `sneakyWithdraw` is safe because a `Withdraw`-entitled reference to 
`Backdoor` can only be obtained from a `Withdraw`-entitled reference to `Vault`. Effectively, this means that the owner of an value with attachments has the sole 
ability to decide the entitlements with which those attachments can be used, since they are the only one who can create references to that value. Since references
are invalidated when resources are transferred, resource owners do not "inherit" attachment entitlements upon receiving a value from another user. 

Note that this necessarily implies that attachments can only support the same entitlements that exist on the `base` value. I.e. an attachment defined for some
`R` that only has entitlements `X` and `Y` will itself only support `X` and `Y`. 

### Drawbacks

This does necessarily limit some use cases; it will no longer be possible to define separate entitlements for attachments for any new functionality that they may
add on top of what the base can do. However, this is the least limiting solution we identified. 

### Alternatives Considered

A number of alternatives exist for this change:

1) Leave the attachments feature as is, and encourage developers to code defensively. We would likely need to add some utility features
to attachments, like a `hasAttachments: Bool` field on resource and a `removeAttachment(ty: Type)` method to allow authors to dynamically sanitize inputs to their 
resources. This would make writing attachments as simple as possible, as they would have the full range of expressivity they do in the current version of Stable Cadence. However, it would achieve this by placing an extreme burden on the authors of resources by forcing them to consider every possible attachment that might exist on their
resources, and to code extremely defensively at all times. We feel that this is unfeasible, and would be an undue burden on developers. 

2) Require `base` references to be entitled to the required entitlements to access attachments on them. In the example outlined in the "Motivation" section, one way 
to resolve this backdooring issue would be to require that any `Vault` reference on which the `Backdoor` attachment is accessed be itself entitled to `Withdraw`. This way, the `var x <- ref[Backdoor]!.sneakyWithdraw(amount: ref.balance)` line would fail, as `ref` is not entitled to `Withdraw`, and thus `Backdoor` would not be accessible. In general, for some attachement `A` `require`ing entitlement `E`, `v[A]` would require that `v` either be a composite, or a reference entitled to `E`. 

    This idea is convenient because it still allows attachment authors to write most of their use cases as normal, and does not require base resource authors to defend 
    against these kinds of backdooring attacks. There is no harm done in the definition of `Backdoor` in this model, as if `Backdoor` is accessible on any base, that `base` was already able to `withdraw` anyways, so no rights escalation occurs in practice. However, this is crucially limiting for one major reason: it becomes impossible to write an attachment that has multiple entitlements that should each be individually available. Consider the following use case, an attachment for a `Vault` that is designed to automatically convert between an `A` and `B` currency, e.g (in pseudo-code):

    ```
    attachment BConverter for AVault {
        require entitlement Deposit, Withdraw

        access(Deposit) deposit(bVault: @BVault) {
            let aVault <- create AVault(b.balance * exchangeRate)
            base.deposit(<- aVault)  // requires Deposit entitlement on base
            destroy bVault
        }
        access(Withdraw) withdraw(amountInB: UFix64): @BVault {
            let amountInA = amountInB * exchangeRate
            let aVault <- base.withdraw(amountInA) // requires Withdraw entitlement on base
            let bVault <- create BVault(a.balance / exchangeRate)
            destroy aVault
            return <- bVault
        }
    }
    ```

    In this example, the `BConverter` requires a `Deposit` and `Withdraw` entitlement to its `AVault` base, so that it can `deposit` into it from its own `deposit` function,
    and `withdraw` from it in its `withdraw` function. However, it does not need these two entitlements at the same time. So, if we were to require that `BConverter` is only accessible on `AVault` references that are entitled to both `Deposit` and `Withdraw`, it would not be possible to access a `BConverter` on an `auth(Deposit) AVault` reference. This is unfortunate, as there is no reason that we should need to be able to `withdraw` from an `AVault` in order to `deposit` a converted `B` currency into it. 

    It could be possible to split the `BConverter` into two attachments `BDepositor` and `BWithdrawer` that each has only one of the two functions and thus requires only one of the two entitlements. Then someone with an `auth(Deposit) &AVault` could access a `BDepositor`, while a `auth(Withdraw) &AVault` would be necessary for to access a `BWithdrawer`. However, this only works when the entitlements in an attachment can be cleanly split like in this example, in more complex use cases this may not be feasible. It furthermore encourages a coding pattern where attachments must only handle exactly one piece of functionality, and "composition" attachments would become necessary to handle interaction between these many different kinds of attachments. Given that the presence of a sub attachment cannot be guaranteed statically, this would also require attachment authors to explicitly handle all possible combinations of sub-attachments that may or may not be necessary for their logic. 

3) Simply remove support for entitlements on attachments. This proposal is maximally safe, in that there is minimal risk of security issues, since it will be impossible
to access any entitled features of the `base` reference. It is, however, also quite limiting, in that it would prevent a large number of potential attachment use cases from being expressible in Cadence. 
