---
status: draft
flip: 198
authors: Huu Thuan Nguyen (nguyenhuuthuan25112003@gmail.com)
sponsor: None
updated: 2023-09-18
---

# FLIP 198: Authorized Call

## Objective

The objective of this FLIP is to introduce the "Authorized Call" feature, which allows functions to be marked as private unless they are called with a specific prefix.

This feature aims to enhance access control in Contracts, providing developers with more flexibility and fine-grained control over function visibility based on caller Contracts.

## Motivation

Flow introduced Cadence, a Resource-Oriented Programming Language which works towards the Capability system, it replaced `msg.sender` and proved to be effective for small projects. \
However, as projects grow in size and complexity, the efficiency of the Capability system decreases compared to the use of `msg.sender`.

Besides, The existing access control mechanisms is relatively simple, they have limitations when it comes to defining private functions that can only be accessed under specific circumstances. This can make it challenging for developers to enforce strict access control rules in complex projects.

To illustrate the issue, let's consider a specific example. Suppose we have a `Vault` Contract with a function called `Vault.swap()`, which should only be called by the `Plugin` Contracts.

```cadence
access(all) contract Vault {
    access(all) resource Admin {
        access(all) fun _swap(from: @FungibleToken.Vault, expectedAmount: UFix64): @FungibleToken.Vault {
           return <- expected;
        }
    }

    init() {
        self.account.save(<- Admin(), to: Vault.AdminPath);
    }
}
```

Currently, we can achieve this with Flow using different approaches:

Approach 1: Saving the `Admin` Resource to the `Plugin` deployer account.

```cadence
access(all) contract Plugin {
    access(all) fun swap(from: @FungibleToken.Vault, expectedAmount: UFix64): @FungibleToken.Vault {
        return self.account.borrow<&Vault.Admin>(from: Vault.AdminPath)!
            .swap(from: <- from, expectedAmount: expectedAmount);
    }
}
```

Approach 2: Saving the `Admin` Capability to the `Plugin` Contract.

```cadence
access(all) contract Plugin {
    let vaultAdmin: Capability<&Vault.Admin>;

    access(all) fun swap(from: @FungibleToken.Vault, expectedAmount: UFix64): @FungibleToken.Vault {
        return self.vaultAdmin.swap(from: <- from, expectedAmount: expectedAmount);
    }

    init(vaultAdmin: Capability<&Vault.Admin>) {
        self.vaultAdmin = vaultAdmin;
    }
}
```

However, both approaches have drawbacks:

- The `Admin` Resource definition increases the code size and makes maintenance and updates more challenging.
- Adding a new `Plugin` Contract requires operating with the `Vault` deployer account, reducing decentralization and introducing unnecessary steps.
- As projects become larger and require more complex access control rules, the need for additional Resources meeting some specific requirements increases, which leads to a more significant increase in code size.
- The deployer accounts will have a lot of unused Resources.
- This is not work with Factory designs, because the changes to account only be done after transaction. If the deployed Contract sends some Capabilities or Resources to the Factory Contract, it cannot be accessed right away.
- There are also other approaches like using `inbox` but also dealing with same issues.

## User Benefit

This proposal is aimed at making Contracts more decentralized, independent of the deployer account. This will be easier to manage and friendly to high complexity projects. Some powerful implementations will also be unlocked.

## Design Proposal

The proposed design introduces the following enhancements

### Proposal Details

#### Upgradations to the `auth` and `access` keywords

This `auth` keyword existed in Cadence as a modifier for References to make it freely upcasted and downcasted. \
But in this proposal, it is also combined with `access` to mark a function as private unless it is called with an `auth` prefix.

Inside the function, the `auth` prefix can be used to access the caller Contract.

```cadence
// FooContract.cdc
access(auth) fun foo() {
    log(auth.address); // The caller Contract address
}
```

In order to make a call to `foo()`, it must have the `auth` prefix which means it is accepted to be identified.

```cadence
// BarContract.cdc
// Deployed at 0x01
import FooContract from "FooContract";

FooContract.foo(); // -> Invalid, `auth` is missing
auth FooContract.foo(); // Valid, log: 0x01
```

Once it is possible to access the caller Contract and utilize its functionalities, developers can build powerful features and implement complex logic to ensure Contract security besides enhance the flexibility and extensibility of Contracts.

#### Improvements to the `import` keyword

With this prefix, we can import the whole Contract as authorized, which all calls to the Contract will be marked as `auth` without the prefix.

```cadence
// BarContract.cdc
// Deployed at 0x01
import FooContract as auth from "FooContract";

FooContract.foo(); // Valid, log: 0x01
```

#### Transaction integration

Not only supports the function to determine the caller Contract, but this proposal also can determine the caller Account (the Authorizer) in Transactions.

Multiple authorizers may be specified in the Transaction.

Due to basing on the Authorizer account, it can only be done in the `prepare` phase. And the `auth` keyword can be changed by the argument label.

```cadence
// the Authorizer address is 0x01
transaction() {
    prepare(auth: AuthAccount) {
        auth FooContract.foo(); // Valid, log: 0x01
    }
}
```

