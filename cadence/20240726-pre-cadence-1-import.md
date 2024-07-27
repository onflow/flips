---
status: proposed
flip: 282
authors: bastian.mueller@flowfoundation.org
updated: 2024-07-26
---

# FLIP 282: Import of pre-Cadence 1.0 Programs

## Objective

The overall goal of this proposal is to allow Cadence 1.0 to import pre-1.0 programs.

In particular, the goals of the proposed feature are to
- Allow access to data (fields), not functionality (functions)
- Allow programs to (keep) directly importing / depending on pre-1.0 programs
- Allow programs indirectly working with values that have types defined in pre-1.0 programs
  to (keep) functioning

The proposal suggests a general feature of Cadence and does not specify the exact details
of how a concrete implementation of this feature will achieve these goals.
It is likely that achieving these goals in general is not possible,
and an implementation will only support the import of certain pre-1.0 programs,
and only certain parts of those programs.

This proposal does NOT suggest adding a general means of importing any pre-1.0 program, fully.

The proposed functionality can for example be used to keep some well-known programs,
such as programs conforming to standards like the `FungibleToken` and `NonFungibleToken`,
working to the degree where at least the fields required by the standards stay readable.

## Motivation

Cadence 1.0 contains many breaking changes.
Pre-1.0 programs deployed to networks need to be updated to 1.0 to stay functional.
However, might not get updated which renders them unusable:
Both directly importing a pre-1.0 program
and indirectly importing it by working with a value that has a type defined in a pre-1.0 program
will very likely result in a parsing or type checking error,
as the pre-1.0 is likely syntactically or semantically invalid in 1.0.

This leads to two problems:

