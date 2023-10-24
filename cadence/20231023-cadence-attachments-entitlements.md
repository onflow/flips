---
status: draft 
flip: NNN (set to the issue number)
authors: Daniel Sainati (daniel.sainati@dapperlabs.com)
sponsor: Daniel Sainati (daniel.sainati@dapperlabs.com)
updated: 2023-10-23 
---

# FLIP NNN: Attachment Entitlements Reversion

## Objective

This FLIP proposes to remove the ability for attachments to `require` entitlements to their `base`, 
to remove a foot-gun from the language. 

## Motivation

Currently, attachments present a major foot-gun with respect to their interaction with entitlements.
Consider the following example, in which an attachment is used to backdoor a resource:

```

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
    from.balance = 0.0;
    destroy from
  }
}

// Victim code starts - designed without thinking about attachments
access(all) resource Company {
    access(self) var vault: @Vault;
    init(incorporationEquity: @Vault){
        pre {
            incorporationEquity.balance >= 100.0
        }
        self.vault <- incorporationEquity
    }
    access(all) fun getDepositOnlyReference(): auth(Deposit) &Vault {
        return &self.vault as auth(Deposit) &Vault
    }
    destroy() {
        destroy self.vault
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

This proposes two main changes to how attachments and entitlements interact:

1) `base` is always unentitled. Instead of allowing attachment authors to ask for specific entitlements to the `base` reference with `require` syntax, the `base` reference in an attachment is always unentitled. This means that no entitled methods on the base resource will be callable at any point, and thus will limit attachments to interaction only with the "publicly" visible portion of a resource. This would prevent the backdooring example above by preventing the definition of `Backdoor`; `sneakyWithdraw` would never be defineable because `base` will not be entitled to `Withdraw`. 

2) The `self` reference will be changed to have differing entitlements from function to function. Specifically, given some attachment definition:

    ```
    entitlement mapping M {
        // ...
    }

    access(M) attachment A for R {
        access(X) fun foo() { ... } 
    }
    ```

    Within `foo`, `self` would be entitled to `X`. In general, within a method body, the `self` reference would always just be entitled to the set of entitlements requested by that method. This change is not strictly necessary for fixing the backdooring problem, but is important to allow us to eventually re-enable entitlements 
    on the `base` later down the line, should we have a non-breaking means for doing so. 

### Drawbacks

This does necessarily limit some use cases; in this limited model it will not be possible to write attachments that interact with 
any entitled methods on the `base`. Howevever, this proposal does still leave open the possibility of entitlement inference/checking for the `base`
to be re-added in a non-breaking way after the release of Cadence 1.0.

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

3) Revert to the old version of attachment entitlement inference originally proposed in the entitlements FLIP. This FLIP originally proposed a mechanism for inferring the entitlements of the `base` reference in an attachment for each method based on the entitlements of that method and the mapping that was used to define the attachment. I.e. for some attachment declaration:

    ```
    entitlement mapping M {
    // ...
    }

    access(M) attachment A for R {
        access(X) fun foo() { ... } 
    }
    ```

    We would infer the entitlements that `base` has in the body of `foo` based on `M` and `X`; specifically, `base` in `foo` would be entitled to `M⁻¹(X)` . Essentially, given that an `X` entitlement to `A` is only available via access on an `R` that is entitled to at least `M⁻¹(X)`, this inference is reverse-engineering the entitlements that the `base` would have needed to have in order to make `foo` callable on `A`. 

    This is extremely complicated, and we moved away from this proposal for the sake of simplicity, to not further complicate an already complex feature. However, were we to move back to this model it would fully resolve the backdooring issue without sacrificing any expressivity on the part of the attachment author or the resource author. 

While each of these 3 alternatives present a solution to the backdooring problem, we ultimately selected the proposed solution because it solves the issue while also 
leaving open the most future improvemetns. By restricting `base` to unentitled references only, in the future we can add more functionality to enable using entitlements with `base` without making any breaking changes. By contrast, each of these 3 proposals would require a breaking change in order to make any alterations to how entitlements interact with `base`. 
