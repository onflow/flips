---
status: draft
flip: 223
authors: Ramtin Seraj (ramtin.seraj@flowfoundation.org), Bastian Müller (bastian@dapperlabs.com)
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

**"Flow EVM"** is a virtual EVM-based blockchain deployed to a specific address (`<EVMAddress>`) on Flow; a child account of the service account. Flow EVM has its own dedicated EVM chain-id, which varies for Testnet and Mainnet. It utilizes the latest version of the EVM byte-code interpreter. While the initial release of Flow EVM provides behaviour similar to Ethereum Mainnet after Shapella upgrade (reference implementation: Geth v1.13), as EVM software improves new versions can be deployed updated using Flow network height coordinated updates in the future. 

This environment, "Flow EVM", does *not* introduce a new native token and instead utilizes the FLOW token, native token of the Flow network, for account balances and value transfers. No additional FLOW tokens are minted for "Flow EVM", and all existing FLOW tokens on the Flow network can freely move between Cadence and Flow EVM. While Cadence employs a user-friendly fixed-precision representation for tracking and storing fungible tokens like FLOW, within "Flow EVM", account balances and transfer values are denominated in a smaller unit of FLOW called Atto-FLOW. This is done to conform with the EVM-ecosystem conventions and is similar to how Wei ("atto-ETH") is used to store ETH values on Ethereum. (1 Flow = 10^18 Atto-FLOW).

"Flow EVM" has its own dedicated storage state, which is separate from the Cadence environment. This storage state is authenticated and verified by the Flow network. The separation of storage is necessary to prevent direct memory access and maintain advanced safety properties of Cadence. The genesis state of the "Flow EVM" is empty, thus there are no accounts with non-zero balances.


However, a high degree of composability between Flow EVM and Cadence environments has been facilitated through several means.

1. **Cadence calls to the Flow EVM**: Cadence can send and receive data by making calls into the Flow EVM environment. 
2. **“Flow EVM” extended precompiles:** a set of smart contracts available on Flow EVM that can be used by other EVM smart contracts to access and interact with the Cadence world. 
3. **Cadence-Owned-Account (COA):** COA is a natively supported EVM smart contract wallet type that allows a Cadence resource to own and control an EVM address. This native wallet provides the primitives needed to bridge or control assets across Flow EVM and Cadence. 

Let’s take a closer look at what each of these mean.

### Cadence direct calls to the Flow EVM:

"Flow EVM" can be accessed only through Flow transactions and scripts or from Cadence smart contracts. However, later in this document, we describe options that have been provided to facilitate interaction between platforms that are only EVM-friendly with "Flow EVM".


Within Cadence the “Flow EVM” can be imported using 

```cadence
import EVM from <ServiceAddress>
```

The "Flow EVM" smart contract defines structures and resources, and exposes functions for querying and interacting with the EVM state.

The very first type defined in this contract is EVMAddress. It is a Cadence type representing an EVM address and constructible with any sequence of 20 bytes EVM.EVMAddress(bytes: bytes). Note that in the EVM world, there is no concept of accounts or a minimum balance requirement. Any sequence of bytes with a length of 20 is considered a valid address.

- These structures allows query about the EVM addresses from the Cadence side
  - `balance() Balance` returns the balance of the address, returning a balance object instead of a basic type is considered to prevent flow to atto-flow conversion mistakes
  - `nonce() UInt64` returns the nonce of the address
  - `code(): [UInt8]` returns the code of the address (if is a smart contract account,  it returns empty otherwise)


```cadence
// Example of balance query
import EVM from <ServiceAddress>

access(all)
fun main(bytes: [UInt8; 20]) {
    let addr = EVM.EVMAddress(bytes: bytes)
    let bal = addr.balance()
}
```

`run` is the next crucial function in this contract which runs RLP-encoded EVM transactions. If the interaction with “Flow EVM” is successful and it changes the state, a new Flow-EVM block is formed and its data is emitted as a Cadence event. An interaction for the first release is considered a single function call but in the future updates we could batch functions and let a single Flow transaction do multiple interactions during a single Flow-EVM block. 

