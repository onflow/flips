---
status: draft 
flip: NNN (set to the issue number)
authors: Ramtin Seraj (ramtin.seraj@flowfoundation.org), ...
sponsor: To be added
updated: 2023-11-16 
---

# FLIP NNN: EVM integration interface

## Objective

- Defining a Cadence interface for the EVM integrated into the FVM. 
- Facilitates seamless interaction between Cadence and EVM enviornments. 

## Motivation

This work would makes it easier for EVM-centeric Dapps and Platforms to adopt Flow.
Please see [this forum discussion](https://forum.flow.com/t/evm-on-flow-beyond-solidity/5260) for motivations.

## Design Proposal

#### EVM as a standard smart contract

An easy way to understand the approach proposed in this Flip is to consider Flow EVM as a virtual blockchain deployed to Flow blockchain at specific address (e.g. service account). While it behaves as if someone has implemented a complete EVM runtime in Cadence, we utilize the extensibility property of the Cadence runtime and use the same EVM go implementation that Ethereum uses. Cadence runtime is designed to be extensible with standard cadence smart contracts implemented in native languages as long is its resource meter-able, resource-bounded and deterministic.  

Every Flow transaction has access to this environment similar to the way it has access to other standard contracts. 

```jsx
import EVM from <ServiceAddress>
```

Every mutable interaction with this smart contract, results in a new block being formed and stored on-chain and emits several FLOW event types ("evm.BlockExecuted”, "evm.TransactionExecuted”) that could be consumed to follow the chain (see here for more details). 

Because we use the Flow transactions to interact for the Flow EVM environment, there is no need for complex block formation logs (e.g. miner/block proposer, uncle chains, …). It is executed and verified on Flow similar to the way all other transactions and smart contracts are executed. Its also protected from MEV malicious behaviours using same mechanism that all other Flow transactions are protected. 

On EVM environment, resource consumptions are metered as `gas usage`, when interacting with EVM environment, the total gas usage is translated back into computation usage and would be paid as part of FLOW transaction fees (weigh-adjusted conversion). 

#### EVM Addresses

On EVM there is no notion of accounts (or minimum balance) and any arbitrary sequence of bytes of length 20 is a valid address. 

Every EVM address has a balance of native token (e.g. ETH on Ethereum), a nonce (for deduplication) and root hash of the state (if smart contract). 
In this design we use FLOW token for this native token. The balance of an EVM address is stored as a denomination of FLOW (atto-FLOW) similar to the way Wei is stored for Ethereum accounts.  

```jsx
access(all) contract EVM {

    /// EVMAddress is an EVM-compatible address
    access(all) struct EVMAddress {

        /// Bytes of the address
        access(all) let bytes: [UInt8; 20]

        /// Constructs a new EVM address from the given byte representation
        init(bytes: [UInt8; 20])

        /// Returns the balance of this address
        access(all) fun balance(): @Balance

        /// Deposits the given vault into the EVM account with the given address
        access(all) fun deposit(from: @FlowToken.Vault)
    }
}
```

A FLOW is equivalent of 10^18 atto-FLOW. Because EVM environments uses different way of storage of value for the native token than FLOW protocol (FLOW protocol uses a fixed point representation), to remove any room for mistakes, EVM environment has structure called `Balance` that could be used.

Note that no new Flow token is minted and every balance on EVM addresses has to be deposited by bridging FLOW tokens into addresses using `deposit` method.

```jsx
access(all)contract EVM {

    access(all) struct Balance {
        /// The balance in FLOW
        access(all) let flow: UFix64

        /// Constructs a new balance, given the balance in FLOW
        init(flow: UFix64) 

        /// Returns the balance in terms of atto-FLOW.
        /// Atto-FLOW is the smallest denomination of FLOW inside EVM
        access(all) fun toAttoFlow(): UInt64
    }
}
```

Every account on FLOW evm could be queried by constructing an EVM structure.  Here is an example: 

```jsx
// Example of balance query
import EVM from <ServiceAddress>

  access(all)
  fun main(bytes: [UInt8; 20]) {
      let addr = EVM.EVMAddress(bytes: bytes)
      let bal = addr.balance()
  }
```

#### EVM-style transaction wrapping

One of the design goals for this work is to facilitate interaction with this environment for tools that are not FLOW-compatible. For this goal, the EVM smart contract accepts RLP encoded transactions to be executed. Any transaction could be wrapped and submitted by any one through Flow transactions. As mentioned earlier, the resource usage during EVM interaction is translated back into Flow transaction fees and has to be paid by the account that wrapped the original transaction. 

To facilitate the wrapping operation and refunding, the run interface also allows a coinbase address to be passed. This coinbase address is an EVM address that would receive the gas usage * gas price (set in transaction). Basically, the transaction wrapper behaves similar to a miner, receives the gas usage fees on an EVM address and pays for the transaction fees. 

Any failure during the execution would revert the whole Flow transaction. 

```jsx
// Example of tx wrapping
import EVM from <ServiceAddress>

  access(all)
  fun main(rlpEncodedTransaction: [UInt8], coinbaseBytes: [UInt8; 20]) {
      let coinbase = EVM.EVMAddress(bytes: coinbaseBytes)
      EVM.run(tx: rlpEncodedTransaction, coinbase: coinbase)
  }
```

For example, a user might use meta-mask to sign transaction for FLOW EVM and broadcast it to services that checks the gas fee on the tx and wraps the transaction to be executed. 

Note that account nonce would protect double execution of a transaction, similar to how other non-virtual blockchains prevent the minor to include a transaction multiple times. 

#### Bridged accounts

Another major goal for this work is seamless composability across environments. For this goal we have introduced a new type of addresses to the EVM environment (besides EOA and Smart Contract accounts). This new address type are similar to EOA’s except instead of being tied to a public/private key it would be controlled by Cadence resources. Any bridged account has a corresponding `BridgedAccount resource` and any Flow account or Cadence smart contract that holds this resource could interact with the EVM environment on behalf of address that stored in the resource. 

```jsx
access(all)contract EVM {

    access(all) resource BridgedAccount {

        access(self)
        let addressBytes: [UInt8; 20]

        /// constructs a new bridged account for the address
        init(addressBytes: [UInt8; 20]) 

        /// The EVM address of the bridged account
        access(all)
        fun address(): EVMAddress 

        /// Withdraws the balance from the bridged account's balance
        access(all)
        fun withdraw(balance: Balance): @FlowToken.Vault

        /// Deploys a contract to the EVM environment.
        /// Returns the address of the newly deployed contract
        access(all)
        fun deploy(code: [UInt8], gasLimit: UInt64, value: Balance): EVMAddress 

        /// Calls a function with the given data.
        /// The execution is limited by the given amount of gas
        access(all)
        fun call(to: EVMAddress, data: [UInt8], gasLimit: UInt64, value: Balance): [UInt8] 
    }

    /// Creates a new bridged account
    access(all)
    fun createBridgedAccount(): @BridgedAccount {
        return <-create BridgedAccount(
            addressBytes: InternalEVM.createBridgedAccount()
        )
    }
}
```

Bridge account addresses are allocated by the FVM and stored inside the resource. Calls through the bridged accounts forms a new type of transactions for the EVM that doesn’t requires signatures and doesn’t need nonce checking. Bridged accounts could deploy smart contract or make calls to the ones that are already deployed on Flow EVM. 

Also these bridged accounts facilitates withdrawal of Flow tokens back from EVM balance environment into the Cadence environment through `withdraw`. 

**What about other fungible and non-tokens?**

The main reason we call these account bridged accounts is their design makes it easy to build bridges. An Cadence smart contract can controls a bridged account which means it could facilitate transactional operation across two environments. For example this smart contract can accept an Fungible on the cadence side and remit the equivalence on the EVM side to a target address in a single Flow transaction. More about this would come in follow up Flips. 

#### Safety and Reproducibility of the EVM state

EVM state is not accessible by Cadence outside of the defined interfaces above, and Cadence state is also protected against raw access from the E
At the start the EVM state is empty (empty root hash) and all the state is stored under a FLOW account that is not controlled by anyone (network owned). The minimum storage balance is always maintained (through fees). 
Any interaction with this environment emits an transactionExecuted event (direct calls for bridged accounts uses `255` as tx type).  

So anyone following these events could reconstruct the whole EVM state by re-executing these transactions.


### Drawbacks

[WIP]

### Alternatives Considered

[WIP]

### Performance Implications

[WIP]

* Do you expect any (speed / memory)? How will you confirm?
* There should be microbenchmarks. Are there?
* There should be end-to-end tests and benchmarks. If there are not 
(since this is still a design), how will you track that these will be created?

### Dependencies

* Dependencies: does this proposal add any new dependencies to Flow?
* Dependent projects: are there other areas of Flow or things that use Flow 
(Access API, Wallets, SDKs, etc.) that this affects? 
How have you identified these dependencies and are you sure they are complete? 
If there are dependencies, how are you managing those changes?

### Engineering Impact

* Do you expect changes to binary size / build time / test times?
* Who will maintain this code? Is this code in its own buildable unit? 
Can this code be tested in its own? 
Is visibility suitably restricted to only a small API surface for others to use?

### Best Practices

* Does this proposal change best practices for some aspect of using/developing Flow? 
How will these changes be communicated/enforced?

### Tutorials and Examples

* If design changes existing API or creates new ones, the design owner should create 
end-to-end examples (ideally, a tutorial) which reflects how new feature will be used. 
Some things to consider related to the tutorial:
    - It should show the usage of the new feature in an end to end example 
    (i.e. from the browser to the execution node). 
    Many new features have unexpected effects in parts far away from the place of 
    change that can be found by running through an end-to-end example.
    - This should be written as if it is documentation of the new feature, 
    i.e., consumable by a user, not a Flow contributor. 
    - The code does not need to work (since the feature is not implemented yet) 
    but the expectation is that the code does work before the feature can be merged. 

### Compatibility

* Does the design conform to the backwards & forwards compatibility [requirements](../docs/compatibility.md)?
* How will this proposal interact with other parts of the Flow Ecosystem?
    - How will it work with FCL?
    - How will it work with the Emulator?
    - How will it work with existing Flow SDKs?

### User Impact

* What are the user-facing changes? How will this feature be rolled out?

## Related Issues

What related issues do you consider out of scope for this proposal, 
but could be addressed independently in the future?

## Prior Art

Does the proposed idea/feature exist in other systems and 
what experience has their community had?

This section is intended to encourage you as an author to think about the 
lessons learned from other projects and provide readers of the proposal 
with a fuller picture.

It's fine if there is no prior art; your ideas are interesting regardless of 
whether or not they are based on existing work.

## Questions and Discussion

State where you would want the community discussion to take place (choosing between the PR itself, or forum post)
Seed with open questions you require feedback on from the FLIP process. 
What parts of the design still need to be defined?