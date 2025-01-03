---
status: draft 
flip: 316 (set to the issue number)
authors: Jordan Ribbink (jordan.ribbink@flowfoundation.org), Chase Fleming (chase.fleming@flowfoundation.org)
sponsor: Jordan Ribbink (jordan.ribbink@flowfoundation.org)
updated: 2024-12-23
---

# FCL Ethereum Provider for Cross-VM Apps

## Objective

This FLIP proposes the integration of an Ethereum provider interface into [FCL-JS](https://www.npmjs.com/package/@onflow/fcl) to interact with a user’s Cadence Owned Account (COA) and behave as a middleware between the FCL protocol and EVM tooling.

The primary objective of this integration is to establish a mutually authenticated interface that unifies session management and interactions for a wallet’s Cadence and EVM functionalities.  An ancillary benefit is that such an interface also enables the use of Flow-native wallets in Flow EVM applications.

This will help facilitate progressive adoption of Cadence, where developers can incrementally integrate FCL interactions into their existing EVM applications.

## Motivation

Flow’s Cadence VM enjoys a unique level of visibility and control over the Flow EVM state, where it is able to atomically interact with this sub-VM within a single composable environment.  This architecture positions Cadence as a **powerful extensibility layer** for EVM applications, allowing developers to enhance their dApps by leveraging the unique advantages of Cadence.

Despite this potential, there are significant **integration challenges** which hinder adoption. Notably, Cadence-aware wallets do not share the same wallet adapter interface as their EVM counterparts.

This fragmentation inhibits interoperability between client tooling for each VM.  Once a Flow EVM application has connected to a Cadence-aware wallet’s Ethereum Provider, there is no mechanism to identify its Cadence capabilities or interact with this wallet’s authenticated session using FCL.  As a result, Flow EVM developers unable to explore a Cadence integration without significant upfront investment in tooling and workflow adaptations.

By addressing these barriers, this FLIP aims to unlock the ability for developers to **incrementally integrate Flow’s capabilities** into their existing Ethereum applications.  This approach provides **progressive adoption benefits**, reducing the risk of deploying on Flow by allowing developers to integrate features over time, rather than requiring a wholesale commitment upfront.

## User Benefit

This proposal lowers the barriers for developers building on Flow EVM to explore and integrate Cadence by bridging incompatible tooling and workflows. By enabling an incremental adoption path, developers can progressively experiment with Flow’s unique capabilities without the need for significant upfront investment or complete commitment to Flow tooling.

The integration also enhances user experience by enabling Flow EVM applications to seamlessly support FCL-compatible wallets. This alignment with user preferences fosters a more inclusive and interoperable ecosystem, encouraging broader adoption and innovation within the Flow and Ethereum communities.

### **Example Usage**

Application developers will typically not interact directly with the FCL Ethereum provider.  Instead, it will typically be transparently configured by developers as follows (e.g. Rainbowkit):

```tsx
import { getDefaultConfig } from "@onflow/cross-vm-rainbowkit"

const config = getDefaultConfig({
  appName: 'My RainbowKit App',
  projectId: 'YOUR_PROJECT_ID',
});
```

In this example, once the user has authenticated with Rainbowkit *with a Cadence-aware wallet*, FCL interactions can be used interchangeably with Wagmi.

If the user's wallet does not support Cadence interactions (e.g. MetaMask), only Wagmi interactions will be available.

```tsx
import * as fcl from "@onflow/fcl";
import { useSendTransaction } from "wagmi"

function MyComponent() {
  const sendTransaction = useSendTransaction()

  const sendEVMTransaction = async () => {
    await sendTransaction({
      to: "0x1234...",
      data: "0x1234...",
      value: "0x1234...",
    })
  }

  const sendCadenceTransaction = async () => {
    await fcl.mutate({ cadence: ... })
  }
}
```

## Design Proposal

### Overview

FCL will expose an [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193) Ethereum provider which can be used to interact with a Cadence-aware wallet’s COA via the Ethereum JSON-RPC API.  This provider will behave as a middleware between EVM tooling and FCL-JS by translating Ethereum JSON-RPC requests to interactions with an FCL wallet provider.

It will be assumed that the user’s COA is stored at the path `/storage/coa`.

The FCL Ethereum provider will be compatible with all FCL-compatible wallets.  Some Cadence-aware wallets already include an integrated Ethereum JSON-RPC API (e.g. Flow Wallet).  However, for wallets which do not include this API, the FCL Ethereum provider can behave as a compatibility adapter to interact with Flow EVM tooling & applications.

### API Design

A new method will be added to the FCL `currentUser` API, allowing developers to access FCL’s integrated Ethereum provider.

```tsx
function createEthereumProvider(config: {
	service?: Service,
	gateway?: Eip1193Provider
}): Eip1193Provider
```

- `service` is an optional FCL service to be used to perform the authentication handshake and will default to the FCL Discovery service.  This can be used to target individual FCL services instead of FCL Discovery.
- `gateway` is the JSON-RPC provider corresponding to the Flow EVM gateway, will default to the public Flow EVM gateway.

The FCL Ethereum provider can be interacted with as follows:

```tsx
import * as fcl from "@onflow/fcl";

// Get COA Ethereum provider
const provider = fcl.currentUser().createEthereumProvider()

// Will authenticate both FCL currentUser & COA Ethereum provider
const evmAddress = await provider.request({ method: "eth_requestAccounts" })[0]

// Single session shared between Ethereum JSON-RPC & FCL
console.log("EVM Address:", evmAddress)
console.log("Cadence Address:", (await fcl.currentUser().snapshot()).addr)

// Send EVM TX from COA
await signer.sendTransaction(...)

// Send FCL Transaction from Cadence account
fcl.mutate({ cadence: ... })
```

In practice, application developers will typically interact directly with the provider, but instead through higher-level tooling where it exists behind layers of abstraction.

### Ethereum JSON-RPC Methods

The FCL Ethereum provider will handle requests relating to wallet interactions internally. Any requests that cannot be processed by the client will be forwarded through the provider to the configured EVM gateway.

The provider will handle the following types of requests:

- **`eth_accounts` / `eth_requestAccounts`**
    
    Respond with the Cadence-Owned Account (COA) corresponding to the account currently authenticated with FCL. FCL-JS will assume that a user's COA exists at the standard `/storage/evm` path and will attempt to create a COA if one does not exist.
    
    If the user has not authenticated with FCL, this method will invoke the FCL authentication handshake and return COA address for the resulting user.
    
- **`eth_sendTransaction`**
    
    The emulated provider will wrap the transaction payload with a Cadence transaction to invoke the `call` method from the corresponding COA resource.
    
    The provider should use the following script to execute transactions from an account’s COA.  The deterministic message format can be used to help wallets distinguish EVM transactions from COAs.
    
    Replace `<<CONTRACT_ADDRESS>>` with the appropriate contract address before using the script.  
    
    - Call COA Transaction
        
        ```tsx
        import EVM from <<CONTRACT_ADDRESS>>
        
        /// Executes the calldata from the signer's COA
        ///
        transaction(evmContractAddressHex: String, calldata: String, gasLimit: UInt64, value: UFix64) {
        
            let evmAddress: EVM.EVMAddress
            let coa: auth(EVM.Call) &EVM.CadenceOwnedAccount
        
            prepare(signer: auth(BorrowValue) &Account) {
                self.evmAddress = EVM.addressFromString(evmContractAddressHex)
        
                self.coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(from: /storage/evm)
                    ?? panic("Could not borrow COA from provided gateway address")
            }
        
            execute {
                let valueBalance = EVM.Balance(attoflow: 0)
                valueBalance.setFLOW(flow: value)
                let callResult = self.coa.call(
                    to: self.evmAddress,
                    data: calldata.decodeHex(),
                    gasLimit: gasLimit,
                    value: valueBalance
                )
                assert(callResult.status == EVM.Status.successful, message: "Call failed")
            }
        }
        ```
        
    
    One the provider has sent the transaction, it will respond to the request with the EVM transaction hash.
    
- **`eth_chainId`**
    
    The provider should respond with the Flow EVM chain ID.
    
- **`eth_personalSign`**
    
    The emulated provider will request a request a user signature from the user’s FCL wallet.  The hex-encoded payload remains unchanged.
    
    The provider should transform the FCL response into an RLP-encoded [COAOwnershipProof](https://github.com/onflow/flow-go/blob/master/fvm/evm/types/proof.go#L139) and return this to the caller.
    
- **`eth_signTypedData_v4`**
    
    The emulated provider will request a request a user signature from the user’s FCL wallet.  The payload to be signed is the hex-encoded typed data hash.
    
    The provider should transform the FCL response into an RLP-encoded [COAOwnershipProof](https://github.com/onflow/flow-go/blob/master/fvm/evm/types/proof.go#L139) and return this to the caller.
    

Any errors encountered when handling these requests will include error codes aligned with [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193#provider-errors).

### Ethereum Provider Events

All of the following events defined in [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193#provider-errors) will be implemented on the FCL Ethereum Provider.

- `connect` should be emitted when FCL user has authenticated
- `disconnect` should be emitted when FCL user has unauthenticated
- `accountsChanged` should be emitted when FCL user has authenticated, corresponding to the user’s COA

### Drawbacks

**Risk of Ethereum JSON-RPC Changes**

This proposal assumes that all necessary Ethereum JSON-RPC methods can be expressed using the FCL Protocol.  If future additions to the API violate this assumption, re-establishing equivalence will require changes to the FCL protocol.

**Risk of Evolving COA Standards**

This proposal assumes that an FCL user’s COA will always be stored at the canonical `/storage/evm` path.  Future evolution of ecosystem standards for COA storage may necessitate additional changes to accommodate alternate schemes.

### Alternatives Considered

**Extending Ethereum Provider JSON-RPC API**

It was considered to adopt a set of [CASA standards](https://github.com/ChainAgnostic) for multi-VM authentication using an Ethereum provider’s existing RPC transport was considered.

This benefits from a looser coupling between Cadence and EVM wallet interactions and allows dApps to express more granular requests to a wallet.  However, this was outweighed by the drawbacks of unnecessary complexity and inherent restrictions of the EVM transport layer.

### Performance Implications

There are no significant performance implications associated with this proposal.  Latency will be similar to existing Etheruem JSON-RPC APIs for COAs (e.g. Flow Wallet).

### Dependencies

Many EVM client libraries will require additional plugins or adapters to interact with the FCL Ethereum provider.

### Engineering Impact

This change should take a medium amount of effort to implement the FCL Ethereum provider, necessary plugins for EVM tooling, and examples.

### Best Practices

New features will be documented in the FCL API reference in the [Flow Developer Docs](https://developers.flow.com/).

Flow EVM applications are encouraged to configure FCL Ethereum providers to support Cadence-aware wallets, even if they do not plan to integrate with Cadence.

### Tutorials and Examples

Guides for adding Cadence functionality to an existing EVM application should be created with specific integration paths for popular EVM client libraries (i.e. Wagmi, Rainbowkit).

An example application should be created to excite EVM developers about the benefits of a Cadence integration and drive adoption efforts.

### Compatibility

The FCL Ethereum provider is an additive feature with no impact on existing FCL functionality.  It implements the [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193) provider interface & full Ethereum JSON-RPC API to guarantee compatibility with existing EVM tooling.

### User Impact

This feature will be introduced in an upcoming release of the `FCL-JS` library.

**EVM Developers** must configure relevant plugins or adapters with their existing tooling when integrating Cadence-aware wallets.

**End-Users** will be unaffected and connect to FCL Ethereum providers in a manner indistinguishable from their usual workflows.

### Related Issues

N/A

### Prior Art

**Flow Wallet**: Implements the Ethereum JSON-RPC API to interact with COAs similar to this proposal.

**Flow EVM Gateway:** Implements the Ethereum JSON-RPC to interact with the Flow EVM state, however does not interact with COAs.

### Questions and Discussion

This FLIP should be discussed in the [FLIPs](https://github.com/onflow/flips) GitHub repository as part of its PR.

Some open questions include:

- Are there additional risks associated with implementing a client-side Etheruem JSON-RPC handler that have not been considered in this document?
- How should wallet provider de-duplication be handled when both an [EIP-6963](https://eips.ethereum.org/EIPS/eip-6963) and FCL Ethereum provider exist for the same wallet? (likely just add RDNS to FCL Service)