```cadence
// the Authorizer addresses are 0x01 and 0x02
transaction() {
    prepare(auth1: AuthAccount, auth2: AuthAccount) {
        auth1 FooContract.foo(); // Valid, log: 0x01
        auth2 FooContract.foo(); // Valid, log: 0x02
        auth FooContract.foo(); // Invalid, `auth` is ambiguous
    }
}
```

#### Interface integration

This proposal also supports the `auth` keyword in Interfaces, which can be used to restrict access to the functions where its Contract implements those Interfaces.

```cadence
// FooInterface.cdc
access(all) contract interface FooInterface {
    access(all) let queue: [Addess];

    access(auth) fun foo() {
        pre {
            self.queue[auth.address] == nil: "Already joined"
        }
    }
}
```

```cadence
// FooContract.cdc
access(auth) contract FooContract: FooInterface {
    access(all) let queue: [Addess] = [0x01];
    access(auth) fun foo();
}
```

```cadence
// BarContract.cdc
// Deployed at 0x01
auth FooContract.foo(); // pre-condition failed: Already joined
```

```cadence
// AnotherBarContract.cdc
// Deployed at 0x02
auth FooContract.foo(); // Valid
```

#### Authorized Contracts

A Contract can be marked as authorized, which needs to be imported with the `auth` prefix, otherwise, it will be completely inaccessible.

```cadence
// FooContract.cdc
access(auth) contract FooContract {
    access(all) fun foo();
}
```

```cadence
// BarContract.cdc
import FooContract from "FooContract"; // Invalid, `auth` is missing

import FooContract as auth from "FooContract"; // Valid
FooContract.foo(); // Valid
```

Applies to Interfaces:

```cadence
// FooInterface.cdc
access(auth) contract interface FooInterface {
    access(all) fun foo();
}
```

```cadence
// FooContract.cdc
access(all) contract FooContract: FooInterface // Invalid, not compatible with FooInterface

access(auth) contract FooContract: FooInterface {
    access(all) fun foo();
}
```

```cadence
// BarContract.cdc
let fooReference: &FooInterface = getAccount(fooAddress).contracts
    .borrow<&FooInterface>(name: "FooContract")!; // Invalid, `auth` is missing

let fooReference: auth &FooInterface = auth getAccount(fooAddress).contracts
    .borrow<&FooInterface>(name: "FooContract")!; // Valid
```

#### Factory recommendation

This proposal recommends the `auth` keyword in `init()` by default. This enables to determine the Factory Contract address in the `init` function.

**Old**:

```cadence
init(router: Address) {
    self.router = router;
}

```

**New**:

```cadence
init() {
    self.router = auth.address;
}
```

### Drawbacks

I can't think of any drawbacks of this proposal yet.

### Alternatives Considered

The keyword `auth` can be considered to be replaced with other keywords.

### Performance Implications

This proposal does is not any performance implications.

### Dependencies

Actually, these are just functions having a hidden argument called `auth`, it is hidden to ensure that the caller cannot pass a fake account to the function.

When calling an `auth` function, the Contract address is passed internally into it.

```cadence
access(auth) fun foo( /auth: PublicAccount/ ); // auth is a hidden argument
```

### Tutorials and Examples

In the below examples, we demonstrate how to restrict access to functions using `auth` keywords.

#### Example 1

Supposes there is a dangerous function should be called only by itself or the deployer account.

```cadence
access(auth) fun collectFee() {
    pre {
        auth.address == self.account.address: "Forbidden"
    }
}
```

#### Example 2

Supposes we have a `Vault` Contract with a `Vault._swap()` function which should be restricted to be callable only by specific `Plugin` Contracts.

```cadence
// Vault.cdc
access(all) contract Vault {
    access(all) let approvedPlugins: [Address];

    access(auth) fun _swap(from: @FungibleToken.Vault, expectedAmount: UFix64) {
        pre {
            self.approvedPlugins.contains(auth.address): "Forbidden"
        }

        return <- expected;
    }
}
```

#### Example 3

Supposes we have a `SwapGOV` Contract which is a Factory Contract, it can create `SwapPair` Contracts and collect fees from them in the future.

```cadence
// SwapGOV.cdc
access(all) contract SwapGOV {
    access(all) let pairs: [Address];

    access(auth) fun createSwapPair() {
        newAccount.contracts.add(
            name: "SwapPair",
            code: SwapPair.code
        );

        self.pairs.append(newAccount.address);
    }

    access(auth) fun collectAllFees() {
        pre {
            auth.address == self.account.address: "Forbidden"
        }

        for pair in self.pairs {
            auth pair.collectGOVFees();
        }
    }
}
```

```cadence
// SwapPair.cdc
access(all) contract SwapPair {
    access(auth) fun collectGOVFees() {
        pre {
            auth.address == self.gov: "Forbidden"
        }
    }

    init() {
        self.gov = auth.address;
    }
}
```

### Compatibility

As this is an additional improvement, it does not affect with anything.

## Prior Art

This works similarly to the `msg.sender` in Solidity (not `tx.origin`).