```cadence
// Example of tx wrapping
import EVM from <ServiceAddress>

transaction(rlpEncodedTransaction: [UInt8], coinbaseBytes: [UInt8; 20]) {

    prepare(signer: AuthAccount) {
        let coinbase = EVM.EVMAddress(bytes: coinbaseBytes)
        EVM.run(tx: rlpEncodedTransaction, coinbase: coinbase)
    }
}
```

However, if an interaction fails, the behavior differs for two categories of functions. The default behavior for regular functions is that if the function fails for any reason, it reverts the entire Flow transaction or script. This ensures atomic operations across two environments. For example, if an `EVM.run` fails during a Flow transaction, the entire transaction is reverted.

If the developer requires more control, an alternative version of most functions is provided with the same name but prefixed with `try`. Calling these methods **does not** revert the outer Flow transaction. Instead, it returns an error code (0 is reserved for no error) and allows the code to control the flow. One example of this is batching and executing multiple EVM transactions within a single Flow transaction.

The gas used during the method calls is aggregated, adjusted and added to the total computation fees of the Flow transaction and paid by the payer. The adjustment is done by multiplying the gas usage by a multiplier set by the network and adjustable by the service account.  

Please refer to the Appendix B for the full list of types and functions available in Flow EVM contract. 


### “Flow EVM” extended precompiles