1. Programs that get updated to 1.0, but import a program that is not, will also fail to type check.

    Due to the
    [contract updatability rules](https://cadence-lang.org/docs/language/contract-updatability),
    removing an import of a pre-1.0 program is not always possible.

    As a result, programs importing pre-1.0 which are not updated to 1.0
    can in turn also not be updated to 1.0.

2. Programs that do not directly import pre-1.0 programs may encounter values at run-time
   that have types defined in pre-1.0 programs.
   Using such values leads to an import of the program where the type is defined,
   which,  just like a static import, will likely fail.

   As a result, the generic program becomes potentially unusable.
   Such "broken" values cannot be used in any way:
   for example, they cannot be passed to functions, returned to functions,
   assigned to variables or fields, put into containers, or destroyed.
   Also, it is not possible to read or write fields, or call functions on the value.

## User Benefit

The feature suggested in this proposal will resolve both problems described in the previous section,
to some degree: Most developers will now be able to stage updates for their contracts
that import pre-1.0 programs,
and programs working with values that have a type defined in a pre-1.0 program
will be usable to a certain degree (e.g. read-only access of fields).

## Design Proposal

The core idea of the suggested feature is to "recover" from parsing/type checking errors,
due to the imported program not being valid Cadence 1.0,
at import time (parsing/type checking time).

When parsing/checking using the new parser/checker fails, a pre-1.0 "recovery" is attempted.
The imported program is determined to be a syntactically valid pre-1.0 program.
For example, this can be done by parsing it using a pre-1.0 parser.

If the program is a pre-1.0 program, it is passed to a configurable "recovery function".
The function is optional and can attempt to recover the pre-1.0 program, in an unspecified way,
and choose to return a Cadence 1.0 program.

If the recovery function returns a program, it is type checked,
and used as the result of the import (i.e. both for type information and program execution).
If the recovery function does not return a program, the recovery is considered to have failed,
and the import of the original program is considered to have failed.

The recovery function is not part of the Cadence implementation,
but may be provided by the embedder of Cadence.
Cadence is not self-standing, but rather embedded into another environment.
For example, in Flow, Cadence is embedded into the Flow reference implementation, flow-go,
which implements concrete implementations of the functionality exposed in Cadence,
such as program storage, key storage, account storage, etc.

This very general feature can then be used to achieve the goals outlined before:
A concrete recovery function may attempt to parse the imported program using the pre-1.0 parser,
and then syntactically analyze the pre-1.0 program and detect that it implements a standard.

For example, the recovery function may detect that the pre-1.0 program implements
the widely used `FungibleToken` standard,
by syntactically checking that
- The contract imports the standard from the well-known address
- The program defines a contract that declares a conformance to the standard
- The contract defines the required fields (e.g. `totalSupply`)
- The contract defines the required types (e.g. `Vault`), etc.

The recovery function can then return a 1.0 program that implements the standard,
defining the required fields and functions, but notably implementing all functions as panics.

### Drawbacks

Implementations of the recovery function will need to extract information
about the imported program.
For example, this can be done by parsing the program using the pre-1.0 parser.
However, this means that the pre-1.0 parser needs to remain in the codebase, leading to tech debt.

This can be alleviated by extracting the information once,
storing it alongside the pre-1.0 program, and reading it at import time.
This could be done in an execution state migration.

### Alternatives Considered

An alternative for allowing pre-1.0 programs to be imported is to replace the contract code
stored in the execution state.

This is problematic, as the code is permanently and irrevocably changed,
without having the authors consent.
Normal contract updates, and also staging a contract update,
requires authorization from the account,
in form of (a) signature(s) on a transaction performing the change.

The proposed approach does not modify the execution state in any way,
and an implementation of a recovery function is comparable to the type checker:
it "makes sense" of what the contract code stored in the execution state.

Performing the program recovery by changing the execution state would require
adding additional complexity to the already very complex Cadence 1.0 state migration.

The biggest benefit of the proposed approach over this alternative is that its deployment
can be done incrementally, through a height-coordinated update of the network,
and no state migration is required, which significantly reduces downtime of the network.

For example, a first version of a recovery function might only recover fungible tokens,
and could be released as part of the Cadence 1.0/Crescendo release.
Later improvements to the recovery function, e.g. adding support for NFTs,
can be deployed when available.
Current estimates do not indicate that a complete solution for FTs, NFTs, etc.
will be available in time for the release of Cadence 1.0/Crescendo.

### Performance Implications

The performance impact of the proposed solution is negligible.

Program recovery is only attempted when a program fails to parse/type check.
Typical implementations of the program recovery function will be cheap,
both in terms of computation, and in memory usage.
The syntactical checking and generation of the recovered program, e.g. by parsing, is cheap.

Finally, the result of parsing/type checking imported programs is cached in general,
which will also apply to the result of the program recovery.

### Dependencies

This proposal has no dependencies.

### Engineering Impact

The implementation of the suggested solution is relatively simple.

A proof-of-concept, available in https://github.com/onflow/cadence/pull/3482,
demonstrates the feasibility and simplicity of the recovery mechanism in Cadence.
The PR also shows what an example of a recovery function that recovers a pre-1.0 `FungibleToken`
contract and makes it compatible with the version compatible with Cadence 1.0.

If the implementation of the recovery function relies on the old parser
to extract the necessary information from the program to recover it, it incurs technical debt.
However, as previously mentioned, this can be alleviated through a state migration.

### Examples

Assume the following implementation of the pre-Cadence 1.0 `FungibleToken` standard is deployed:

```cadence
const oldExampleToken = `
import "FungibleToken"

pub contract ExampleToken: FungibleToken {

   pub var totalSupply: UFix64

   pub resource Vault: FungibleToken.Provider, FungibleToken.Receiver, FungibleToken.Balance {

       pub var balance: UFix64

       init(balance: UFix64) {
           self.balance = balance
       }

       pub fun withdraw(amount: UFix64): @FungibleToken.Vault {
           self.balance = self.balance - amount
           emit TokensWithdrawn(amount: amount, from: self.owner?.address)
           return <-create Vault(balance: amount)
       }

       pub fun deposit(from: @FungibleToken.Vault) {
           let vault <- from as! @ExampleToken.Vault
           self.balance = self.balance + vault.balance
           emit TokensDeposited(amount: vault.balance, to: self.owner?.address)
           vault.balance = 0.0
           destroy vault
       }

       destroy() {
           if self.balance > 0.0 {
               ExampleToken.totalSupply = ExampleToken.totalSupply - self.balance
           }
       }
   }

   pub fun createEmptyVault(): @Vault {
       return <-create Vault(balance: 0.0)
   }

   init() {
       self.totalSupply = 0.0
   }
}
```

An implementation of the proposed recovery function could replace it with the following
Cadence 1.0 program, which implements the Fungible Token V2 standard,
which is compatible with Cadence 1.0:

```cadence
import "FungibleToken"

access(all)
contract ExampleToken: FungibleToken {

    access(all)
    var totalSupply: UFix64

    init() {
        self.totalSupply = 0.0
    }

    access(all)
    resource Vault: FungibleToken.Vault {

        access(all)
        var balance: UFix64

        init(balance: UFix64) {
            self.balance = balance
        }

        access(FungibleToken.Withdraw)
        fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
            panic("withdraw is not implemented")
        }

        access(all)
        view fun isAvailableToWithdraw(amount: UFix64): Bool {
            panic("isAvailableToWithdraw is not implemented")
        }

        access(all)
        fun deposit(from: @{FungibleToken.Vault}) {
            panic("deposit is not implemented")
        }

        access(all) fun createEmptyVault(): @{FungibleToken.Vault} {
            panic("createEmptyVault is not implemented")
        }
    }

    access(all)
    fun createEmptyVault(vaultType: Type): @{FungibleToken.Vault} {
        panic("createEmptyVault is not implemented")
    }
}
```

As a result the following function would work as expected and succeed:

```cadence
import "FungibleToken"
import "ExampleToken"

