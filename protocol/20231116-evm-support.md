---
status: draft 
flip: 223
authors: Ramtin Seraj (ramtin.seraj@flowfoundation.org), Bastian Müller (bastian@dapperlabs.com)
sponsor: Dieter Shirley (dete@dapperlabs.com)
updated: 2023-12-04
---

# FLIP 223: EVM integration interface

## Objective

- Defining a Cadence interface for the EVM integrated into the FVM. 
- Facilitates seamless interaction between Cadence and EVM environments. 

## Motivation

Following the discussion [here](https://forum.flow.com/t/evm-on-flow-beyond-solidity/5260), this proposal outlines a path to achieve full EVM equivalence on Flow, enabling developers to deploy any Ethereum decentralized application (dApp) on Flow without making any code changes. This allows developers to fully leverage the native functionality of Flow. Trusted tools and protocols such as Uniswap, Opensea, Metamask, Chainlink Oracle, Layerzero, AAVE, Curve, Remix, and Hardhat will be readily compatible with Flow. Additionally, developers will still have the ability to write Cadence smart contracts to extend and enhance the functionality of Solidity smart contracts, ensuring full composability.

Support for EVM on Flow enables developers to leverage the network effects and tooling available in Solidity and EVM, while also benefiting from Flow's user-friendly features and mainstream focus for onboarding and user experience. Additionally, developers can take advantage of Cadence's unique capabilities.

## Design Proposal

#### EVM as a standard smart contract

To better understand the approach proposed in this Flip, consider EVM on Flow as a virtual blockchain deployed to the Flow blockchain at a specific address (e.g., a service account). EVM on Flow functions as a smart contract that emulates the EVM with its own dedicated chain-ID. Signed transactions are inputted, and a chain of blocks is produced as output. Similar to other built-in standard contracts (e.g., RLP encoding), this EVM environment can be imported into any Flow transaction or script.

This is made possible using selective integration of the core EVM runtime without the supporting software stack in which it currently exists on Ethereum. For equivalence we also provide an EVM compatible JSON-RPC API implementation to facilitate EVM on Flow interactions from existing EVM clients.```

```
import EVM from <ServiceAddress>
```

Within the Flow transaction, if EVM interaction is successful 

- it makes changes to the on-chain data
- forms a new block if successful,
- emits several Flow event types (see [here](https://github.com/onflow/flow-go/blob/master/fvm/evm/types/events.go)) that can be consumed to track the chain progress.
And if unsuccessful, it reverts the transaction.

As EVM interactions are encapsulated within Flow transactions, they leverage the security measures provided by Flow. These transactions undergo the same process of collection, execution, and verification as other Flow transactions, without any EVM intervention. Consequently, there is no requirement for intricate block formation logic (such as handling forks and reorganizations), mempools, or additional safeguards against malicious MEV (Miner Extractable Value) behaviours. More information about Flow's consensus model is available [here](https://flow.com/core-protocol).

In the EVM environment, resource consumption is metered as "gas usage". When interacting with the EVM environment, the total gas usage is translated back into Flow computation usage and is be paid as part of FLOW transaction fees (weigh-adjusted conversion).

#### EVM Addresses

In the EVM world, there is no concept of accounts or a minimum balance requirement. Any sequence of bytes with a length of 20 is considered a valid address.

Every EVM address has a balance of native tokens (e.g. ETH on Ethereum), a nonce (for deduplication) and a root hash of the state (if smart contract). 
In this design, we use the FLOW token for this native token. The balance of an EVM address is stored as a smaller denomination of FLOW called `Atto-FLOW`, it works similarly to the way Wei is used to store ETH values on Ethereum.  

```
access(all) contract EVM {

    /// EVMAddress is an EVM-compatible address
    access(all) struct EVMAddress {

        /// Bytes of the address
        access(all) let bytes: [UInt8; 20]

        /// Constructs a new EVM address from the given byte representation
        init(bytes: [UInt8; 20])

        /// Returns the balance of this address
        access(all) fun balance(): Balance

        /// Deposits the given vault into the EVM account with the given address
        access(all) fun deposit(from: @FlowToken.Vault)
    }
}
```

A FLOW is equivalent of 10^18 atto-FLOW. Because EVM environments uses different way of storage of value for the native token than FLOW protocol (FLOW protocol uses a fixed point representation), to remove any room for mistakes, EVM environment has structure called `Balance` that could be used.

Note that no new FLOW token is minted and every balance on EVM addresses has to be deposited by bridging FLOW tokens into addresses using `deposit` method.

```
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

Every account on Flow EVM could be queried by constructing an EVM structure.  Here is an example: 

```
// Example of balance query
import EVM from <ServiceAddress>

access(all)
fun main(bytes: [UInt8; 20]) {
    let addr = EVM.EVMAddress(bytes: bytes)
    let bal = addr.balance()
}
```

#### EVM-style transaction wrapping

One of the design goals of this work is to ensure that existing EVM ecosystem tooling and products which builders use can integrate effortlessly. To achieve this, the EVM smart contract accepts RLP-encoded transactions for execution. Any transaction can be wrapped and submitted by any user through Flow transactions. As mentioned earlier, the resource usage during EVM interaction is translated into Flow transaction fees, which must be paid by the account that wrapped the original transaction.

To facilitate the wrapping operation and refunding, the run interface also allows a `coinbase` address to be passed. The use of the `coinbase` address in this context indicates the EVM address which will receive the gas usage * gas price (set in transaction). Essentially, the transaction wrapper behaves similarly to a miner, receives the gas usage fees on an EVM address and pays for the transaction fees. 

Any failure during the execution would revert the whole Flow transaction. 

```
// Example of tx wrapping
import EVM from <ServiceAddress>

access(all)
fun main(rlpEncodedTransaction: [UInt8], coinbaseBytes: [UInt8; 20]) {
    let coinbase = EVM.EVMAddress(bytes: coinbaseBytes)
    EVM.run(tx: rlpEncodedTransaction, coinbase: coinbase)
}
```

For example, a user might use Metamask to sign a transaction for Flow EVM and broadcast it to services that check the gas fee on the transaction and wrap the transaction to be executed. 

Note that account nonce would protect against double execution of a transaction, similar to how other non-virtual blockchains prevent the minor from including a transaction multiple times. 

#### Bridged accounts

Another major goal for this work is seamless composability across environments. For this goal, we have introduced a new type of address a bridged account, to the EVM environment (besides EOA and Smart Contract accounts). This new address type is similar to EOA’s except instead of being tied to a public/private key it would be controlled by Cadence resources. Unlike EOAs which are created using the key presented by the wallet there is no corresponding EVM key present in a BridgedAccount. Any bridged account is interacted with through BridgedAccount resource and any Flow account or Cadence smart contract that holds this resource could interact with the EVM environment on behalf of the address that is stored in the resource. 

```
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
        /// Returns the address of the newly deployed contract.
        /// The value (balance) is taken from the EVM account.
        access(all)
        fun deploy(code: [UInt8], gasLimit: UInt64, value: Balance): EVMAddress 

        /// Calls a function with the given data.
        /// The execution is limited by the given amount of gas.
        /// The value (balance) is taken from the EVM account.
        access(all)
        fun call(to: EVMAddress, data: [UInt8], gasLimit: UInt64, value: Balance): [UInt8] 
    }

    /// Creates a new bridged account
    access(all)
    fun createBridgedAccount(): @BridgedAccount
}
```

Bridged account addresses are unique and allocated by the FVM and stored inside the resource. Calls through bridged accounts form a new type of transaction for the EVM that doesn’t require signatures and doesn’t need nonce checking. Bridged accounts could deploy smart contracts or make calls to the ones that are already deployed on Flow EVM. 

Bridged accounts also facilitate the withdrawal of Flow tokens back from the EVM balance environment into the Cadence environment through `withdraw`.

**What about other fungible and non-tokens?**

The term "bridged accounts" is used because their design facilitates the building of bridges. A Cadence smart contract can control a bridged account, enabling transactional operations between two environments. For instance, this smart contract can receive a Fungible on the Cadence side and send the equivalent on the EVM side to a specified address, all in a single Flow transaction. More details on this will be provided in subsequent updates.

#### Safety and Reproducibility of the EVM state

EVM state is not accessible by Cadence outside of the defined interfaces above, and Cadence state is also protected against raw access from the EVM. 

At the start, the EVM state is empty (empty root hash), and all the state is stored under a Flow account that is not controlled by anyone (network-owned). Any interaction with this environment emits a transactionExecuted event (direct calls for bridged accounts use `255` as transaction type).  

So anyone following these events could reconstruct the whole EVM state by re-executing these transactions.

### Dependencies

The project introduces no new dependencies from the Ethereum codebase since those required are already included in flow-go

### Tutorials and Examples

A proof of concept implementation of this proposal is available [here](https://github.com/onflow/flow-emulator/releases/tag/v0.57.4-evm-poc). 

What parts of the design still need to be defined?