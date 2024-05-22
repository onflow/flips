---
status: proposed
flip: 223
authors: Ramtin Seraj (ramtin.seraj@flowfoundation.org), Bastian Müller (bastian.mueller@flowfoundation.org), Gregor Gololicic (gregor.gololicic@flowfoundation.org)
sponsor: Dieter Shirley (dete@dapperlabs.com)
updated: 2023-12-04
---

# FLIP 223: EVM integration interface

## Objective

Integrating the Ethereum Virtual Machine (EVM) into Cadence without compromising composability and enabling seamless interactions between Cadence and EVM environments

## Motivation

Following the discussion [here](https://forum.flow.com/t/evm-on-flow-beyond-solidity/5260), this proposal outlines a path to achieve full EVM equivalence on Flow, enabling developers to deploy any Ethereum decentralized application (dApp) on Flow without making any code changes. This allows developers to fully leverage the native functionality of Flow. Trusted tools and protocols such as Uniswap, Opensea, Metamask, Chainlink Oracle, Layerzero, AAVE, Curve, Remix, and Hardhat will be readily compatible with Flow. Additionally, developers will still have the ability to write Cadence smart contracts to extend and enhance the functionality of Solidity smart contracts, ensuring full composability.

Support for EVM on Flow enables developers to leverage the network effects and tooling available in Solidity and EVM, while also benefiting from Flow's user-friendly features and mainstream focus for onboarding and user experience. Additionally, developers can take advantage of Cadence's unique capabilities.

## Design Proposal


### Flow EVM

**"Flow EVM"** is a virtual EVM-based blockchain deployed to a specific address (`<EVMAddress>`) on Flow; a child account of the service account. Flow EVM has its own dedicated EVM chain-id, which varies for Testnet and Mainnet. It utilizes the latest version of the EVM byte-code interpreter. While the initial release of Flow EVM provides behavior similar to Ethereum Mainnet after the Shapella upgrade (reference implementation: Geth v1.13), as EVM software improves new versions can be deployed and updated using Flow network height coordinated updates in the future. 

This environment, "Flow EVM", does *not* introduce a new native token and instead utilizes the FLOW token, the native token of the Flow network, for account balances and value transfers. No additional FLOW tokens are minted for "Flow EVM", and all existing FLOW tokens on the Flow network can freely move between Cadence and Flow EVM. While Cadence employs a user-friendly fixed-precision representation for tracking and storing fungible tokens like FLOW, within "Flow EVM", account balances and transfer values are denominated in a smaller unit of FLOW called Atto-FLOW. This is done to conform with the EVM-ecosystem conventions and is similar to how Wei ("atto-ETH") is used to store ETH values on Ethereum. (1 Flow = 10^18 Atto-FLOW).

"Flow EVM" has its own dedicated storage state, which is separate from the Cadence environment. This storage state is authenticated and verified by the Flow network. The separation of storage is necessary to prevent direct memory access and maintain the advanced safety properties of Cadence. The genesis state of the "Flow EVM" is empty, thus there are no accounts with non-zero balances.


However, a high degree of composability between Flow EVM and Cadence environments has been facilitated through several means.

1. **Cadence calls to the Flow EVM**: Cadence can send and receive data by making calls into the Flow EVM environment. 
2. **“Flow EVM” extended precompiles:** a set of smart contracts available on Flow EVM that can be used by other EVM smart contracts to access and interact with the Cadence world. 
3. **Cadence-Owned-Account (COA):** COA is a natively supported EVM smart contract wallet type that allows a Cadence resource to own and control an EVM address. This native wallet provides the primitives needed to bridge or control assets across Flow EVM and Cadence. 

Let’s take a closer look at what each of these means.

### Cadence direct calls to the Flow EVM:

"Flow EVM" can be accessed only through Flow transactions and scripts or from Cadence smart contracts. However, later in this document, we describe options that have been provided to facilitate interaction between platforms that are only EVM-friendly with "Flow EVM".


Within Cadence, the “Flow EVM” can be imported using 

```cadence
import EVM from <ServiceAddress>
```

The "Flow EVM" smart contract defines structures and resources and exposes functions for querying and interacting with the EVM state.

The very first type defined in this contract is `EVMAddress`. It is a Cadence type representing an EVM address and constructible with any sequence of 20 bytes `EVM.EVMAddress(bytes: bytes)`. Note that in the EVM world, there is no concept of accounts or a minimum balance requirement. Any sequence of bytes with a length of 20 is considered a valid address.

- These structures allow queries about the EVM addresses from the Cadence side
  - `balance() Balance` returns the balance of the address, returning a balance object instead of a basic type is considered to prevent flow to atto-flow conversion mistakes
  - `nonce() UInt64` returns the nonce of the address
  - `code(): [UInt8]` returns the code of the address (if is a smart contract account,  it returns empty otherwise)
  - `codeHash(): [UInt8]` returns the code hash of the address


```cadence
// Example of balance query
import EVM from <ServiceAddress>

access(all)
fun main(bytes: [UInt8; 20]) {
    let addr = EVM.EVMAddress(bytes: bytes)
    let bal = addr.balance()
}
```

### Running a transaction

`run` is the next crucial function in this contract which runs RLP-encoded EVM transactions. If the interaction with “Flow EVM” is successful and it changes the state, a new Flow-EVM block is formed and its data is emitted as a Cadence event.  Using `run` limits the EVM block to a single EVM transaction, a `batchRun` provides an option for batching EVM transaction execution in a single block. Using batch run you can provide an array of RLP-encoded EVM transactions as input and they will be all executed in a new block, the function will return an array of results `[EVM.Result]` which will be the same length as the array of input transactions and will match the order.

```cadence
import EVM from <ServiceAddress>

transaction(rlpEncodedTransaction: [UInt8], coinbaseBytes: [UInt8; 20]) {
    execute {
        let coinbase = EVM.EVMAddress(bytes: coinbaseBytes)
        let result = EVM.run(tx: rlpEncodedTransaction, coinbase: coinbase)
        assert(
            runResult.status == Status.successful,
            message: "tx was not executed successfully."
        )
    }
}
```

Example of batch run:
```
import EVM from <ServiceAddress>

transaction(rlpEncodedTransactions: [[UInt8]], coinbaseBytes: [UInt8; 20]) {
    execute {
      let coinbase = EVM.EVMAddress(bytes: coinbaseBytes)
      let results = EVM.batchRun(txs: txs, coinbase: coinbase)
      
      assert(results.length == txs.length, message: "invalid result length")
      for res in results {
        assert(res.status == EVM.Status.successful, message: "unexpected status")
      }
    }
}

```

Note that calling `EVM.run` or `EVM.batchRun` doesn't revert the outer flow transaction and it requires the developer to take proper action based on the result.Status. 

All transaction run variants produce the Result as the return value.

#### Result
The result type returned from any run variant contains important values:

- `status: Status` 
  - `Status.invalid`: The execution of an EVM transaction/call has failed at the validation step (e.g. nonce mismatch). An invalid transaction/call is rejected to be executed or be included in a block (no state change).
  - `Status.failed`: The execution of an EVM transaction/call has been successful but the VM has reported an error as the outcome of execution (e.g. running out of gas). A failed tx/call is included in a block.
  Note that resubmission of a failed transaction would result in invalid status on the second attempt, given the nonce would become invalid.
  
  - `Status.successful`: The execution of an EVM transaction/call has been successful and no error is reported by the VM.

- `errorCode: UInt64`: Specific error code that caused the failure, a value of `0` means no error. Error codes are divided into validation error codes (201-300) and execution error codes (301-400), see Appendix D.
- `gasUsed: UInt64`: The amount of gas the transaction used for the execution
- `data: [UInt8]`: Contains any data that is returned from the EVM when using "call". When deploying a contract this will contain the deployed code.
- `deployedContract: EVMAddress?`: This is an optional field, which is only set when a new contract was deployed, it will contain the address of the newly deployed contract, otherwise, it will be nil


There are multiple variants of the run method for executing a transaction each having its own unique abilities:
- `run`: will execute a transaction, and create a new block containing that transaction only. If the transaction fails, the Flow transaction won't be reverted.
- `mustRun`: will execute a transaction, and create a new block containing that transaction only. If the transaction fails, it will revert the Flow transaction as well.
- `batchRun`: will execute a list of transactions and create a new block containing all the transactions. If the transaction fails, the Flow transaction won't be reverted.
- `dryRun`: will simulate transaction execution, but won't commit any changes. This is useful for estimating gas usage or calling view methods on contracts. If run in a Cadence script it won't consume any gas. 

The gas used during the method calls is aggregated, adjusted, and added to the total computation fees of the Flow transaction and paid by the payer. The adjustment is done by multiplying the gas usage by a multiplier set by the network and adjustable by the service account.  

Please refer to Appendix B for the full list of types and functions available in the Flow EVM contract. 

#### Events
Each newly created block or executed transaction will emit an event:
- Transaction Executed: each executed transaction will emit a Flow event, that will have the type ID: `A.<ServiceAddress>.EVM.TransactionExecuted` and will contain the following fields: `hash`, `index`, `type`, `payload` (RLP and hex-encoded transaction payload), `errorCode` (matches the code in the Result type), `gasConsumed`, `contractAddress` (provided in case of contract deployment), `logs`, `blockHeight`, `blockHash`.
- Block Executed: each newly created block will emit a Flow event, that will have the type ID: `A.<ServiceAddress>.EVM.BlockExecuted` and will contain the following fields: `height`, `hash`, `timestamp`, `totalSupply`, `totalGasUsed`, `parentHash`, `receiptRoot`, `transactionHashes` (list of all transaction hashes included in the block).


### “Flow EVM” extended precompiles

Precompiles are smart contracts built into the EVM environment yet not implemented in Solidity (learn more about precompiles [here](https://www.evm.codes/precompiled)). Flow EVM environment extends the original set the standard precompiles with a few more precompile smart contracts. This extended set of precompile contracts are deployed at addresses prefixed `0x000000000000000000000001` leaving a huge space (`2^64`) available for the future standard precompiles.


#### Cadence Arch 
Cadence Arch is the very first augmented precompiled smart contract. Cadence Arch is a multi-function smart contract (deployed at `0x0000000000000000000000010000000000000001`) that allows any smart contract on Flow EVM to interact with the Cadence side.

Here is the list of some of the functions available on the Cadence Arch smart contract in the first release: 

- `FlowBlockHeight() uint64` (signature: 0x53e87d66) returns the current flow block height, this could be used instead of flow evm block heights to trigger scheduled actions given it's more predictable when a block might be formed. 

- `VerifyCOAOwnershipProof(bytes32 _hash, bytes memory _signature)(bool success)` returns true if proof is valid. An ownership proof verifies that a Flow wallet controls a COA account. This is done by checking signatures and their weights, loading the COA resource from the specified storage path and check the EVM address of the COA resource. More details are available in the next section.

- `getRandomSource(uint64) uint64` returns a secure on-chain random source. This can be used for the creation of PRGs (learn more about [secure random on Flow here](https://developers.flow.com/build/advanced-concepts/randomness)).

- `revertibleRandom() uint64` returns a pseudo-random value that is produced by the Flow PRG and uses a secure on-chain random source with salt as the seed to the generator. Using this random value is safe, but it allows for a transaction to revert in case of an unfavorable outcome, so it must be used carefully and it's best to [follow safety guidelines](https://developers.flow.com/build/advanced-concepts/randomness).

Cadence arch can be updated over time with more functions, some could trigger actions on the Cadence side, but there would be follow-up Flips for it. 


### Cadence-Owned-Account (COA)

Another native tool ensuring seamless composability across environments is Cadence Owned Account. *COA* is a natively supported smart contract wallet type on the Flow EVM, facilitating composability between Cadence and EVM environments; COA is an EVM smart contract wallet controlled by a Cadence resource:

- it's a smart contract deployed to Flow EVM, is accessible by other Flow EVM users, and can accept and control EVM assets such as ERC721s. However, unlike other EVM smart contracts, they can initiate transactions (COA’s EVM address acts as `tx.origin`). This behavior is different than other EVM environments in that only EOA accounts can initiate a transaction. A new EVM transaction type (`TxType = 0xff`) is used to differentiate these transactions from other types of EVM transactions (e.g, DynamicFeeTxType (`0x02`). 

- its owned and controlled by a Cadence resource, which means they don’t need an EVM transaction to be triggered (e.g. a transaction to make a call to `execute` or EIP-4337’s `validateUserOpmethod`). Instead, using the Cadence interface on the resource, direct call transactions can be triggered. Calling this method emits direct call transaction events on the Flow EVM side.

Each COA smart contract can only be deployed through the Cadence side.
`EVM.createCadenceOwnedAccount(): @CadenceOwnedAccount` constructs and returns a Cadence resource, allocates a unique Flow EVM address (based on the UUID of the resource and with the prefix `0x000000000000000000000002`), and deploys the smart contract wallet byte codes to the given address. The address `0x0000000000000000000000020000000000000000` is reserved for COA factory, an address that deploys contracts for COA accounts. 

The Cadence resource associated with the smart contract like other Cadence resources can be stored in a default storage path in a Flow account or can be owned by a Cadence smart contract. 

As mentioned earlier COAs expose two interfaces for interaction, one on the Flow EVM side and one on the Cadence resource side. Let's take a closer look at each: 


#### COA’s Cadence resource interface

- `address(): EVMAddress` returns the address of the smart contract, and the EVM address that is returned could be used to query balance, code, nonce, etc.

- `deposit(from: @FlowToken.Vault)` allows depositing FLOW tokens into the smart contract (Cadence to Flow EVM). On the EVM side, the money for the deposits is always transferred from `0x0000000000000000000000010000000000000000` (native token bridge address). The balance of that address is adjusted before transfer.  

- `withdraw(balance: Balance): @FlowToken.Vault` allows withdrawing balance from the Flow EVM address and bridges it back as a FlowToken Vault to be handled on the Cadence side. On the EVM side, the money for the withdrawal is always transferred to `0x0000000000000000000000010000000000000000` (native token bridge address) and then the balance of that address is adjusted.  

- `deploy(code: [UInt8], gasLimit: UInt64, value: Balance): Result` lets the COA smart contract deploy smart contracts, and returns the result containing the newly deployed contract address. The value (balance) is taken from the COA smart contract and moved to the new smart contract address (if they accept it). 

- `call(to: EVMAddress, data: [UInt8], gasLimit: UInt64, value: Balance): Result` makes a call on behalf of the COA smart contract and returns the result of the call which could be processed by the Cadence side. The value (balance) is taken from the COA smart contract. Calling this method emits direct call transaction events. 


#### COA’s EVM smart contract interface

- `function supportsInterface(bytes4 interfaceId) external view returns (bool)` provides the functionality needed to satisfy EIP-165. If the given interfaceID is available it returns true. 

- `receive() external payable` is the standard method needed to allow other EVM users to send Flow to the smart contract directly, if data is passed to this method it would revert. This method is not usually directly called and any call to smart contract it without the data is automatically rerouted to this function.

- `function onERC721Received(address _operator, address _from, uint256 _tokenId, bytes calldata _data) external returns (bytes4)` is provided to support safe transfers from ERC721 asset contracts. It's called on the recipient after a transfer. Return of any value other than the magic value (`0x150b7a02`, as defined in the [ERC-721 standard](https://eips.ethereum.org/EIPS/eip-721)) must result in the transaction being reverted. 

- `function tokensReceived(address operator, address from, address to, uint256 amount, bytes calldata data, bytes calldata operatorData) external` is provided to support safe transfers from ERC777 asset contracts. is called by the ERC777 token contract after a successful transfer or a minting operation. It doesn’t return anything.

- `function onERC1155Received(address _operator, address _from, uint256 _id, uint256 _value, bytes calldata _data) external returns (bytes4)` is provided to support safe transfers from ERC1155 asset contracts.  It should return 0xf23a6e61 if supporting the receiving of a single ERC1155 token type. 

- `function onERC1155BatchReceived(address _operator, address _from, uint256[] calldata _ids, uint256[] calldata _values, bytes calldata _data) external returns (bytes4)` is provided to support safe transfers from ERC1155 asset contracts (batch of assets). It returns `0xbc197c81` if supporting the receiving of a batch of ERC1155 token types.

- `function isValidSignature(bytes32 _hash, bytes memory _signature) external view virtual returns (bytes4)` returns the bytes4 magic value `0x1626ba7e` (as defined in [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271)) when the signature is valid. This method is usually used to verify a personal sign for the smart contract wallets (see EIP-1271 for more details). In the context of COA smart contracts, we consider the signature as an aggregation of Flow account address, key index, path to COA resource, and a set of signatures (Flow account). we return true if the signatures are valid, it provides enough weight for the account, and an account holds the resource at the given path. Under the hood, this method uses a Cadence Arch contract for verification.


## Appendix A - Embracing the EVM ecosystem.

In this FLIP, we described the foundation of the Flow EVM, yet there are other works built on top of this foundation to ensure an effortless onboarding experience for the existing EVM ecosystem product and tools. 

**Flow EVM Gateway Software**

A separate software package is provided by the Flow Foundation that can be run by anyone (including the Flow Foundation), follows the Flow EVM chain, sends queries to the Flow access nodes, and provides the JSON-RPC endpoints to any 3rd party applications that want to interact directly with the Flow EVM. With the exception of account state-proof endpoints, all other endpoints of the JSON-RPC will be provided at the release. 

Any 3rd party running this software can accept EVM RLP encoded transactions from users and wrap them in a Cadence transaction.  While they will pay for the fees, `EVM.run` provides a utility parameter (`coinbase`) to collect the gas fee from the user on the Flow EVM side while running the EVM transaction. The use of the `coinbase` address in this context indicates the EVM address which will receive the `gas usage * gas price` (set in a transaction). Essentially, the transaction wrapper behaves similarly to a miner, receives the gas usage fees on an EVM address, and pays for the transaction fees. The `gas price per unit of gas` creates a marketplace for these 3rd parties to compete over transactions. 

For example, a user might use MetaMask to sign a transaction for Flow EVM and broadcast it to services that check the gas fee on the transaction and wrap the transaction to be executed. Note that account nonce would protect against double execution of a transaction, similar to how other non-virtual blockchains prevent the miner from including a transaction multiple times.

As EVM interactions are encapsulated within Flow transactions, they leverage the security measures provided by Flow. These transactions undergo the same process of collection, execution, and verification as other Flow transactions, without any EVM intervention. Consequently, there is no requirement for intricate block formation logic (such as handling forks and reorganizations), mempools, or additional safeguards against malicious MEV (Miner Extractable Value) behaviors. More information about Flow's consensus model is available [here](https://flow.com/core-protocol).

You can read more about this work in the [Flow EVM Gateway](https://github.com/onflow/flips/pull/235) improvement proposal. 

**Flow VM Bridge (Cadence <> Flow EVM)** 

COAs provide out-of-the-box $FLOW token bridging between environments (see `deposit(from: @FlowToken.Vault)`). They are also powerful resources that integrate native cross-VM bridging capabilities through which applications can bridge arbitrary fungible and/or non-fungible tokens between Cadence and Flow EVM. Check out the [Flow VM Bridge ](https://github.com/onflow/flips/pull/233) improvement proposal for more details.

Note that transferring $FLOW from COAs to EOAs is not dependent on the VM bridge. This can be done in a similar manner to transfers in other EVM environments via a call without data transmitting value, as showcased below:

```cadence
/// Assuming the signer has a COA in storage & an EVM $FLOW balance to cover the amount specified...
transaction(to: EVM.EVMAddress, amount: UFix64) {
    prepare(signer: auth(BorrowValue) &Account) {
        let coa = signer.storage.borrow<&EVM.CadenceOwnedAccount>(from: /storage/evm)!
        coa.call(
            to: to,
            data: [],
            gasLimit: gasLimit,
            value: EVM.Balance().setFlow(flow: amount),
        )
    }
}
```


## Appendix B - “Flow EVM”’s smart contract (in Cadence)

```cadence
import Crypto
import "NonFungibleToken"
import "FungibleToken"
import "FlowToken"

access(all)
contract EVM {

    // Entitlements enabling finer-graned access control on a CadenceOwnedAccount
    access(all) entitlement Validate
    access(all) entitlement Withdraw
    access(all) entitlement Call
    access(all) entitlement Deploy
    access(all) entitlement Owner
    access(all) entitlement Bridge

    /// Block executed event is emitted when a new block is created,
    /// which always happens when a transaction is executed.
    access(all)
    event BlockExecuted(
        // height or number of the block
        height: UInt64,
        // hash of the block
        hash: String,
        // timestamp of the block creation
        timestamp: UInt64,
        // total Flow supply
        totalSupply: Int,
        // all gas used in the block by transactions included
        totalGasUsed: UInt64,
        // parent block hash
        parentHash: String,
        // hash of all the transaction receipts
        receiptRoot: String,
        // all the transactions included in the block
        transactionHashes: [String]
    )

    /// Transaction executed event is emitted everytime a transaction
    /// is executed by the EVM (even if failed).
    access(all)
    event TransactionExecuted(
        // hash of the transaction
        hash: String,
        // index of the transaction in a block
        index: UInt16,
        // type of the transaction
        type: UInt8,
        // RLP and hex encoded transaction payload
        payload: String,
        // code indicating a specific validation (201-300) or execution (301-400) error
        errorCode: UInt16,
        // the amount of gas transaction used
        gasConsumed: UInt64,
        // if transaction was a deployment contains a newly deployed contract address
        contractAddress: String,
        // RLP and hex encoded logs
        logs: String,
        // block height in which transaction was inclued
        blockHeight: UInt64,
        // block hash in which transaction was included
        blockHash: String
    )

    access(all)
    event CadenceOwnedAccountCreated(address: String)

    /// FLOWTokensDeposited is emitted when FLOW tokens is bridged
    /// into the EVM environment. Note that this event is not emitted
    /// for transfer of flow tokens between two EVM addresses.
    access(all)
    event FLOWTokensDeposited(address: String, amount: UFix64)

    /// FLOWTokensWithdrawn is emitted when FLOW tokens are bridged
    /// out of the EVM environment. Note that this event is not emitted
    /// for transfer of flow tokens between two EVM addresses.
    access(all)
    event FLOWTokensWithdrawn(address: String, amount: UFix64)

    /// BridgeAccessorUpdated is emitted when the BridgeAccessor Capability
    /// is updated in the stored BridgeRouter along with identifying
    /// information about both.
    access(all)
    event BridgeAccessorUpdated(
        routerType: Type,
        routerUUID: UInt64,
        routerAddress: Address,
        accessorType: Type,
        accessorUUID: UInt64,
        accessorAddress: Address
    )

    /// EVMAddress is an EVM-compatible address
    access(all)
    struct EVMAddress {

        /// Bytes of the address
        access(all)
        let bytes: [UInt8; 20]

        /// Constructs a new EVM address from the given byte representation
        view init(bytes: [UInt8; 20]) {
            self.bytes = bytes
        }

        /// Balance of the address
        access(all)
        view fun balance(): Balance {
            let balance = InternalEVM.balance(
                address: self.bytes
            )
            return Balance(attoflow: balance)
        }

        /// Nonce of the address
        access(all)
        fun nonce(): UInt64 {
            return InternalEVM.nonce(
                address: self.bytes
            )
        }

        /// Code of the address
        access(all)
        fun code(): [UInt8] {
            return InternalEVM.code(
                address: self.bytes
            )
        }

        /// CodeHash of the address
        access(all)
        fun codeHash(): [UInt8] {
            return InternalEVM.codeHash(
                address: self.bytes
            )
        }

        /// Deposits the given vault into the EVM account with the given address
        access(all)
        fun deposit(from: @FlowToken.Vault) {
            let amount = from.balance
            if amount == 0.0 {
                panic("calling deposit function with an empty vault is not allowed")
            }
            InternalEVM.deposit(
                from: <-from,
                to: self.bytes
            )
            emit FLOWTokensDeposited(address: self.toString(), amount: amount)
        }

        /// Serializes the address to a hex string without the 0x prefix
        /// Future implementations should pass data to InternalEVM for native serialization
        access(all)
        view fun toString(): String {
            return String.encodeHex(self.bytes.toVariableSized())
        }

        /// Compares the address with another address
        access(all)
        view fun equals(_ other: EVMAddress): Bool {
            return self.bytes == other.bytes
        }
    }

    /// Converts a hex string to an EVM address if the string is a valid hex string
    /// Future implementations should pass data to InternalEVM for native deserialization
    access(all)
    fun addressFromString(_ asHex: String): EVMAddress {
        pre {
            asHex.length == 40 || asHex.length == 42: "Invalid hex string length for an EVM address"
        }
        // Strip the 0x prefix if it exists
        var withoutPrefix = (asHex[1] == "x" ? asHex.slice(from: 2, upTo: asHex.length) : asHex).toLower()
        let bytes = withoutPrefix.decodeHex().toConstantSized<[UInt8;20]>()!
        return EVMAddress(bytes: bytes)
    }

    access(all)
    struct Balance {

        /// The balance in atto-FLOW
        /// Atto-FLOW is the smallest denomination of FLOW (1e18 FLOW)
        /// that is used to store account balances inside EVM
        /// similar to the way WEI is used to store ETH divisible to 18 decimal places.
        access(all)
        var attoflow: UInt

        /// Constructs a new balance
        access(all)
        view init(attoflow: UInt) {
            self.attoflow = attoflow
        }

        /// Sets the balance by a UFix64 (8 decimal points), the format
        /// that is used in Cadence to store FLOW tokens.
        access(all)
        fun setFLOW(flow: UFix64){
            self.attoflow = InternalEVM.castToAttoFLOW(balance: flow)
        }

        /// Casts the balance to a UFix64 (rounding down)
        /// Warning! casting a balance to a UFix64 which supports a lower level of precision
        /// (8 decimal points in compare to 18) might result in rounding down error.
        /// Use the toAttoFlow function if you care need more accuracy.
        access(all)
        view fun inFLOW(): UFix64 {
            return InternalEVM.castToFLOW(balance: self.attoflow)
        }

        /// Returns the balance in Atto-FLOW
        access(all)
        view fun inAttoFLOW(): UInt {
            return self.attoflow
        }

        /// Returns true if the balance is zero
        access(all)
        fun isZero(): Bool {
            return self.attoflow == 0
        }
    }

    /// reports the status of evm execution.
    access(all) enum Status: UInt8 {
        /// is (rarely) returned when status is unknown
        /// and something has gone very wrong.
        access(all) case unknown

        /// is returned when execution of an evm transaction/call
        /// has failed at the validation step (e.g. nonce mismatch).
        /// An invalid transaction/call is rejected to be executed
        /// or be included in a block.
        access(all) case invalid

        /// is returned when execution of an evm transaction/call
        /// has been successful but the vm has reported an error as
        /// the outcome of execution (e.g. running out of gas).
        /// A failed tx/call is included in a block.
        /// Note that resubmission of a failed transaction would
        /// result in invalid status in the second attempt, given
        /// the nonce would be come invalid.
        access(all) case failed

        /// is returned when execution of an evm transaction/call
        /// has been successful and no error is reported by the vm.
        access(all) case successful
    }

    /// reports the outcome of evm transaction/call execution attempt
    access(all) struct Result {
        /// status of the execution
        access(all)
        let status: Status

        /// error code (error code zero means no error)
        access(all)
        let errorCode: UInt64

        /// returns the amount of gas metered during
        /// evm execution
        access(all)
        let gasUsed: UInt64

        /// returns the data that is returned from
        /// the evm for the call. For coa.deploy
        /// calls it returns the code deployed to
        /// the address provided in the contractAddress field.
        access(all)
        let data: [UInt8]

        /// returns the newly deployed contract address
        /// if the transaction caused such a deployment
        /// otherwise the value is nil.
        access(all)
        let deployedContract: EVMAddress?

        init(
            status: Status,
            errorCode: UInt64,
            gasUsed: UInt64,
            data: [UInt8],
            contractAddress: [UInt8; 20]?
        ) {
            self.status = status
            self.errorCode = errorCode
            self.gasUsed = gasUsed
            self.data = data

            if let addressBytes = contractAddress {
                self.deployedContract = EVMAddress(bytes: addressBytes)
            } else {
                self.deployedContract = nil
            }
        }
    }

    access(all)
    resource interface Addressable {
        /// The EVM address
        access(all)
        view fun address(): EVMAddress
    }

    access(all)
    resource CadenceOwnedAccount: Addressable {

        access(self)
        var addressBytes: [UInt8; 20]

        init() {
            // address is initially set to zero
            // but updated through initAddress later
            // we have to do this since we need resource id (uuid)
            // to calculate the EVM address for this cadence owned account
            self.addressBytes = [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
        }

        access(contract)
        fun initAddress(addressBytes: [UInt8; 20]) {
           // only allow set address for the first time
           // check address is empty
            for item in self.addressBytes {
                assert(item == 0, message: "address byte is not empty")
            }
           self.addressBytes = addressBytes
        }

        /// The EVM address of the cadence owned account
        access(all)
        view fun address(): EVMAddress {
            // Always create a new EVMAddress instance
            return EVMAddress(bytes: self.addressBytes)
        }

        /// Get balance of the cadence owned account
        access(all)
        view fun balance(): Balance {
            return self.address().balance()
        }

        /// Deposits the given vault into the cadence owned account's balance
        access(all)
        fun deposit(from: @FlowToken.Vault) {
            self.address().deposit(from: <-from)
        }

        /// The EVM address of the cadence owned account behind an entitlement, acting as proof of access
        access(Owner | Validate)
        view fun protectedAddress(): EVMAddress {
            return self.address()
        }

        /// Withdraws the balance from the cadence owned account's balance
        /// Note that amounts smaller than 10nF (10e-8) can't be withdrawn
        /// given that Flow Token Vaults use UFix64s to store balances.
        /// If the given balance conversion to UFix64 results in
        /// rounding error, this function would fail.
        access(Owner | Withdraw)
        fun withdraw(balance: Balance): @FlowToken.Vault {
            if balance.isZero() {
                panic("calling withdraw function with zero balance is not allowed")
            }
            let vault <- InternalEVM.withdraw(
                from: self.addressBytes,
                amount: balance.attoflow
            ) as! @FlowToken.Vault
            emit FLOWTokensWithdrawn(address: self.address().toString(), amount: balance.inFLOW())
            return <-vault
        }

        /// Deploys a contract to the EVM environment.
        /// Returns the result which contains address of
        /// the newly deployed contract
        access(Owner | Deploy)
        fun deploy(
            code: [UInt8],
            gasLimit: UInt64,
            value: Balance
        ): Result {
            return InternalEVM.deploy(
                from: self.addressBytes,
                code: code,
                gasLimit: gasLimit,
                value: value.attoflow
            ) as! Result
        }

        /// Calls a function with the given data.
        /// The execution is limited by the given amount of gas
        access(Owner | Call)
        fun call(
            to: EVMAddress,
            data: [UInt8],
            gasLimit: UInt64,
            value: Balance
        ): Result {
            return InternalEVM.call(
                from: self.addressBytes,
                to: to.bytes,
                data: data,
                gasLimit: gasLimit,
                value: value.attoflow
            ) as! Result
        }

        /// Bridges the given NFT to the EVM environment, requiring a Provider from which to withdraw a fee to fulfill
        /// the bridge request
        access(all)
        fun depositNFT(
            nft: @{NonFungibleToken.NFT},
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        ) {
            EVM.borrowBridgeAccessor().depositNFT(nft: <-nft, to: self.address(), feeProvider: feeProvider)
        }

        /// Bridges the given NFT from the EVM environment, requiring a Provider from which to withdraw a fee to fulfill
        /// the bridge request. Note: the caller should own the requested NFT in EVM
        access(Owner | Bridge)
        fun withdrawNFT(
            type: Type,
            id: UInt256,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        ): @{NonFungibleToken.NFT} {
            return <- EVM.borrowBridgeAccessor().withdrawNFT(
                caller: &self as auth(Call) &CadenceOwnedAccount,
                type: type,
                id: id,
                feeProvider: feeProvider
            )
        }

        /// Bridges the given Vault to the EVM environment, requiring a Provider from which to withdraw a fee to fulfill
        /// the bridge request
        access(all)
        fun depositTokens(
            vault: @{FungibleToken.Vault},
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        ) {
            EVM.borrowBridgeAccessor().depositTokens(vault: <-vault, to: self.address(), feeProvider: feeProvider)
        }

        /// Bridges the given fungible tokens from the EVM environment, requiring a Provider from which to withdraw a
        /// fee to fulfill the bridge request. Note: the caller should own the requested tokens & sufficient balance of
        /// requested tokens in EVM
        access(Owner | Bridge)
        fun withdrawTokens(
            type: Type,
            amount: UInt256,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        ): @{FungibleToken.Vault} {
            return <- EVM.borrowBridgeAccessor().withdrawTokens(
                caller: &self as auth(Call) &CadenceOwnedAccount,
                type: type,
                amount: amount,
                feeProvider: feeProvider
            )
        }
    }

    /// Creates a new cadence owned account
    access(all)
    fun createCadenceOwnedAccount(): @CadenceOwnedAccount {
        let acc <-create CadenceOwnedAccount()
        let addr = InternalEVM.createCadenceOwnedAccount(uuid: acc.uuid)
        acc.initAddress(addressBytes: addr)

        emit CadenceOwnedAccountCreated(address: acc.address().toString())
        return <-acc
    }

    /// Runs an a RLP-encoded EVM transaction, deducts the gas fees,
    /// and deposits the gas fees into the provided coinbase address.
    access(all)
    fun run(tx: [UInt8], coinbase: EVMAddress): Result {
        return InternalEVM.run(
                tx: tx,
                coinbase: coinbase.bytes
        ) as! Result
    }

    /// mustRun runs the transaction using EVM.run yet it
    /// rollback if the tx execution status is unknown or invalid.
    /// Note that this method does not rollback if transaction
    /// is executed but an vm error is reported as the outcome
    /// of the execution (status: failed).
    access(all)
    fun mustRun(tx: [UInt8], coinbase: EVMAddress): Result {
        let runResult = self.run(tx: tx, coinbase: coinbase)
        assert(
            runResult.status == Status.failed || runResult.status == Status.successful,
            message: "tx is not valid for execution"
        )
        return runResult
    }

    /// Simulates running unsigned RLP-encoded transaction using
    /// the from address as the signer.
    /// The transaction state changes are not persisted.
    /// This is useful for gas estimation or calling view contract functions.
    access(all)
    fun dryRun(tx: [UInt8], from: EVMAddress): Result {
        return InternalEVM.dryRun(
            tx: tx,
            from: from.bytes,
        ) as! Result
    }

    /// Runs a batch of RLP-encoded EVM transactions, deducts the gas fees,
    /// and deposits the gas fees into the provided coinbase address.
    /// An invalid transaction is not executed and not included in the block.
    access(all)
    fun batchRun(txs: [[UInt8]], coinbase: EVMAddress): [Result] {
        return InternalEVM.batchRun(
            txs: txs,
            coinbase: coinbase.bytes,
        ) as! [Result]
    }

    access(all)
    fun encodeABI(_ values: [AnyStruct]): [UInt8] {
        return InternalEVM.encodeABI(values)
    }

    access(all)
    fun decodeABI(types: [Type], data: [UInt8]): [AnyStruct] {
        return InternalEVM.decodeABI(types: types, data: data)
    }

    access(all)
    fun encodeABIWithSignature(
        _ signature: String,
        _ values: [AnyStruct]
    ): [UInt8] {
        let methodID = HashAlgorithm.KECCAK_256.hash(
            signature.utf8
        ).slice(from: 0, upTo: 4)
        let arguments = InternalEVM.encodeABI(values)

        return methodID.concat(arguments)
    }

    access(all)
    fun decodeABIWithSignature(
        _ signature: String,
        types: [Type],
        data: [UInt8]
    ): [AnyStruct] {
        let methodID = HashAlgorithm.KECCAK_256.hash(
            signature.utf8
        ).slice(from: 0, upTo: 4)

        for byte in methodID {
            if byte != data.removeFirst() {
                panic("signature mismatch")
            }
        }

        return InternalEVM.decodeABI(types: types, data: data)
    }

    /// ValidationResult returns the result of COA ownership proof validation
    access(all)
    struct ValidationResult {
        access(all)
        let isValid: Bool

        access(all)
        let problem: String?

        init(isValid: Bool, problem: String?) {
            self.isValid = isValid
            self.problem = problem
        }
    }

    /// validateCOAOwnershipProof validates a COA ownership proof
    access(all)
    fun validateCOAOwnershipProof(
        address: Address,
        path: PublicPath,
        signedData: [UInt8],
        keyIndices: [UInt64],
        signatures: [[UInt8]],
        evmAddress: [UInt8; 20]
    ): ValidationResult {

        // make signature set first
        // check number of signatures matches number of key indices
        if keyIndices.length != signatures.length {
            return ValidationResult(
                isValid: false,
                problem: "key indices size doesn't match the signatures"
            )
        }

        var signatureSet: [Crypto.KeyListSignature] = []
        for signatureIndex, signature in signatures{
            signatureSet.append(Crypto.KeyListSignature(
                keyIndex: Int(keyIndices[signatureIndex]),
                signature: signature
            ))
        }

        // fetch account
        let acc = getAccount(address)

        // constructing key list
        let keyList = Crypto.KeyList()
        for signature in signatureSet {
            let key = acc.keys.get(keyIndex: signature.keyIndex)!
            assert(!key.isRevoked, message: "revoked key is used")
            keyList.add(
              key.publicKey,
              hashAlgorithm: key.hashAlgorithm,
              weight: key.weight,
           )
        }

        let isValid = keyList.verify(
            signatureSet: signatureSet,
            signedData: signedData,
            domainSeparationTag: "FLOW-V0.0-user"
        )

        if !isValid{
            return ValidationResult(
                isValid: false,
                problem: "the given signatures are not valid or provide enough weight"
            )
        }

        let coaRef = acc.capabilities.borrow<&EVM.CadenceOwnedAccount>(path)
        if coaRef == nil {
             return ValidationResult(
                 isValid: false,
                 problem: "could not borrow bridge account's resource"
             )
        }

        // verify evm address matching
        var addr = coaRef!.address()
        for index, item in coaRef!.address().bytes {
            if item != evmAddress[index] {
                return ValidationResult(
                    isValid: false,
                    problem: "evm address mismatch"
                )
            }
        }

        return ValidationResult(
            isValid: true,
            problem: nil
        )
    }

    /// Block returns information about the latest executed block.
    access(all)
    struct EVMBlock {
        access(all)
        let height: UInt64

        access(all)
        let hash: String

        access(all)
        let totalSupply: Int

        access(all)
        let timestamp: UInt64

        init(height: UInt64, hash: String, totalSupply: Int, timestamp: UInt64) {
            self.height = height
            self.hash = hash
            self.totalSupply = totalSupply
            self.timestamp = timestamp
        }
    }

    /// Returns the latest executed block.
    access(all)
    fun getLatestBlock(): EVMBlock {
        return InternalEVM.getLatestBlock() as! EVMBlock
    }

    /// Interface for a resource which acts as an entrypoint to the VM bridge
    access(all)
    resource interface BridgeAccessor {

        /// Endpoint enabling the bridging of an NFT to EVM
        access(Bridge)
        fun depositNFT(
            nft: @{NonFungibleToken.NFT},
            to: EVMAddress,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        )

        /// Endpoint enabling the bridging of an NFT from EVM
        access(Bridge)
        fun withdrawNFT(
            caller: auth(Call) &CadenceOwnedAccount,
            type: Type,
            id: UInt256,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        ): @{NonFungibleToken.NFT}

        /// Endpoint enabling the bridging of a fungible token vault to EVM
        access(Bridge)
        fun depositTokens(
            vault: @{FungibleToken.Vault},
            to: EVMAddress,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        )

        /// Endpoint enabling the bridging of fungible tokens from EVM
        access(Bridge)
        fun withdrawTokens(
            caller: auth(Call) &CadenceOwnedAccount,
            type: Type,
            amount: UInt256,
            feeProvider: auth(FungibleToken.Withdraw) &{FungibleToken.Provider}
        ): @{FungibleToken.Vault}
    }

    /// Interface which captures a Capability to the bridge Accessor, saving it within the BridgeRouter resource
    access(all)
    resource interface BridgeRouter {

        /// Returns a reference to the BridgeAccessor designated for internal bridge requests
        access(Bridge) view fun borrowBridgeAccessor(): auth(Bridge) &{BridgeAccessor}

        /// Sets the BridgeAccessor Capability in the BridgeRouter
        access(Bridge) fun setBridgeAccessor(_ accessor: Capability<auth(Bridge) &{BridgeAccessor}>) {
            pre {
                accessor.check(): "Invalid BridgeAccessor Capability provided"
                emit BridgeAccessorUpdated(
                    routerType: self.getType(),
                    routerUUID: self.uuid,
                    routerAddress: self.owner?.address ?? panic("Router must have an owner to be identified"),
                    accessorType: accessor.borrow()!.getType(),
                    accessorUUID: accessor.borrow()!.uuid,
                    accessorAddress: accessor.address
                )
            }
        }
    }

    /// Returns a reference to the BridgeAccessor designated for internal bridge requests
    access(self)
    view fun borrowBridgeAccessor(): auth(Bridge) &{BridgeAccessor} {
        return self.account.storage.borrow<auth(Bridge) &{BridgeRouter}>(from: /storage/evmBridgeRouter)
            ?.borrowBridgeAccessor()
            ?? panic("Could not borrow reference to the EVM bridge")
    }
}
```

## Appendix C - COA’s smart contract (in Solidity)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.7.0 <0.9.0;

interface IERC165 {
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
}

interface ERC721TokenReceiver {
    function onERC721Received(
       address _operator, 
       address _from, 
       uint256 _tokenId, 
       bytes calldata _data
    ) external returns (bytes4);
}

interface ERC777TokensRecipient {
  function tokensReceived(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes calldata data,
        bytes calldata operatorData
    ) external;
}

interface ERC1155TokenReceiver {
  function onERC1155Received(
        address _operator,
        address _from,
        uint256 _id,
        uint256 _value,
        bytes calldata _data
    ) external returns (bytes4);
  function onERC1155BatchReceived(
        address _operator,
        address _from,
        uint256[] calldata _ids,
        uint256[] calldata _values,
        bytes calldata _data
    ) external returns (bytes4);
}

contract COA is ERC1155TokenReceiver, ERC777TokensRecipient, ERC721TokenReceiver, IERC165 {

    address constant public cadenceArch = 0x0000000000000000000000010000000000000001;

    event FlowReceived(address indexed sender, uint256 value);

    receive() external payable  { 
        emit FlowReceived(msg.sender, msg.value);
    }

    function supportsInterface(bytes4 id) external view virtual override returns (bool) {
        return
            id == type(ERC1155TokenReceiver).interfaceId ||
            id == type(ERC721TokenReceiver).interfaceId ||
            id == type(ERC777TokensRecipient).interfaceId ||
            id == type(IERC165).interfaceId;
    }

    function onERC1155Received(
    	address, 
    	address, 
    	uint256, 
    	uint256, 
    	bytes calldata
    ) external pure override returns (bytes4) {
        return 0xf23a6e61;
    }

    function onERC1155BatchReceived(
        address,
        address,
        uint256[] calldata,
        uint256[] calldata,
        bytes calldata
    ) external pure override returns (bytes4) {
        return 0xbc197c81;
    }

    function onERC721Received(
        address, 
        address, 
        uint256, 
        bytes calldata
    ) external pure override returns (bytes4) {
        return 0x150b7a02;
    }

    function tokensReceived(
        address, 
        address, 
        address, 
        uint256, 
        bytes calldata, 
        bytes calldata
    ) external pure override {}

    function isValidSignature(
        bytes32 _hash,
        bytes memory _sig
    ) external view virtual returns (bytes4){
        (bool ok, bytes memory data) = cadenceArch.staticcall(abi.encodeWithSignature("verifyCOAOwnershipProof(bytes32, bytes)", _hash, _sig));
        require(ok);
        bool output = abi.decode(data, (bool));
        if (output) {
            return 0x1626ba7e;
        }
        return 0xffffffff;
    }
}
```

## Appendix D - Status error codes


| Error code | Category | Description |
|------------|----------|-------------|
| 0          | -           | no error    |
| 100        | validation | Generic validation error that is returned for cases that don't have a specific code. |
| 101        | validation | Invalid balance is provided (e.g. negative value). |
| 102        | validation | Insufficient computation is left in the flow transaction. |
| 103        | validation | An unauthorized method has been called. |
| 104        | validation | Withdraw balance is prone to rounding error. |
| 201        | validation | The nonce of the transaction is lower than expected. |
| 202        | validation | The nonce of the transaction is higher than expected. |
| 203        | validation | The transaction's sender account has reached the maximum nonce. |
| 204        | validation | Not enough gas is available on the block to include this transaction. |
| 205        | validation | The transaction sender doesn't have enough funds for transfer(topmost call only). |
| 206        | validation | The contract creation transaction provides the init code bigger than the init code size limit. |
| 207        | validation | The total cost of executing a transaction is higher than the balance of the user's account. |
| 208        | validation | An overflow is detected when calculating the gas usage. |
| 209        | validation | The transaction is specified to use less gas than required to start the invocation. |
| 210        | validation | The transaction is not supported in the current network configuration. |
| 211        | validation | The tip was set to higher than the total fee cap. |
| 212        | validation | An extremely big number is set for the tip field. |
| 213        | validation | An extremely big number is set for the fee cap field. |
| 214        | validation | The transaction fee cap is less than the base fee of the block. |
| 215        | validation | The sender of a transaction is a contract. |
| 216        | validation | The transaction fee cap is less than the blob gas fee of the block. |
| 301        | execution | "execution ran out of gas" is returned by the VM. |
| 302        | execution | "contract creation code storage out of gas" is returned by the VM. |
| 303        | execution | "max call depth exceeded" is returned by the VM. |
| 304        | execution | "insufficient balance for transfer" is returned by the VM.  |
| 305        | execution | "contract address collision" is returned by the VM. |
| 306        | execution | "execution reverted" is returned by the VM. |
| 307        | execution | "max initcode size exceeded" is returned by the VM. |
| 308        | execution | "max code size exceeded" is returned by the VM. |
| 309        | execution | "invalid jump destination" is returned by the VM. |
| 310        | execution | "write protection" is returned by the VM. |
| 311        | execution | "return data out of bounds" is returned by the VM. |
| 312        | execution | "gas uint64 overflow" is returned by the VM |
| 313        | execution | "invalid code: must not begin with 0xef" is returned by the VM. |
| 314        | execution | "nonce uint64 overflow" is returned by the VM. |
| 400        | execution  | Generic execution error returned for cases that don't have a specific code |