// A transaction that loads the broken ExampleToken contract and the broken ExampleToken.Vault.
// Accessing the broken ExampleToken contract value and ExampleToken.Vault resource
// does not cause a panic.
// Casting the ExampleToken.Vault resource to a FungibleToken.Vault
// and saving it back to storage does not cause a panic either.

transaction {

    let vault: @ExampleToken.Vault
    let signer: auth(LoadValue, SaveValue) &Account

    prepare(signer: auth(LoadValue, SaveValue) &Account) {
        self.vault <- signer.storage.load<@ExampleToken.Vault>(from: /storage/exampleTokenVault)!
        self.signer = signer
    }

    execute {
        log(ExampleToken.totalSupply)
        log(self.vault.balance)
        log(ExampleToken.getType())
        log(self.vault.getType())

        let exampleVault <- self.vault
        let someVault <- exampleVault as! @{FungibleToken.Vault}

        self.signer.storage.save(<-someVault, to: /storage/someTokenVault)
    }
}
```

The following transaction would however fail, due to call of the `withdraw` function:

```cadence
import "FungibleToken"
import "ExampleToken"

// A transaction that calls a function on the stored vault.
// Function calls on recovered values panic.

transaction {

    let vault: auth(FungibleToken.Withdraw) &{FungibleToken.Vault}

    prepare(signer: auth(BorrowValue) &Account) {
        self.vault = signer.storage
            .borrow<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(
                from: /storage/someTokenVault
            )!
    }

    execute {
        let vault <- self.vault.withdraw(amount: 1.0)
        destroy vault
    }
}
```

Finally, the following transaction would work as expected and succeed:

```cadence
import "FungibleToken"
import "ExampleToken"

// A transaction that loads the broken ExampleToken contract and the broken ExampleToken.Vault.
// Accessing the broken ExampleToken contract value and ExampleToken.Vault resource again
// does not cause a panic.
// Casting the FungibleToken.Vault resource to an ExampleToken.Vault
// and destroying the resource does not cause a panic either.