Precompiles are smart contracts built into the EVM environment yet not implemented in Solidity (learn more about precompiles [here](https://www.evm.codes/precompiled)). Flow EVM environment extends the original set the standard precompiles with a few more precompile smart contracts. This extended set of precompile contracts are deployed at addresses prefixed `0x000000000000000000000001` leaving a huge space (`2^64`) for available for the future standard precompiles.


#### Cadence Arch 
Cadence Arch is the very first augmented precompiled smart contract. Cadence Arch is a multi-function smart contract (deployed at `0x0000000000000000000000010000000000000001`) allows any smart contract on Flow EVM to interact with the Cadence side.

Here is the list of some of the functions available on the Cadence Arch smart contract in the first release: 

- `FlowBlockHeight() uint64` (signature: 0x53e87d66) returns the current flow block height, this could be used instead of flow evm block heights to trigger scheduled actions given it's more predictable when a block might be formed. 

- `VerifyCOAOwnershipProof(bytes32 _hash, bytes memory _signature)(bool success)` returns true if proof is valid. An ownership proof verifies that a Flow wallet controls a COA account. This is done by checking signatures and their weights, loading the COA resource from the specified storage path and check the EVM address of the COA resource. More details available in the next section. 

Cadence arch can be updated over time with more functions, some could trigger actions on the Cadence side, but there would be follow up Flips for it. 


### Cadence-Owned-Account (COA)

Another native tool ensuring seamless composability across environments are Cadence Owned Accounts. *COA* is a natively supported smart contract wallet type on the Flow EVM, facilitating composability between Cadence and EVM environments; COA is a EVM smart contract wallet controlled by a Cadence resource:

- its an a smart contract deployed to Flow EVM, is accessible by other Flow EVM users and can accept and controls EVM assets such as ERC721s. However unlike other EVM smart contracts, they can initiate transactions (COA’s EVM address acts as `tx.origin`). This behaviour is different than other EVM environments that only EOA accounts can initiate a transaction. A new EVM transaction type (`TxType = 0xff`) is used to differentiate these transactions from other types of EVM transactions (e.g, DynamicFeeTxType (`0x02`). 

- its owned and controlled by a Cadence resource, which means they don’t need an EVM transaction to be triggered (e.g. a transaction to make a call to `execute` or EIP-4337’s `validateUserOpmethod`). Instead, using the Cadence interface on the resource, direct call transactions can be triggered. Calling this method emits direct call transaction events on the Flow EVM side.

Each COA smart contract can only be deployed through the Cadence side.
`EVM.createCadenceOwnedAccount(): @CadenceOwnedAccount` constructs and returns a Cadence resource, allocates a unique Flow EVM address (based on the UUID of the resource and with the prefix `0x000000000000000000000002`) and deploys the smart contract wallet byte codes to the given address. 

The Cadence resource associated with the smart contract like other Cadence resources can be stored in a default storage path in a Flow account or can be owned by a Cadence smart contract. 

As mentioned earlier COAs expose two interfaces for interaction, one on the Flow EVM side and one on the Cadence resource side. Lets take a closer look at each: 


#### COA’s Cadence resource interface

- `address(): EVMAddress` returns the address of the smart contract, and the EVM address that is returned could be used to query balance, code, nonce, etc.

- `deposit(from: @FlowToken.Vault)` allows depositing FLOW tokens into the smart contract (Cadence to Flow EVM)

- `withdraw(balance: Balance): @FlowToken.Vault` allows withdrawing balance from the Flow EVM address and bridges it back as a FlowToken Vault to be handled on the Cadence side. 

- `deploy(code: [UInt8], gasLimit: UInt64, value: Balance): EVMAddress` lets the COA smart contract deploy smart contracts, and the returned address is the address of the new smart contract. The value (balance) is taken from the COA smart contract and moved to the new smart contract address (if they accept it). 

- `call(to: EVMAddress, data: [UInt8], gasLimit: UInt64, value: Balance): [UInt8]` makes a call on behalf of the COA smart contract and the result of the call (the returned value) could be processed by the cadence side. The value (balance) is taken from the COA smart contract. Calling this method emits direct call transaction events. 


#### COA’s EVM smart contract interface

- `function supportsInterface(bytes4 interfaceId) external view returns (bool)` provides the functionality needed to satisfy EIP-165. If the given interfaceID is available it returns true. 

- `receive() external payable` is the standard method needed to allow other EVM users to send ETH to the smart contract directly, if data is passed to this method it would revert. This method is not usually directly called and any call to smart contract it without the data is automatically rerouted to this function.

- `function onERC721Received(address _operator, address _from, uint256 _tokenId, bytes calldata _data) external returns (bytes4)` is provided to support safe transfers from ERC721 asset contracts. It's called on the recipient after a transfer. Return of any value other than the magic value (`0x150b7a02`) must result in the transaction being reverted. 

- `function tokensReceived(address operator, address from, address to, uint256 amount, bytes calldata data, bytes calldata operatorData) external` is provided to support safe transfers from ERC777 asset contracts. is called by the ERC777 token contract after a successful transfer or a minting operation. It doesn’t return anything.

- `function onERC1155Received(address _operator, address _from, uint256 _id, uint256 _value, bytes calldata _data) external returns (bytes4)` is provided to support safe transfers from ERC1155 asset contracts.  It should return 0xf23a6e61 if supporting the receiving of a single ERC1155 token type. 

- `function onERC1155BatchReceived(address _operator, address _from, uint256[] calldata _ids, uint256[] calldata _values, bytes calldata _data) external returns (bytes4)` is provided to support safe transfers from ERC1155 asset contracts (batch of assets). It returns `0xbc197c81` if supporting the receiving of a batch of ERC1155 token types.

- `function isValidSignature(bytes32 _hash, bytes memory _signature) external view virtual returns (bytes4)` returns the bytes4 magic value `0x1626ba7e` when the signature is valid. This method is usually used to verify a personal sign for the smart contract wallets (see EIP-1271 for more details). In the context of COA smart contracts, we consider the signature as an aggregation of Flow account address, key index, path to COA resource and a set of signatures (Flow account). we return true if the signatures are valid, it provides enough weight for the account, and an account holds the resource at the given path. Under the hood, this method uses Cadence Arch contract for verification.


## Appendix A - Embracing the EVM ecosystem.

In this FLIP, we described the foundation of the Flow EVM, yet there are other works built on top of this foundation to ensure an effortless onboarding experience for the existing EVM ecosystem product and tools. 

**Flow EVM Gateway Software**

A separate software package is provided by the Flow Foundation that can be run by anyone (including the Flow Foundation), that follows the Flow EVM chain, sends queries to the Flow access nodes, and provides the JSON-RPC endpoints to any 3rd party applications that want to interact directly with the Flow EVM. With the exception of account state proof endpoints, all other endpoints of the JSON-RPC will be provided at the release. 

Any 3rd party running this software can accept EVM RLP encoded transactions from users and wrap them in a Flow transaction.  While they will pay for the fees, `EVM.run` provides a utility parameter (`coinbase`) to collect the gas fee from the user on the Flow EVM side while running the EVM transaction. The use of the `coinbase` address in this context indicates the EVM address which will receive the `gas usage * gas price` (set in transaction). Essentially, the transaction wrapper behaves similarly to a miner, receives the gas usage fees on an EVM address and pays for the transaction fees. The `gas price per unit of gas` creates a marketplace for these 3rd parties to compete over transactions. 

For example, a user might use MetaMask to sign a transaction for Flow EVM and broadcast it to services that check the gas fee on the transaction and wrap the transaction to be executed. Note that account nonce would protect against double execution of a transaction, similar to how other non-virtual blockchains prevent the miner from including a transaction multiple times.

As EVM interactions are encapsulated within Flow transactions, they leverage the security measures provided by Flow. These transactions undergo the same process of collection, execution, and verification as other Flow transactions, without any EVM intervention. Consequently, there is no requirement for intricate block formation logic (such as handling forks and reorganizations), mempools, or additional safeguards against malicious MEV (Miner Extractable Value) behaviours. More information about Flow's consensus model is available [here](https://flow.com/core-protocol).

You can read more about this work in the [Flow EVM Gateway](https://github.com/onflow/flips/pull/235) improvement proposal. 

**Flow VM Bridge (Cadence <> Flow EVM)** 
COAs provide out of the box $FLOW token bridging between environments. They are also powerful resources which integrate native cross-VM bridging capabilities through which applications can bridge arbitrary fungible and/or non-fungible tokens between Cadence and Flow EVM. Checkout the [Flow VM Bridge ](https://github.com/onflow/flips/pull/233) improvement proposal for more details. 


## Appendix B - “Flow EVM”’s smart contract (in Cadence)

```cadence
import "FlowToken"

access(all)
contract EVM {

    /// EVMAddress is an EVM-compatible address
    access(all)
    struct EVMAddress {

        /// Bytes of the address
        access(all)
        let bytes: [UInt8; 20]

        /// Constructs a new EVM address from the given byte representation
        init(bytes: [UInt8; 20]) {
            self.bytes = bytes
        }

        /// Balance of the address
        access(all)
        fun balance(): Balance {
            let balance = InternalEVM.balance(
                address: self.bytes
            )
            return Balance(flow: balance)
        }

        /// Code of the address
        access(all)
        fun code(): [UInt8] {
            let code = InternalEVM.code(
                address: self.bytes
            )
            return code
        }

        /// Nonce of the address
        access(all)
        fun nonce(): UInt64 {
            let nonce = InternalEVM.nonce(
                address: self.bytes
            )
            return nonce
        }
    }

    access(all)
    struct Balance {

        /// The balance in FLOW
        access(all)
        let flow: UFix64

        /// Constructs a new balance, given the balance in FLOW
        init(flow: UFix64) {
            self.flow = flow
        }

        /// Returns the balance in terms of atto-FLOW.
        /// Atto-FLOW is the smallest denomination of FLOW inside EVM
        access(all)
        fun toAttoFlow(): UInt64
    }

    /// Runs an RLP-encoded EVM transaction, deducts the gas fees,
    /// and deposits the gas fees into the provided coinbase address.
    ///
    /// if the transaction execution fails,
    /// it reverts the outer Flow transaction
    access(all)
    fun run(tx: [UInt8], coinbase: EVMAddress) {
        InternalEVM.run(tx: tx, coinbase: coinbase.bytes)
    }

    /// TryRun tries to run an EVM transaction similar to the  
    /// way run works, except if it fails it returns 
    /// an error code. Note that some failures such as going
    /// running out of computation still reverts outer transaction.  
    /// if successful error code zero is returnd.
    access(all)
      fun tryRun(tx: [UInt8], coinbase: EVMAddress): UInt8{
        InternalEVM.run(tx: tx, coinbase: coinbase.bytes)
    }

        /// EncodeABI abi encodes given values 
    access(all)
    fun encodeABI(_ values: [AnyStruct]): [UInt8] {
        return InternalEVM.encodeABI(values)
    }

        /// DecodeABI decodes an ABI encoded data based on 
    /// the given Cadence types 
    access(all)
    fun decodeABI(types: [Type], data: [UInt8]): [AnyStruct] {
        return InternalEVM.decodeABI(types: types, data: data)
    }

    /// Creates a new Cadence Owned Account (COA)
    access(all)
    fun createCadenceOwnedAccount(): @CadenceOwnedAccount {
        return <-create CadenceOwnedAccount(
            addressBytes: InternalEVM.createCadenceOwnedAccount()
        )
    }

    access(all)
    resource CadenceOwnedAccount {

        access(self)
        let addressBytes: [UInt8; 20]

        init(addressBytes: [UInt8; 20]) {
           self.addressBytes = addressBytes
        }

        /// The EVM address of the account
        access(all)
        fun address(): EVMAddress {
            // Always create a new EVMAddress instance
            return EVMAddress(bytes: self.addressBytes)
        }

        /// Get the balance of the account
        access(all)
        fun balance(): Balance {
            return self.address().balance()
        }

                /// Deposits the given vault into the account's balance
        access(all)
        fun deposit(from: @FlowToken.Vault) {
            InternalEVM.deposit(
                from: <-from,
                to: self.bytes
            )
        }

        /// Withdraws the balance from the account's balance
        access(all)
        fun withdraw(balance: Balance): @FlowToken.Vault {
            let vault <- InternalEVM.withdraw(
                from: self.addressBytes,
                amount: balance.flow
            ) as! @FlowToken.Vault
            return <-vault
        }

        /// Deploys a contract to the EVM environment.
        /// Returns the address of the newly deployed contract
        access(all)
        fun deploy(
            code: [UInt8],
            gasLimit: UInt64,
            value: Balance
        ): EVMAddress {
            let addressBytes = InternalEVM.deploy(
                from: self.addressBytes,
                code: code,
                gasLimit: gasLimit,
                value: value.flow
            )
            return EVMAddress(bytes: addressBytes)
        }

        /// Calls a function with the given data.
        /// The execution is limited by the given amount of gas
        access(all)
        fun call(
            to: EVMAddress,
            data: [UInt8],
            gasLimit: UInt64,
            value: Balance
        ): [UInt8] {
             return InternalEVM.call(
                 from: self.addressBytes,
                 to: to.bytes,
                 data: data,
                 gasLimit: gasLimit,
                 value: value.flow
            )
        }
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

    event SafeReceived(address indexed sender, uint256 value);

    receive() external payable  { 
        emit SafeReceived(msg.sender, msg.value);
    }

    function supportsInterface(bytes4 id) external view virtual override returns (bool) {
        return
            id == type(ERC1155TokenReceiver).interfaceId ||
            id == type(ERC721TokenReceiver).interfaceId ||
            id == type(ERC777TokensRecipient).interfaceId ||
            id == type(IERC165).interfaceId;
    }

    function onERC1155Received(address, address, uint256, uint256, bytes calldata) external pure override returns (bytes4) {
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
        (bool ok, bytes memory out) = cadenceArch.staticcall(abi.encode(_hash, _sig));
        if (ok) {
            return 0x1626ba7e;
        }
        return 0xffffffff;
      }

}
```