transaction {

    let vault: @{FungibleToken.Vault}

    prepare(signer: auth(LoadValue) &Account) {
        self.vault <- signer.storage.load<@{FungibleToken.Vault}>(from: /storage/someTokenVault)!
    }

    execute {
        log(ExampleToken.totalSupply)
        log(self.vault.balance)
        log(ExampleToken.getType())
        log(self.vault.getType())

        let someVault <- self.vault
        let exampleVault <- someVault as! @ExampleToken.Vault

        destroy exampleVault
    }
}
```

### Compatibility

The suggested solution allows some level of backward-compatibility
for pre-1.0 Cadence programs to be provided.

### User Impact

Developers and users of Cadence 1.0 programs will be able to import pre-1.0 programs
and be able to use parts of the program and access parts of stored state.
This is a significant improvement over the current situation,
where many 1.0 programs will be rendered non-functional because they import pre-1.0 programs
(directly or indirectly).

The suggested feature and implementation of the recovery function can be released
as part of ongoing upgrades to networks (height-coordinated updates)
and do not require a state migration.

## Related Issues

<!-- What related issues do you consider out of scope for this proposal,
but could be addressed independently in the future? -->

## Prior Art

None

## Questions and Discussion

### Applicable Contracts

Should only programs be recovered which are syntactically valid pre-1.0 programs?
Would it be safe to recover the import of any program which results in a parsing/checking failure?
Should only programs be recovered that used to work pre-1.0, but did not get updated to Cadence 1.0?

The benefit of recovering any import failure is that this would also handle
*future* breakage of programs, e.g. when dependencies (imported contracts) change.

It is not possible to deploy or update to programs that are syntactically or semantically invalid.

For now, the recovery mechanism is applied to any program that fails to parse/type check,
and is syntactically a valid pre-1.0 program.
A future proposal may propose to restrict this, or even propose to extend this mechanism
to programs that are not necessarily syntactically valid pre-1.0 programs.

### Function Implementations

The current proposal suggests having all function implementations panic.

Can "standard functions", like the `FungibleToken` function `createEmptyVault`,
and `withdraw`/`deposit`, be implemented with a "typical" implementation?

Failing functions reduce fragmentation, because they encourage recovered values to get replaced.

While working functions with "typical implementations" are sometimes possible,
this is not not always the case (e.g. `FungibleToken.resolveView`),
and programs using recovered values may still not function as expected.

For now, implementations of the recovery function will have function implementations panic.
A future proposal may suggest providing implementations.

### Determining recovered types

It might be useful for programs to determine if a type has been recovered.
For example, a program may choose to avoid function calls on a value that has a recovered type,
to avoid the program from panicing, reject recovered values,
or automatically replace (redeem) them for another value.

This could be achieved through several ways. For example:

- Recovered programs could implement a new interface-type (e.g. `RecoveredType`, `LegacyType`).
  Then a program can check whether a type is a recovered type by using the `isSubtype()` function.

- A new boolean field could be added to the `Type` type (run-time type values) which indicates
  if the type was recovered (e.g. `isRecovered: Bool`, `isLegacy: Bool`).

For now, the proposal does not suggest any means to determine if a type was recovered.
A future proposal may suggest a particular mechanism.

### Access to additional / unique, and potentially private, fields

The proposal currently does not specify exactly how much of a program gets recovered.
As described before, an implementation of the recovery function might determine
that the pre-1.0 program implements certain standards and at least allow read-access
to these standards' provided/required fields.

However, programs importing and working with pre-1.0 programs might want to access
other fields uniquely defined in a pre-1.0 program.
In some cases, redeeming a recovered value for a working replacement
might even require access to these unique fields.
For example, an implementation of the `NonFungibleToken` might define several more fields
in addition to the `id` field required to be defined by the standard.
Some of the fields might also be defined with non-public access in the pre-1.0 program.

Is it safe to allow access to additional / unique fields?
Is it safe to switch private to public (read-only)?

A future proposal may suggest particular implementations of the recovery function,
which might provide access to additional / unique fields,
and may also suggest making non-publicly accessible fields accessible in a read-only manner.

Additional data stored in recovered values does not necessarily have to be provided on-chain,
and may instead be provided through off-chain means.
