---
status: draft
flip: 338
title: DeFiActions: Composable DeFi Standards for Flow
authors: Giovanni Sanchez (giovanni.sanchez@flowfoundation.org)
sponsor: Kan Zhang (kan.zhang@flowfoundation.org)
updated: 2025-07-30
---

# FLIP 338: DeFiActions - Composable DeFi Standards for Flow

> Standardized interfaces enabling composition of common DeFi operations through pluggable, reusable components

<details>

<summary>Table of contents</summary>

- [FLIP 338: DeFiActions - Composable DeFi Standards for Flow](#flip-338-defiactions---composable-defi-standards-for-flow)
  - [Objective](#objective)
  - [Motivation](#motivation)
  - [User Benefit](#user-benefit)
  - [Design Proposal](#design-proposal)
    - [Core Philosophy](#core-philosophy)
    - [Component Model Overview](#component-model-overview)
    - [Interfaces](#interfaces)
      - [Source Interface](#source-interface)
      - [Sink Interface](#sink-interface)
      - [Swapper Interface](#swapper-interface)
      - [PriceOracle Interface](#priceoracle-interface)
      - [Flasher Interface](#flasher-interface)
    - [Component Composition](#component-composition)
    - [Identification \& Traceability](#identification--traceability)
    - [Stack Introspection](#stack-introspection)
      - [Simple Stack Introspection Example](#simple-stack-introspection-example)
      - [Complex Stack Introspection Example](#complex-stack-introspection-example)
  - [Implementation Details](#implementation-details)
    - [Core Interfaces](#core-interfaces)
    - [Connector Examples](#connector-examples)
      - [FungibleToken Connectors](#fungibletoken-connectors)
      - [Swap Connectors](#swap-connectors)
      - [DEX Connectors](#dex-connectors)
      - [Flash Loan Connectors](#flash-loan-connectors)
    - [AutoBalancer Component](#autobalancer-component)
    - [Event System](#event-system)
  - [Use Cases](#use-cases)
    - [Automated Token Transmission](#automated-token-transmission)
  - [Considerations](#considerations)
    - [Security Model](#security-model)
    - [Performance Implications](#performance-implications)
    - [Testing Challenges](#testing-challenges)
    - [Drawbacks](#drawbacks)
    - [Interface Concerns](#interface-concerns)
  - [Compatibility](#compatibility)
  - [Future Extensions](#future-extensions)

</details>

## Objective

This proposal introduces [DeFiActions (DFA)](https://github.com/onflow/DeFiActions), a suite of standardized Cadence
interfaces that enable developers to compose complex DeFi workflows by connecting small, reusable components. DFA
provides a "money LEGO" framework where each component performs a single DeFi operation (deposit, withdraw, swap, price
lookup, flash loan) while maintaining composability with other components to create sophisticated financial strategies
executable in a single atomic transaction.

## Motivation

Flow's DeFi ecosystem currently lacks standardized interfaces for connecting protocols and creating complex workflows.
Developers building applications that interact with multiple DeFi protocols face several challenges:

1. **Protocol Fragmentation**: Each DeFi protocol implements unique interfaces, requiring custom integration code and
   deep protocol-specific knowledge
2. **Workflow Complexity**: Building multi-step DeFi strategies (like leverage, yield farming, or automated rebalancing)
   requires managing multiple protocol calls
3. **Limited Composability**: Without shared interfaces, protocols cannot easily integrate with each other, limiting
   composability and increasing the barrier to entry for new developers
4. **Development Overhead**: Each application must implement protocol-specific logic, leading to duplicated effort and
   increased maintenance burden

DeFiActions addresses these challenges by providing a unified abstraction layer that makes DeFi protocols interoperable
while maintaining the security and flexibility developers expect.

## User Benefit

DeFiActions provides significant benefits to different stakeholders in the Flow ecosystem:

**For Application Developers:**
- **Simplified Integration**: Connect to any DFA-compatible protocol through standardized interfaces
- **Rapid Prototyping**: Build complex DeFi workflows by composing pre-built components
- **Reduced Maintenance**: Protocol updates are abstracted away by connector implementations, enabling more modular
  dependency architectures
- **Enhanced Functionality**: Create sophisticated strategies that would be complex to implement from scratch

**For Protocol Developers:**
- **Increased Adoption**: Protocols become instantly compatible with any DFA-built application by simply creating DFA
  connectors adapted to their protocol
- **Network Effects**: Benefit from integration work done by other protocols in the ecosystem and tapping into a
  community of DFA-focussed developers

**For End Users:**
- **Advanced Strategies**: Access to sophisticated DeFi workflows through simple interfaces
- **Atomic Execution**: Complex multi-protocol operations execute in single transactions
- **Autonomous Operations**: Integration with scheduled callbacks enables self-executing strategies, enabling trustless
  active management

## Design Proposal

### Core Philosophy

DeFiActions is inspired by Unix terminal piping, where simple command outputs can be connected together to create
complex operations in aggregate. Analagously, each DFA component should exhibit:

- **Single Responsibility**: Each component performs one specific DeFi operation
- **Composable**: Shared standards in an open environment allow developers to reuse and remix actions built by others
- **Standardized**: All components of the same type implement identical interfaces
- **Graceful Failure**: Components handle edge cases gracefully rather than reverting where possible

### Component Model Overview

DFA defines five core component types, each representing a fundamental DeFi operation:

1. **Source**: Provides tokens on demand (e.g. withdraw from vault, claim rewards, pull liquidity)
2. **Sink**: Accepts tokens up to capacity (e.g. deposit to vault, repay loan, add liquidity)
3. **Swapper**: Exchanges one token type for another (e.g. targetted DEX trades, multi-protocol aggregated swaps)
4. **PriceOracle**: Provides price data for assets (e.g. external price feeds, DEX prices, price caching)
5. **Flasher**: Provides flash loans with atomic repayment (e.g. arbitrage, liquidations)

Additional specialized components build upon and/or support these primitives:

1. **AutoBalancer**: Automated rebalancing system that uses Sources, Sinks, and PriceOracles to maintain a token balance
   around the value of historical deposits, directing excess value to a Sink and topping up deficient value from a
   Source if configured.
2. **Quote**: Data structure for swap price estimates and execution parameters, allowing Swapper consumers to cache swap
   quotes in either direction.
3. **UniqueIdentifier**: Identifies components as related to the same composition or "stack" in both the Cadence runtime
   and DFA interface events. If two components share the same `UniqueIdentifier`, they be assumed to be a part of the
   same workflow stack.
4. **IdentifiableStruct/IdentifiableResource**: Interfaces inherited by all other DFA components allowing for them to be
   identified by their corresponding `UniqueIdentifier`, for stacks to be extended with newly identified components, and
   to enable stack introspection allowing for the querying of included components.
5.  **ComponentInfo**: Serves basic information about a component and its inner components, allowing for introspection
    across a stack of components.

### Interfaces

> :information_source: Initial DeFi Actions implementations can be found in the [onflow/DeFiActions
> repo](https://github.com/onflow/DeFiActions)

#### Source Interface

Similar to `FungibleToken.Provider`, a `Source` provides tokens while gracefully handling scenarios where the requested
amount may not be fully available:

```cadence
access(all) struct interface Source : Identifiable {
    /// Returns the Vault type provided by this Source
    access(all) view fun getSourceType(): Type
    /// Returns an estimate of how much can be withdrawn
    access(all) fun minimumAvailable(): UFix64
    /// Withdraws up to maxAmount, returning what's actually available
    access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault}
}
```

Key design principles:
- **Graceful Degradation**: Returns available amount rather than reverting when the full amount unavailable
- **Predictable Interface**: Always returns a Vault, even if empty
- **Estimation**: Provides an estimate of the available amount as well as the return type

#### Sink Interface

Similar to `FungibleToken.Receiver`, a Sink accepts tokens up to its capacity:

```cadence
access(all) struct interface Sink : Identifiable {
    /// Returns the Vault type accepted by this Sink
    access(all) view fun getSinkType(): Type
    /// Returns an estimate of remaining capacity
    access(all) fun minimumCapacity(): UFix64
    /// Deposits up to capacity, leaving remainder in the referenced vault
    access(all) fun depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
}
```

Key design principles:
- **Capacity Management**: Only accepts tokens up to its current capacity which may include the full balance of the
  referenced `Vault`
- **Non-destructive**: Excess tokens remain in the referenced `Vault`
- **Flexible Limits**: Capacity can be dynamic based on underlying recipient's capacity

#### Swapper Interface  

A Swapper exchanges tokens between different types with support for bidirectional swaps and price estimation:

```cadence
access(all) struct interface Swapper : Identifiable {
    /// Input and output token types - in and out token types via default `swap()` route
    access(all) view fun inType(): Type
    access(all) view fun outType(): Type
    
    /// Price estimation methods - quote required amount given some desired output & output for some provided input
    access(all) fun quoteIn(forDesired: UFix64, reverse: Bool): {Quote}
    access(all) fun quoteOut(forProvided: UFix64, reverse: Bool): {Quote}
    
    /// Swap execution methods
    access(all) fun swap(quote: {Quote}?, inVault: @{FungibleToken.Vault}): @{FungibleToken.Vault}
    access(all) fun swapBack(quote: {Quote}?, residual: @{FungibleToken.Vault}): @{FungibleToken.Vault}
}
```

Key design principles:
- **Bidirectional**: Supports swaps in both directions via `swapBack()`, a requisite feature in the event an inner
  connector can't accept the full swap output balance
- **Price Discovery**: Provides estimated amounts in and out before execution via `{Quote}` object
- **Quote System**: Enables price caching and execution parameter optimization

#### PriceOracle Interface

A PriceOracle provides price data for assets with a consistent denomination:

```cadence
access(all) struct interface PriceOracle : Identifiable {
    /// Returns the denomination asset (e.g., USDCf, FLOW)
    access(all) view fun unitOfAccount(): Type
    /// Returns current price or nil if unavailable, conditions for which are implementation-specific
    access(all) fun price(ofToken: Type): UFix64?
}
```

Key design principles:
- **Consistent Denomination**: All prices returned in the same unit of account
- **Graceful Unavailability**: Returns `nil` rather than reverting in the event a price is unavailable
- **Type-Based**: Prices indexed by Cadence Type, requiring a distinct Cadence-based token type for which to serve
  prices

#### Flasher Interface

A Flasher provides flash loans with atomic repayment requirements:

```cadence
access(all) struct interface Flasher : Identifiable {
    /// Returns the asset type this Flasher can issue as a flash loan
    access(all) view fun borrowType(): Type
    /// Returns the estimated fee for a flash loan of the specified amount
    access(all) fun calculateFee(loanAmount: UFix64): UFix64
    /// Performs a flash loan of the specified amount. The callback function is passed the fee amount, a loan Vault,
    /// and data. The callback function should return a Vault containing the loan + fee.
    access(all) fun flashLoan(
        amount: UFix64,
        data: AnyStruct?,
        callback: fun(UFix64, @{FungibleToken.Vault}, AnyStruct?): @{FungibleToken.Vault} // fee, loan, data
    )
}
```

Key design principles:
- **Atomic Repayment**: Loan must be repaid within the callback's function scope
- **Callback Pattern**: Consumer logic runs in provided function rather than separate components
- **Fee Transparency**: Implementations provide fee calculation before execution
- **Repayment Guarantee**: Implementers must validate full repayment (loan + fee) before transaction completion

**Design Rationale**: The callback function pattern was chosen over requiring separate Sink/Source components for
several reasons:
- **Lighter Weight**: No contract deployment required to define flash loan logic
- **Competitive Advantage**: Transaction-scoped logic maintains user edge over permanent on-chain code
- **Consolidated Context**: Single execution scope rather than split between multiple components
- **Flexibility**: Implementations may still leverage Sink/Source connectors in callback scope, but they're not required

### Component Composition

Components are designed to connect seamlessly where compatible:

```cadence
// Example: Source -> Swapper -> Sink pipeline
let tokens <- source.withdrawAvailable(maxAmount: 100.0)
let swappedTokens <- swapper.swap(quote: nil, inVault: <-tokens)  
sink.depositCapacity(from: &swappedTokens as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
```

Compositions connectors include:
- **SwapSink**: Combines Swapper + Sink for automatic token conversion before deposit (e.g. deposit to SwapSink as
  TokenA, swap to TokenB and deposit TokenB to inner Sink)
- **SwapSource**: Combines Source + Swapper for automatic token conversion after withdrawal (e.g. initiate withdrawal of
  TokenA, withdraw from inner Source as TokenB, swap to TokenA and return the swapped result)
- **MultiSwapper**: Aggregates multiple Swappers to find optimal pricing

### Identification & Traceability

The `UniqueIdentifier` enables protocols to trace stack operations via DeFiActions interface-level events, identifying
them by IDs. `IdentifiableResource` Implementations should ensure that access to the identifier is encapsulated by the
structures they identify.

While Cadence struct types can be created in any context (including being passed in as transaction parameters), the
authorized `AuthenticationToken` Capability ensures that only those issued by the DFA contract can be utilized in
connectors, preventing forgery.

```cadence
access(all) struct UniqueIdentifier {
    /// The ID value of this UniqueIdentifier
    access(all) let id: UInt64
    /// The AuthenticationToken Capability required to create this UniqueIdentifier. Since this is a struct which 
    /// can be created in any context, this authorized Capability ensures that the UniqueIdentifier can only be 
    /// created by the DeFiActions contract, thus preventing forged UniqueIdentifiers from being created.
    access(self) let authCap: Capability<auth(Identify) &AuthenticationToken>

    access(contract) view init(_ id: UInt64, _ authCap: Capability<auth(Identify) &AuthenticationToken>) {
        pre {
            authCap.check(): "Invalid AuthenticationToken Capability provided"
        }
        self.id = id
        self.authCap = authCap
    }
}
```

All DFA connectors implement the `IdentifiableStruct` interface, which includes an optional `UniqueIdentifier` resource
for operation tracing. Since the AutoBalancer is a resource type, an analagous interface `IdentifiableResource` is also
proposed (though omitted in this doc) for use in existing and future resource typed DFA components.

```cadence
access(all) struct interface IdentifiableStruct {
    /// An optional identifier allowing protocols to identify stacked connector operations by defining a protocol-
    /// specific Identifier to associated connectors on construction
    access(contract) var uniqueID: UniqueIdentifier?
    /// Convenience method returning the inner UniqueIdentifier's id or `nil` if none is set.
    ///
    /// NOTE: This interface method may be spoofed if the function is overridden, so callers should not rely on it
    /// for critical identification unless the implementation is known and trusted
    access(all) view fun id(): UInt64? {
        return self.uniqueID?.id
    }
    /// Returns a ComponentInfo struct containing information about this component and a list of ComponentInfo for
    /// each inner component in the stack.
    access(all) fun getComponentInfo(): ComponentInfo
    /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
    /// a DeFiActions stack. See DeFiActions.align() for more information.
    access(contract) view fun copyID(): UniqueIdentifier? {
        post {
            result?.id == self.uniqueID?.id:
            "UniqueIdentifier of \(self.getType().identifier) was not successfully copied"
        }
    }
    /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
    /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
    access(contract) fun setID(_ id: UniqueIdentifier?) {
        post {
            self.uniqueID?.id == id?.id:
            "UniqueIdentifier of \(self.getType().identifier) was not successfully set"
            DeFiActions.emitUpdatedID(
                oldID: before(self.uniqueID?.id),
                newID: self.uniqueID?.id,
                component: self.getType().identifier,
                uuid: nil // no UUID for structs
            ): "Unknown error emitting DeFiActions.UpdatedID from IdentifiableStruct \(self.getType().identifier) with ID "
                .concat(self.id()?.toString() ?? "UNASSIGNED")
        }
    }
}
```

This enables:
- **Event Correlation**: All component operations emit events tagged with the same ID
- **Stack Tracing**: Understanding the complete nested component chain
- **Analytics**: Tracking complex workflow performance and usage patterns via event analysis


Readers may notice that the `copyID()` and `setID()` methods are `access(contract)`. This is to restrict access to the
`UniqueIdentifier` values while still allowing for the extension of workflow stacks. A contract method `alignIDs()` is
proposed to align IDs between two authorized component references. Due to the number of permutations between struct &
resource types, the method is proposed with `AnyStruct` parameters with casting logic that prevents setting if values
fail to cast properly.

```cadence
/// Aligns the UniqueIdentifier of the provided component with the provided component, setting the UniqueIdentifier of
/// the provided component to the UniqueIdentifier of the provided component. Parameters are AnyStruct to allow for
/// alignment of both IdentifiableStruct and IdentifiableResource. However, note that the provided component must
/// be an auth(Extend) &{IdentifiableStruct} or auth(Extend) &{IdentifiableResource} to be aligned.
access(all) fun alignID(toUpdate: AnyStruct, with: AnyStruct) {
    let maybeISToUpdate = toUpdate as? auth(Extend) &{IdentifiableStruct}
    let maybeIRToUpdate = toUpdate as? auth(Extend) &{IdentifiableResource}
    let maybeISWith = with as? auth(Identify) &{IdentifiableStruct}
    let maybeIRWith = with as? auth(Identify) &{IdentifiableResource}

    if maybeISToUpdate != nil && maybeISWith != nil {
        maybeISToUpdate!.setID(maybeISWith!.copyID())
    } else if maybeISToUpdate != nil && maybeIRWith != nil {
        maybeISToUpdate!.setID(maybeIRWith!.copyID())
    } else if maybeIRToUpdate != nil && maybeISWith != nil {
        maybeIRToUpdate!.setID(maybeISWith!.copyID())
    } else if maybeIRToUpdate != nil && maybeIRWith != nil {
        maybeIRToUpdate!.setID(maybeIRWith!.copyID())
    }
    return
}
```

While this logic could be embedded in the interfaces as default methods, implementations overriding the default methods
would also override the post-condition block emitting the interface event. Instead, the interface as proposed
prioritizes the guaranteed emission of `UpdatedID` events on `setID()` execution over the convenience of methods.


### Stack Introspection

Components can be inspected to understand their composition via `Identifiable.getComponentInfo(): ComponentInfo`:

```cadence
access(all) struct ComponentInfo {
    /// The type of the component
    access(all) let type: Type
    /// The UniqueIdentifier.id of the component
    access(all) let id: UInt64?
    /// The inner component types of the serving component
    access(all) let innerComponents: [ComponentInfo]
}
```

This allows:
- **Dynamic Workflow Analysis**: Understanding component relationships programmatically
- **Debugging Support**: Identifying which components are involved in complex operations

> :information_source: Due to the implementation-specific nature of DFA connectors, it's not possible to provide a
> default implementation at the interface level that would satisfy all connectors nor guarantee through pre-post
> conditions that `ComponentInfo.innerComponents` serves correct information. Similar to NFT metadata, it's therefore
> the responsibility of the developer to ensure the method is implemented correctly, requiring trust on the part of the
> consumer.

#### Simple Stack Introspection Example

The FungibleTokenStack VaultSink is a simple sink just direct deposited funds to a Vault via Capability, so it has not
inner commponents.

```cadence
access(all) struct VaultSink : DeFiActions.Sink {
    // ...
    /// Simply returns info about itself since there exist no inner components
    access(all) fun getComponentInfo(): DeFiActionsComponentInfo {
        return DeFiActions.ComponentInfo(
                type: self.getType(),
                id: self.id(),
                innerComponents: []
            )
    }
    // ...
}
```

#### Complex Stack Introspection Example

The AutoBalancer contains at minimum a PriceOracle, but can also optionally include a Sink (to direct excess value) and
Source (to top up deficient value). Introspection results should then include not only the AutoBalancer's
`ComponentInfo`, but also the `ComponentInfo` of each contained connector and any connectors those may also contain. The
hierarchy of each element can be inferred from the `innerComponents` value.

```cadence
access(all) resource AutoBalancer : IdentifiableResource, ... {
    // ...
    (all) fun getComponentInfo(): ComponentInfo {
            // get the inner components
            let oracle = self._borrowOracle()
            let inner: [ComponentInfo] = [oracle.getComponentInfo()]

            // get the info for the optional inner components if they exist
            let maybeSink = self._borrowSink()
            let maybeSource = self._borrowSource()
            if let sink = maybeSink {
                inner.append(sink.getComponentInfo())
            }
            if let source = maybeSource {
                inner.append(source.getComponentInfo())
            }

            // create the ComponentInfo for the AutoBalancer and insert it at the beginning of the list
            return ComponentInfo(
                type: self.getType(),
                id: self.id(),
                innerComponents: inner
            )
        }
    // ...
}
```

## Implementation Details

### Core Interfaces

The complete DeFiActions interface specification includes:

<details>
<summary>Full DeFiActions Code</summary>

```cadence
import "Burner"
import "ViewResolver"
import "FungibleToken"

import "DeFiActionsUtils"

/// !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
/// THIS CONTRACT IS IN BETA AND IS NOT FINALIZED - INTERFACES MAY CHANGE AND/OR PENDING CHANGES MAY REQUIRE REDEPLOYMENT
/// !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
///
/// [BETA] DeFiActions
///
/// DeFiActions is a library of small DeFi components that act as glue to connect typical DeFi primitives (dexes, lending
/// pools, farms) into individual aggregations.
///
/// The core component of DeFiActions is the “Connector”; a conduit between the more complex pieces of the DeFi puzzle.
/// Connectors aren't to do anything especially complex, but make it simple and straightforward to connect the
/// traditional DeFi pieces together into new, custom aggregations.
///
/// Connectors should be thought of analogously with the small text processing tools of Unix that are mostly meant to be
/// connected with pipe operations instead of being operated individually. All Connectors are either a “Source” or
/// “Sink”.
///
access(all) contract DeFiActions {

    /* --- FIELDS --- */

    /// The current ID assigned to UniqueIdentifiers as they are initialized
    /// It is incremented by 1 every time a UniqueIdentifier is created so each ID is only ever used once
    access(all) var currentID: UInt64
    /// The AuthenticationToken Capability required to create a UniqueIdentifier
    access(self) let authTokenCap: Capability<auth(Identify) &AuthenticationToken>
    /// The StoragePath for the AuthenticationToken resource
    access(self) let AuthTokenStoragePath: StoragePath

    /* --- INTERFACE-LEVEL EVENTS --- */

    /// Emitted when value is deposited to a Sink
    access(all) event Deposited(
        type: String,
        amount: UFix64,
        fromUUID: UInt64,
        uniqueID: UInt64?,
        sinkType: String
    )
    /// Emitted when value is withdrawn from a Source
    access(all) event Withdrawn(
        type: String,
        amount: UFix64,
        withdrawnUUID: UInt64,
        uniqueID: UInt64?,
        sourceType: String
    )
    /// Emitted when a Swapper executes a Swap
    access(all) event Swapped(
        inVault: String,
        outVault: String,
        inAmount: UFix64,
        outAmount: UFix64,
        inUUID: UInt64,
        outUUID: UInt64,
        uniqueID: UInt64?,
        swapperType: String
    )
    /// Emitted when a Flasher executes a flash loan
    access(all) event Flashed(
        requestedAmount: UFix64,
        borrowType: String,
        uniqueID: UInt64?,
        flasherType: String
    )
    /// Emitted when an IdentifiableResource's UniqueIdentifier is aligned with another DFA component
    access(all) event UpdatedID(
        oldID: UInt64?,
        newID: UInt64?,
        component: String,
        uuid: UInt64?
    )
    /// Emitted when an AutoBalancer is created
    access(all) event CreatedAutoBalancer(
        lowerThreshold: UFix64,
        upperThreshold: UFix64,
        vaultType: String,
        vaultUUID: UInt64,
        uuid: UInt64,
        uniqueID: UInt64?
    )
    /// Emitted when AutoBalancer.rebalance() is called
    access(all) event Rebalanced(
        amount: UFix64,
        value: UFix64,
        unitOfAccount: String,
        isSurplus: Bool,
        vaultType: String,
        vaultUUID: UInt64,
        balancerUUID: UInt64,
        address: Address?,
        uuid: UInt64,
        uniqueID: UInt64?
    )

    /* --- CONSTRUCTS --- */

    access(all) entitlement Identify

    /// AuthenticationToken
    ///
    /// A resource intended to ensure UniqueIdentifiers are only created by the DeFiActions contract
    (all) resource AuthenticationToken {}

    /// UniqueIdentifier
    ///
    /// This construct enables protocols to trace stack operations via DeFiActions interface-level events, identifying
    /// them by UniqueIdentifier IDs. IdentifiableResource Implementations should ensure that access to them is
    /// encapsulated by the structures they are used to identify.
    (all) struct UniqueIdentifier {
        /// The ID value of this UniqueIdentifier
        access(all) let id: UInt64
        /// The AuthenticationToken Capability required to create this UniqueIdentifier. Since this is a struct which
        /// can be created in any context, this authorized Capability ensures that the UniqueIdentifier can only be
        /// created by the DeFiActions contract, thus preventing forged UniqueIdentifiers from being created.
        access(self) let authCap: Capability<auth(Identify) &AuthenticationToken>

        access(contract) view init(_ id: UInt64, _ authCap: Capability<auth(Identify) &AuthenticationToken>) {
            pre {
                authCap.check(): "Invalid AuthenticationToken Capability provided"
            }
            self.id = id
            self.authCap = authCap
        }
    }

    /// ComponentInfo
    ///
    /// A struct containing minimal information about a DeFiActions component and its inner components
    (all) struct ComponentInfo {
        /// The type of the component
        access(all) let type: Type
        /// The UniqueIdentifier.id of the component
        access(all) let id: UInt64?
        /// The inner component types of the serving component
        access(all) let innerComponents: [ComponentInfo]
        init(
            type: Type,
            id: UInt64?,
            innerComponents: [ComponentInfo]
        ) {
            self.type = type
            self.id = id
            self.innerComponents = innerComponents
        }
    }

    /// Extend entitlement allowing for the authorized copying of UniqueIdentifiers from existing components
    access(all) entitlement Extend

    /// IdentifiableResource
    ///
    /// A resource interface containing a UniqueIdentifier and convenience getters about it
    (all) struct interface IdentifiableStruct {
        /// An optional identifier allowing protocols to identify stacked connector operations by defining a protocol-
        /// specific Identifier to associated connectors on construction
        access(contract) var uniqueID: UniqueIdentifier?
        /// Convenience method returning the inner UniqueIdentifier's id or `nil` if none is set.
        ///
        /// NOTE: This interface method may be spoofed if the function is overridden, so callers should not rely on it
        /// for critical identification unless the implementation itself is known and trusted
        access(all) view fun id(): UInt64? {
            return self.uniqueID?.id
        }
        /// Returns a ComponentInfo struct containing information about this component and a list of ComponentInfo for
        /// each inner component in the stack.
        access(all) fun getComponentInfo(): ComponentInfo
        /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
        /// a DeFiActions stack. See DeFiActions.align() for more information.
        access(contract) view fun copyID(): UniqueIdentifier? {
            post {
                result?.id == self.uniqueID?.id:
                "UniqueIdentifier of \(self.getType().identifier) was not successfully copied"
            }
        }
        /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
        /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
        access(contract) fun setID(_ id: UniqueIdentifier?) {
            post {
                self.uniqueID?.id == id?.id:
                "UniqueIdentifier of \(self.getType().identifier) was not successfully set"
                DeFiActions.emitUpdatedID(
                    oldID: before(self.uniqueID?.id),
                    newID: self.uniqueID?.id,
                    component: self.getType().identifier,
                    uuid: nil // no UUID for structs
                ): "Unknown error emitting DeFiActions.UpdatedID from IdentifiableStruct \(self.getType().identifier) with ID ".concat(self.id()?.toString() ?? "UNASSIGNED")
            }
        }
    }

    /// IdentifiableResource
    ///
    /// A resource interface containing a UniqueIdentifier and convenience getters about it
    (all) resource interface IdentifiableResource {
        /// An optional identifier allowing protocols to identify stacked connector operations by defining a protocol-
        /// specific Identifier to associated connectors on construction
        access(contract) var uniqueID: UniqueIdentifier?
        /// Convenience method returning the inner UniqueIdentifier's id or `nil` if none is set.
        ///
        /// NOTE: This interface method may be spoofed if the function is overridden, so callers should not rely on it
        /// for critical identification unless the implementation itself is known and trusted
        access(all) view fun id(): UInt64? {
            return self.uniqueID?.id
        }
        /// Returns a ComponentInfo struct containing information about this component and a list of ComponentInfo for
        /// each inner component in the stack.
        access(all) fun getComponentInfo(): ComponentInfo
        /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
        /// a DeFiActions stack. See DeFiActions.align() for more information.
        access(contract) view fun copyID(): UniqueIdentifier? {
            post {
                result?.id == self.uniqueID?.id:
                "UniqueIdentifier of \(self.getType().identifier) was not successfully copied"
            }
        }
        /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
        /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
        access(contract) fun setID(_ id: UniqueIdentifier?) {
            post {
                self.uniqueID?.id == id?.id:
                "UniqueIdentifier of \(self.getType().identifier) was not successfully set"
                DeFiActions.emitUpdatedID(
                    oldID: before(self.uniqueID?.id),
                    newID: self.uniqueID?.id,
                    component: self.getType().identifier,
                    uuid: self.uuid
                ): "Unknown error emitting DeFiActions.UpdatedID from IdentifiableStruct \(self.getType().identifier) with ID ".concat(self.id()?.toString() ?? "UNASSIGNED")
            }
        }
    }

    /// Sink
    ///
    /// A Sink Connector (or just “Sink”) is analogous to the Fungible Token Receiver interface that accepts deposits of
    /// funds. It differs from the standard Receiver interface in that it is a struct interface (instead of resource
    /// interface) and allows for the graceful handling of Sinks that have a limited capacity on the amount they can
    /// accept for deposit. Implementations should therefore avoid the possibility of reversion with graceful fallback
    /// on unexpected conditions, executing no-ops instead of reverting.
    (all) struct interface Sink : IdentifiableStruct {
        /// Returns the Vault type accepted by this Sink
        access(all) view fun getSinkType(): Type
        /// Returns an estimate of how much can be withdrawn from the depositing Vault for this Sink to reach capacity
        access(all) fun minimumCapacity(): UFix64
        /// Deposits up to the Sink's capacity from the provided Vault
        access(all) fun depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault}) {
            pre {
                from.getType() == self.getSinkType():
                "Invalid vault provided for deposit - \(from.getType().identifier) is not \(self.getSinkType().identifier)"
            }
            post {
                DeFiActions.emitDeposited(
                    type: from.getType().identifier,
                    beforeBalance: before(from.balance),
                    afterBalance: from.balance,
                    fromUUID: from.uuid,
                    uniqueID: self.uniqueID?.id,
                    sinkType: self.getType().identifier
                ): "Unknown error emitting DeFiActions.Withdrawn from Sink \(self.getType().identifier) with ID ".concat(self.id()?.toString() ?? "UNASSIGNED")
            }
        }
    }

    /// Source
    ///
    /// A Source Connector (or just “Source”) is analogous to the Fungible Token Provider interface that provides funds
    /// on demand. It differs from the standard Provider interface in that it is a struct interface (instead of resource
    /// interface) and allows for graceful handling of the case that the Source might not know exactly the total amount
    /// of funds available to be withdrawn. Implementations should therefore avoid the possibility of reversion with
    /// graceful fallback on unexpected conditions, executing no-ops or returning an empty Vault instead of reverting.
    (all) struct interface Source : IdentifiableStruct {
        /// Returns the Vault type provided by this Source
        access(all) view fun getSourceType(): Type
        /// Returns an estimate of how much of the associated Vault Type can be provided by this Source
        access(all) fun minimumAvailable(): UFix64
        /// Withdraws the lesser of maxAmount or minimumAvailable(). If none is available, an empty Vault should be
        /// returned
        access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
            post {
                result.getType() == self.getSourceType():
                "Invalid vault provided for withdraw - \(result.getType().identifier) is not \(self.getSourceType().identifier)"
                DeFiActions.emitWithdrawn(
                    type: result.getType().identifier,
                    amount: result.balance,
                    withdrawnUUID: result.uuid,
                    uniqueID: self.uniqueID?.id ?? nil,
                    sourceType: self.getType().identifier
                ): "Unknown error emitting DeFiActions.Withdrawn from Source \(self.getType().identifier) with ID ".concat(self.id()?.toString() ?? "UNASSIGNED")
            }
        }
    }

    /// Quote
    ///
    /// An interface for an estimate to be returned by a Swapper when asking for a swap estimate. This may be helpful
    /// for passing additional parameters to a Swapper relevant to the use case. Implementations may choose to add
    /// fields relevant to their Swapper implementation and downcast in swap() and/or swapBack() scope.
    (all) struct interface Quote {
        /// The quoted pre-swap Vault type
        access(all) let inType: Type
        /// The quoted post-swap Vault type
        access(all) let outType: Type
        /// The quoted amount of pre-swap currency
        access(all) let inAmount: UFix64
        /// The quoted amount of post-swap currency for the defined inAmount
        access(all) let outAmount: UFix64
    }

    /// Swapper
    ///
    /// A basic interface for a struct that swaps between tokens. Implementations may choose to adapt this interface
    /// to fit any given swap protocol or set of protocols.
    (all) struct interface Swapper : IdentifiableStruct {
        /// The type of Vault this Swapper accepts when performing a swap
        access(all) view fun inType(): Type
        /// The type of Vault this Swapper provides when performing a swap
        access(all) view fun outType(): Type
        /// The estimated amount required to provide a Vault with the desired output balance
        access(all) fun quoteIn(forDesired: UFix64, reverse: Bool): {Quote} // fun quoteIn/Out
        /// The estimated amount delivered out for a provided input balance
        access(all) fun quoteOut(forProvided: UFix64, reverse: Bool): {Quote}
        /// Performs a swap taking a Vault of type inVault, outputting a resulting outVault. Implementations may choose
        /// to swap along a pre-set path or an optimal path of a set of paths or even set of contained Swappers adapted
        /// to use multiple Flow swap protocols.
        access(all) fun swap(quote: {Quote}?, inVault: @{FungibleToken.Vault}): @{FungibleToken.Vault} {
            pre {
                inVault.getType() == self.inType():
                "Invalid vault provided for swap - \(inVault.getType().identifier) is not \(self.inType().identifier)"
                (quote?.inType ?? inVault.getType()) == inVault.getType():
                "Quote.inType type \(quote!.inType.identifier) does not match the provided inVault \(inVault.getType().identifier)"
            }
            post {
                emit Swapped(
                    inVault: before(inVault.getType().identifier),
                    outVault: result.getType().identifier,
                    inAmount: before(inVault.balance),
                    outAmount: result.balance,
                    inUUID: before(inVault.uuid),
                    outUUID: result.uuid,
                    uniqueID: self.uniqueID?.id ?? nil,
                    swapperType: self.getType().identifier
                )
            }
        }
        /// Performs a swap taking a Vault of type outVault, outputting a resulting inVault. Implementations may choose
        /// to swap along a pre-set path or an optimal path of a set of paths or even set of contained Swappers adapted
        /// to use multiple Flow swap protocols.
        // TODO: Impl detail - accept quote that was just used by swap() but reverse the direction assuming swap() was just called
        access(all) fun swapBack(quote: {Quote}?, residual: @{FungibleToken.Vault}): @{FungibleToken.Vault} {
            pre {
                residual.getType() == self.outType():
                "Invalid vault provided for swapBack - \(residual.getType().identifier) is not \(self.outType().identifier)"
            }
            post {
                emit Swapped(
                    inVault: before(residual.getType().identifier),
                    outVault: result.getType().identifier,
                    inAmount: before(residual.balance),
                    outAmount: result.balance,
                    inUUID: before(residual.uuid),
                    outUUID: result.uuid,
                    uniqueID: self.uniqueID?.id ?? nil,
                    swapperType: self.getType().identifier
                )
            }
        }
    }

    /// PriceOracle
    ///
    /// An interface for a price oracle adapter. Implementations should adapt this interface to various price feed
    /// oracles deployed on Flow
    (all) struct interface PriceOracle : IdentifiableStruct {
        /// Returns the asset type serving as the price basis - e.g. USD in FLOW/USD
        access(all) view fun unitOfAccount(): Type
        /// Returns the latest price data for a given asset denominated in unitOfAccount() if available, otherwise `nil`
        /// should be returned. Callers should note that although an optional is supported, implementations may choose
        /// to revert.
        access(all) fun price(ofToken: Type): UFix64?
    }

    /// Flasher
    ///
    /// An interface for a flash loan adapter. Implementations should adapt this interface to various flash loan
    /// protocols deployed on Flow
    (all) struct interface Flasher : IdentifiableStruct {
        /// Returns the asset type this Flasher can issue as a flash loan
        access(all) view fun borrowType(): Type
        /// Returns the estimated fee for a flash loan of the specified amount
        access(all) fun calculateFee(loanAmount: UFix64): UFix64
        /// Performs a flash loan of the specified amount. The executor function is passed the fee amount and a Vault
        /// containing the loan. The executor function should return a Vault containing the loan and fee.
        access(all) fun flashLoan(
            amount: UFix64,
            data: AnyStruct?,
            callback: fun(UFix64, @{FungibleToken.Vault}, AnyStruct?): @{FungibleToken.Vault} // fee, loan, data
        ) {
            post {
                emit Flashed(
                    requestedAmount: amount,
                    borrowType: self.borrowType().identifier,
                    uniqueID: self.uniqueID?.id ?? nil,
                    flasherType: self.getType().identifier
                )
            }
        }
    }

    /// AutoBalancerSink
    ///
    /// A DeFiActions Sink enabling the deposit of funds to an underlying AutoBalancer resource. As written, this Source
    /// may be used with externally defined AutoBalancer implementations
    (all) struct AutoBalancerSink : Sink {
        /// The Type this Sink accepts
        access(self) let type: Type
        /// An authorized Capability on the underlying AutoBalancer where funds are deposited
        access(self) let autoBalancer: Capability<&AutoBalancer>
        /// An optional identifier allowing protocols to identify stacked connector operations by defining a protocol-
        /// specific Identifier to associated connectors on construction
        access(contract) var uniqueID: UniqueIdentifier?

        init(autoBalancer: Capability<&AutoBalancer>, uniqueID: UniqueIdentifier?) {
            pre {
                autoBalancer.check():
                "Invalid AutoBalancer Capability Provided"
            }
            self.type = autoBalancer.borrow()!.vaultType()
            self.autoBalancer = autoBalancer
            self.uniqueID = uniqueID
        }

        /// Returns the Vault type accepted by this Sink
        access(all) view fun getSinkType(): Type {
            return self.type
        }
        /// Returns an estimate of how much can be withdrawn from the depositing Vault for this Sink to reach capacity
        /// can currently only be UFix64.max or 0.0
        access(all) fun minimumCapacity(): UFix64 {
            return self.autoBalancer.check() ? UFix64.max : 0.0
        }
        /// Deposits up to the Sink's capacity from the provided Vault
        access(all) fun depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault}) {
            if let ab = self.autoBalancer.borrow() {
                ab.deposit(from: <-from.withdraw(amount: from.balance))
            }
            return
        }
        /// Returns a ComponentInfo struct containing information about this component and a list of ComponentInfo for
        /// each inner component in the stack.
        access(all) fun getComponentInfo(): ComponentInfo {
            return ComponentInfo(
                type: self.getType(),
                id: self.id(),
                innerComponents: []
            )
        }
        /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
        /// a DeFiActions stack. See DeFiActions.align() for more information.
        access(contract) view fun copyID(): UniqueIdentifier? {
            return self.uniqueID
        }
        /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
        /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
        access(contract) fun setID(_ id: UniqueIdentifier?) {
            self.uniqueID = id
        }
    }

    /// AutoBalancerSource
    ///
    /// A DeFiActions Source targeting an underlying AutoBalancer resource. As written, this Source may be used with
    /// externally defined AutoBalancer implementations
    (all) struct AutoBalancerSource : Source {
        /// The Type this Source provides
        access(self) let type: Type
        /// An authorized Capability on the underlying AutoBalancer where funds are sourced
        access(self) let autoBalancer: Capability<auth(FungibleToken.Withdraw) &AutoBalancer>
        /// An optional identifier allowing protocols to identify stacked connector operations by defining a protocol-
        /// specific Identifier to associated connectors on construction
        access(contract) var uniqueID: UniqueIdentifier?

        init(autoBalancer: Capability<auth(FungibleToken.Withdraw) &AutoBalancer>, uniqueID: UniqueIdentifier?) {
            pre {
                autoBalancer.check():
                "Invalid AutoBalancer Capability Provided"
            }
            self.type = autoBalancer.borrow()!.vaultType()
            self.autoBalancer = autoBalancer
            self.uniqueID = uniqueID
        }

        /// Returns the Vault type provided by this Source
        access(all) view fun getSourceType(): Type {
            return self.type
        }
        /// Returns an estimate of how much of the associated Vault Type can be provided by this Source
        access(all) fun minimumAvailable(): UFix64 {
            if let ab = self.autoBalancer.borrow() {
                return ab.vaultBalance()
            }
            return 0.0
        }
        /// Withdraws the lesser of maxAmount or minimumAvailable(). If none is available, an empty Vault should be
        /// returned
        access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
            if let ab = self.autoBalancer.borrow() {
                return <-ab.withdraw(
                    amount: maxAmount <= ab.vaultBalance() ? maxAmount : ab.vaultBalance()
                )
            }
            return <- DeFiActionsUtils.getEmptyVault(self.type)
        }
        /// Returns a ComponentInfo struct containing information about this component and a list of ComponentInfo for
        /// each inner component in the stack.
        access(all) fun getComponentInfo(): ComponentInfo {
            return ComponentInfo(
                type: self.getType(),
                id: self.id(),
                innerComponents: []
            )
        }
        /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
        /// a DeFiActions stack. See DeFiActions.align() for more information.
        access(contract) view fun copyID(): UniqueIdentifier? {
            return self.uniqueID
        }
        /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
        /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
        access(contract) fun setID(_ id: UniqueIdentifier?) {
            self.uniqueID = id
        }
    }

    /// Entitlement used by the AutoBalancer to set inner Sink and Source
    access(all) entitlement Auto
    access(all) entitlement Set
    access(all) entitlement Get

    /// AutoBalancer
    ///
    /// A resource designed to enable permissionless rebalancing of value around a wrapped Vault. An
    /// AutoBalancer can be a critical component of DeFiActions stacks by allowing for strategies to compound, repay
    /// loans or direct accumulated value to other sub-systems and/or user Vaults.
    (all) resource AutoBalancer : IdentifiableResource, FungibleToken.Receiver, FungibleToken.Provider, ViewResolver.Resolver, Burner.Burnable {
        /// The value in deposits & withdrawals over time denominated in oracle.unitOfAccount()
        access(self) var _valueOfDeposits: UFix64
        /// The percentage low and high thresholds defining when a rebalance executes
        /// Index 0 is low, index 1 is high
        access(self) var _rebalanceRange: [UFix64; 2]
        /// Oracle used to track the baseValue for deposits & withdrawals over time
        access(self) let _oracle: {PriceOracle}
        /// The inner Vault's Type captured for the ResourceDestroyed event
        access(self) let _vaultType: Type
        /// Vault used to deposit & withdraw from made optional only so the Vault can be burned via Burner.burn() if the
        /// AutoBalancer is burned and the Vault's burnCallback() can be called in the process
        access(self) var _vault: @{FungibleToken.Vault}?
        /// An optional Sink used to deposit excess funds from the inner Vault once the converted value exceeds the
        /// rebalance range. This Sink may be used to compound yield into a position or direct excess value to an
        /// external Vault
        access(self) var _rebalanceSink: {Sink}?
        /// An optional Source used to deposit excess funds to the inner Vault once the converted value is below the
        /// rebalance range
        access(self) var _rebalanceSource: {Source}?
        /// Capability on this AutoBalancer instance
        access(self) var _selfCap: Capability<auth(FungibleToken.Withdraw) &AutoBalancer>?
        /// An optional UniqueIdentifier tying this AutoBalancer to a given stack
        access(contract) var uniqueID: UniqueIdentifier?

        /// Emitted when the AutoBalancer is destroyed
        access(all) event ResourceDestroyed(
            uuid: UInt64 = self.uuid,
            vaultType: String = self._vaultType.identifier,
            balance: UFix64? = self._vault?.balance,
            uniqueID: UInt64? = self.uniqueID?.id
        )

        init(
            lower: UFix64,
            upper: UFix64,
            oracle: {PriceOracle},
            vaultType: Type,
            outSink: {Sink}?,
            inSource: {Source}?,
            uniqueID: UniqueIdentifier?
        ) {
            pre {
                lower < upper && 0.01 <= lower && lower < 1.0 && 1.0 < upper && upper < 2.0:
                "Invalid rebalanceRange [lower, upper]: [\(lower), \(upper)] - thresholds must be set such that 0.01 <= lower < 1.0 and 1.0 < upper < 2.0 relative to value of deposits"
                DeFiActionsUtils.definingContractIsFungibleToken(vaultType):
                "The contract defining Vault \(vaultType.identifier) does not conform to FungibleToken contract interface"
            }
            assert(oracle.price(ofToken: vaultType) != nil,
                message: "Provided Oracle \(oracle.getType().identifier) could not provide a price for vault \(vaultType.identifier)")
            self._valueOfDeposits = 0.0
            self._rebalanceRange = [lower, upper]
            self._oracle = oracle
            self._vault <- DeFiActionsUtils.getEmptyVault(vaultType)
            self._vaultType = vaultType
            self._rebalanceSink = outSink
            self._rebalanceSource = inSource
            self._selfCap = nil
            self.uniqueID = uniqueID

            emit CreatedAutoBalancer(
                lowerThreshold: lower,
                upperThreshold: upper,
                vaultType: vaultType.identifier,
                vaultUUID: self._borrowVault().uuid,
                uuid: self.uuid,
                uniqueID: self.id()
            )
        }

        /* Core AutoBalancer Functionality */

        /// Returns the balance of the inner Vault
        ///
        /// @return the current balance of the inner Vault
        ///
        access(all) view fun vaultBalance(): UFix64 {
            return self._borrowVault().balance
        }
        /// Returns the Type of the inner Vault
        ///
        /// @return the Type of the inner Vault
        ///
        access(all) view fun vaultType(): Type {
            return self._borrowVault().getType()
        }
        /// Returns the low and high rebalance thresholds as a fixed length UFix64 containing [low, high]
        ///
        /// @return a sorted fixed-length array containing the relative lower and upper thresholds conditioning
        ///     rebalance execution
        ///
        access(all) view fun rebalanceThresholds(): [UFix64; 2] {
            return self._rebalanceRange
        }
        /// Returns the value of all accounted deposits/withdraws as they have occurred denominated in unitOfAccount.
        /// The returned value is the value as tracked historically, not necessarily the current value of the inner
        /// Vault's balance.
        ///
        /// @return the historical value of deposits
        ///
        access(all) view fun valueOfDeposits(): UFix64 {
            return self._valueOfDeposits
        }
        /// Returns the token Type serving as the price basis of this AutoBalancer
        ///
        /// @return the price denomination of value of the underlying vault as returned from the inner PriceOracle
        ///
        access(all) view fun unitOfAccount(): Type {
            return self._oracle.unitOfAccount()
        }
        /// Returns the current value of the inner Vault's balance. If a price is not available from the AutoBalancer's
        /// PriceOracle, `nil` is returned
        ///
        /// @return the current value of the inner's Vault's balance denominated in unitOfAccount() if a price is
        ///     available, `nil` otherwise
        ///
        access(all) fun currentValue(): UFix64? {
            if let price = self._oracle.price(ofToken: self.vaultType()) {
                return price * self._borrowVault().balance
            }
            return nil
        }
        /// Returns a ComponentInfo struct containing information about this AutoBalancer and its inner DFA components
        ///
        /// @return a ComponentInfo struct containing information about this component and a list of ComponentInfo for
        ///     each inner component in the stack.
        ///
        access(all) fun getComponentInfo(): ComponentInfo {
            // get the inner components
            let oracle = self._borrowOracle()
            let inner: [ComponentInfo] = [oracle.getComponentInfo()]

            // get the info for the optional inner components if they exist
            let maybeSink = self._borrowSink()
            let maybeSource = self._borrowSource()
            if let sink = maybeSink {
                inner.append(sink.getComponentInfo())
            }
            if let source = maybeSource {
                inner.append(source.getComponentInfo())
            }

            // create the ComponentInfo for the AutoBalancer and insert it at the beginning of the list
            return ComponentInfo(
                type: self.getType(),
                id: self.id(),
                innerComponents: inner
            )
        }
        /// Convenience method issuing a Sink allowing for deposits to this AutoBalancer. If the AutoBalancer's
        /// Capability on itself is not set or is invalid, `nil` is returned.
        ///
        /// @return a Sink routing deposits to this AutoBalancer
        ///
        access(all) fun createBalancerSink(): {Sink}? {
            if self._selfCap == nil || !self._selfCap!.check() {
                return nil
            }
            return AutoBalancerSink(autoBalancer: self._selfCap!, uniqueID: self.uniqueID)
        }
        /// Convenience method issuing a Source enabling withdrawals from this AutoBalancer. If the AutoBalancer's
        /// Capability on itself is not set or is invalid, `nil` is returned.
        ///
        /// @return a Source routing withdrawals from this AutoBalancer
        ///
        access(Get) fun createBalancerSource(): {Source}? {
            if self._selfCap == nil || !self._selfCap!.check() {
                return nil
            }
            return AutoBalancerSource(autoBalancer: self._selfCap!, uniqueID: self.uniqueID)
        }
        /// A setter enabling an AutoBalancer to set a Sink to which overflow value should be deposited
        ///
        /// @param sink: The optional Sink DeFiActions connector from which funds are sourced when this AutoBalancer
        ///     current value rises above the upper threshold relative to its valueOfDeposits(). If `nil`, overflown
        ///     value will not rebalance
        ///
        access(Set) fun setSink(_ sink: {Sink}?, updateSinkID: Bool) {
            if sink != nil && updateSinkID {
                let toUpdate = &sink! as auth(Extend) &{IdentifiableStruct}
                let toAlign = &self as auth(Identify) &{IdentifiableResource}
                DeFiActions.alignID(toUpdate: toUpdate, with: toAlign)
            }
            self._rebalanceSink = sink
        }
        /// A setter enabling an AutoBalancer to set a Source from which underflow value should be withdrawn
        ///
        /// @param source: The optional Source DeFiActions connector from which funds are sourced when this AutoBalancer
        ///     current value falls below the lower threshold relative to its valueOfDeposits(). If `nil`, underflown
        ///     value will not rebalance
        ///
        access(Set) fun setSource(_ source: {Source}?, updateSourceID: Bool) {
            if source != nil && updateSourceID {
                let toUpdate = &source! as auth(Extend) &{IdentifiableStruct}
                let toAlign = &self as auth(Identify) &{IdentifiableResource}
                DeFiActions.alignID(toUpdate: toUpdate, with: toAlign)
            }
            self._rebalanceSource = source
        }
        /// Enables the setting of a Capability on the AutoBalancer for the distribution of Sinks & Sources targeting
        /// the AutoBalancer instance. Due to the mechanisms of Capabilities, this must be done after the AutoBalancer
        /// has been saved to account storage and an authorized Capability has been issued.
        access(Set) fun setSelfCapability(_ cap: Capability<auth(FungibleToken.Withdraw) &AutoBalancer>) {
            pre {
                self._selfCap == nil || self._selfCap!.check() != true:
                "Internal AutoBalancer Capability has been set and is still valid - cannot be re-assigned"
                cap.check(): "Invalid AutoBalancer Capability provided"
                self.getType() == cap.borrow()!.getType() && self.uuid == cap.borrow()!.uuid:
                "Provided Capability does not target this AutoBalancer of type \(self.getType().identifier) with UUID \(self.uuid) - "
                    .concat("provided Capability for AutoBalancer of type \(cap.borrow()!.getType().identifier) with UUID \(cap.borrow()!.uuid)")
            }
            self._selfCap = cap
        }
        /// Sets the rebalance range of this AutoBalancer
        ///
        /// @param range: a sorted array containing lower and upper thresholds that condition rebalance execution. The
        ///     thresholds must be values such that 0.01 <= range[0] < 1.0 && 1.0 < range[1] < 2.0
        ///
        access(Set) fun setRebalanceRange(_ range: [UFix64; 2]) {
            pre {
                range[0] < range[1] && 0.01 <= range[0] && range[0] < 1.0 && 1.0 < range[1] && range[1] < 2.0:
                "Invalid rebalanceRange [lower, upper]: [\(range[0]), \(range[1])] - thresholds must be set such that 0.01 <= range[0] < 1.0 and 1.0 < range[1] < 2.0 relative to value of deposits"
            }
            self._rebalanceRange = range
        }
        /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
        /// a DeFiActions stack. See DeFiActions.align() for more information.
        access(contract) view fun copyID(): UniqueIdentifier? {
            return self.uniqueID
        }
        /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
        /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
        access(contract) fun setID(_ id: UniqueIdentifier?) {
            self.uniqueID = id
        }
        /// Allows for external parties to call on the AutoBalancer and execute a rebalance according to it's rebalance
        /// parameters. This method must be called by external party regularly in order for rebalancing to occur.
        ///
        /// @param force: if false, rebalance will occur only when beyond upper or lower thresholds; if true, rebalance
        ///     will execute as long as a price is available via the oracle and the current value is non-zero
        ///
        access(Auto) fun rebalance(force: Bool) {
            let currentPrice = self._oracle.price(ofToken: self._vaultType)
            if currentPrice == nil {
                return // no price available -> do nothing
            }
            let currentValue = self.currentValue()!
            // calculate the difference between the current value and the historical value of deposits
            var valueDiff: UFix64 = currentValue < self._valueOfDeposits ? self._valueOfDeposits - currentValue : currentValue - self._valueOfDeposits
            // if deficit detected, choose lower threshold, otherwise choose upper threshold
            let isDeficit = currentValue < self._valueOfDeposits
            let threshold = isDeficit ? (1.0 - self._rebalanceRange[0]) : (self._rebalanceRange[1] - 1.0)

            if currentPrice == 0.0 || valueDiff == 0.0 || ((valueDiff / self._valueOfDeposits) < threshold && !force) {
                // division by zero, no difference, or difference does not exceed rebalance ratio & not forced -> no-op
                return
            }

            let vault = self._borrowVault()
            var amount = valueDiff / currentPrice!
            var executed = false
            let maybeRebalanceSource = &self._rebalanceSource as auth(FungibleToken.Withdraw) &{Source}?
            let maybeRebalanceSink = &self._rebalanceSink as &{Sink}?
            if isDeficit && maybeRebalanceSource != nil {
                // rebalance back up to baseline sourcing funds from _rebalanceSource
                vault.deposit(from:  <- maybeRebalanceSource!.withdrawAvailable(maxAmount: amount))
                executed = true
            } else if !isDeficit && maybeRebalanceSink != nil {
                // rebalance back down to baseline depositing excess to _rebalanceSink
                if amount > vault.balance {
                    amount = vault.balance // protect underflow
                }
                let surplus <- vault.withdraw(amount: amount)
                maybeRebalanceSink!.depositCapacity(from: &surplus as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
                executed = true
                if surplus.balance == 0.0 {
                    Burner.burn(<-surplus) // could destroy
                } else {
                    amount = amount - surplus.balance // update the rebalanced amount
                    valueDiff = valueDiff - (surplus.balance * currentPrice!) // update the value difference
                    vault.deposit(from: <-surplus) // deposit any excess not taken by the Sink
                }
            }
            // emit event only if rebalance was executed
            if executed {
                emit Rebalanced(
                    amount: amount,
                    value: valueDiff,
                    unitOfAccount: self.unitOfAccount().identifier,
                    isSurplus: !isDeficit,
                    vaultType: self.vaultType().identifier,
                    vaultUUID: self._borrowVault().uuid,
                    balancerUUID: self.uuid,
                    address: self.owner?.address,
                    uuid: self.uuid,
                    uniqueID: self.id()
                )
            }
        }

        /* ViewResolver.Resolver conformance */

        /// Passthrough to inner Vault's view Types
        access(all) view fun getViews(): [Type] {
            return self._borrowVault().getViews()
        }
        /// Passthrough to inner Vault's view resolution
        access(all) fun resolveView(_ view: Type): AnyStruct? {
            return self._borrowVault().resolveView(view)
        }

        /* FungibleToken.Receiver & .Provider conformance */

        /// Only the nested Vault type is supported by this AutoBalancer for deposits & withdrawal for the sake of
        /// single asset accounting
        access(all) view fun getSupportedVaultTypes(): {Type: Bool} {
            return { self.vaultType(): true }
        }
        /// True if the provided Type is the nested Vault Type, false otherwise
        access(all) view fun isSupportedVaultType(type: Type): Bool {
            return self.getSupportedVaultTypes()[type] == true
        }
        /// Passthrough to the inner Vault's isAvailableToWithdraw() method
        access(all) view fun isAvailableToWithdraw(amount: UFix64): Bool {
            return self._borrowVault().isAvailableToWithdraw(amount: amount)
        }
        /// Deposits the provided Vault to the nested Vault if it is of the same Type, reverting otherwise. In the
        /// process, the current value of the deposited amount (denominated in unitOfAccount) increments the
        /// AutoBalancer's baseValue. If a price is not available via the internal PriceOracle, an average price is
        /// calculated base on the inner vault balance & valueOfDeposits and valueOfDeposits is incremented by the
        /// value of the deposited vault on the basis of that average
        access(all) fun deposit(from: @{FungibleToken.Vault}) {
            pre {
                from.getType() == self.vaultType():
                "Invalid Vault type \(from.getType().identifier) deposited - this AutoBalancer only accepts \(self.vaultType().identifier)"
            }
            // if no price available use an average price based on historical value of deposits and inner vault balance
            let price = self._oracle.price(ofToken: from.getType()) ?? (self.valueOfDeposits() / self.vaultBalance())
            self._valueOfDeposits = self._valueOfDeposits + (from.balance * price)
            self._borrowVault().deposit(from: <-from)
        }
        /// Returns the requested amount of the nested Vault type, reducing the baseValue by the current value
        /// (denominated in unitOfAccount) of the token amount. The AutoBalancer's valueOfDeposits is decremented
        /// in proportion to the amount withdrawn relative to the inner Vault's balance
        access(FungibleToken.Withdraw) fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
            pre {
                amount <= self.vaultBalance(): "Withdraw amount \(amount) exceeds current vault balance \(self.vaultBalance())"
            }
            if amount == 0.0 {
                return <- self._borrowVault().createEmptyVault()
            }
            // adjust historical value of deposits proportionate to the amount withdrawn & return withdrawn vault
            self._valueOfDeposits = (1.0 - amount / self.vaultBalance()) * self._valueOfDeposits
            return <- self._borrowVault().withdraw(amount: amount)
        }

        /* Burnable.Burner conformance */

        /// Executed in Burner.burn(). Passes along the inner vault to be burned, executing the inner Vault's
        /// burnCallback() logic
        access(contract) fun burnCallback() {
            let vault <- self._vault <- nil
            Burner.burn(<-vault) // executes the inner Vault's burnCallback()
        }

        /* Internal */

        /// Returns a reference to the inner Vault
        access(self) view fun _borrowVault(): auth(FungibleToken.Withdraw) &{FungibleToken.Vault} {
            return (&self._vault)!
        }
        /// Returns a reference to the inner Vault
        access(self) view fun _borrowOracle(): &{PriceOracle} {
            return &self._oracle
        }
        /// Returns a reference to the inner Vault
        access(self) view fun _borrowSink(): &{Sink}? {
            return &self._rebalanceSink
        }
        /// Returns a reference to the inner Source
        access(self) view fun _borrowSource(): auth(FungibleToken.Withdraw) &{Source}? {
            return &self._rebalanceSource as auth(FungibleToken.Withdraw) &{Source}?
        }
    }

    /* --- PUBLIC METHODS --- */

    /// Returns an AutoBalancer wrapping the provided Vault.
    ///
    /// @param oracle: The oracle used to query deposited & withdrawn value and to determine if a rebalance should execute
    /// @param vault: The Vault wrapped by the AutoBalancer
    /// @param rebalanceRange: The percentage range from the AutoBalancer's base value at which a rebalance is executed
    /// @param outSink: An optional DeFiActions Sink to which excess value is directed when rebalancing
    /// @param inSource: An optional DeFiActions Source from which value is withdrawn to the inner vault when rebalancing
    /// @param uniqueID: An optional DeFiActions UniqueIdentifier used for identifying rebalance events
    (all) fun createAutoBalancer(
        oracle: {PriceOracle},
        vaultType: Type,
        lowerThreshold: UFix64,
        upperThreshold: UFix64,
        rebalanceSink: {Sink}?,
        rebalanceSource: {Source}?,
        uniqueID: UniqueIdentifier?
    ): @AutoBalancer {
        let ab <- create AutoBalancer(
            lower: lowerThreshold,
            upper: upperThreshold,
            oracle: oracle,
            vaultType: vaultType,
            outSink: rebalanceSink,
            inSource: rebalanceSource,
            uniqueID: uniqueID
        )
        return <- ab
    }

    /// Creates a new UniqueIdentifier used for identifying action stacks
    ///
    /// @return a new UniqueIdentifier
    (all) fun createUniqueIdentifier(): UniqueIdentifier {
        let id = UniqueIdentifier(self.currentID, self.authTokenCap)
        self.currentID = self.currentID + 1
        return id
    }

    /// Aligns the UniqueIdentifier of the provided component with the provided component, setting the UniqueIdentifier of
    /// the provided component to the UniqueIdentifier of the provided component. Parameters are AnyStruct to allow for
    /// alignment of both IdentifiableStruct and IdentifiableResource. However, note that the provided component must
    /// be an auth(Extend) &{IdentifiableStruct} or auth(Extend) &{IdentifiableResource} to be aligned.
    ///
    /// @param toUpdate: The component to update the UniqueIdentifier of. Must be an auth(Extend) &{IdentifiableStruct}
    ///     or auth(Extend) &{IdentifiableResource}
    /// @param with: The component to align the UniqueIdentifier of the provided component with. Must be an
    ///     auth(Identify) &{IdentifiableStruct} or auth(Identify) &{IdentifiableResource}
    (all) fun alignID(toUpdate: AnyStruct, with: AnyStruct) {
        let maybeISToUpdate = toUpdate as? auth(Extend) &{IdentifiableStruct}
        let maybeIRToUpdate = toUpdate as? auth(Extend) &{IdentifiableResource}
        let maybeISWith = with as? auth(Identify) &{IdentifiableStruct}
        let maybeIRWith = with as? auth(Identify) &{IdentifiableResource}

        if maybeISToUpdate != nil && maybeISWith != nil {
            maybeISToUpdate!.setID(maybeISWith!.copyID())
        } else if maybeISToUpdate != nil && maybeIRWith != nil {
            maybeISToUpdate!.setID(maybeIRWith!.copyID())
        } else if maybeIRToUpdate != nil && maybeISWith != nil {
            maybeIRToUpdate!.setID(maybeISWith!.copyID())
        } else if maybeIRToUpdate != nil && maybeIRWith != nil {
            maybeIRToUpdate!.setID(maybeIRWith!.copyID())
        }
        return
    }

    /* --- INTERNAL CONDITIONAL EVENT EMITTERS --- */

    /// Emits Deposited event if a change in balance is detected
    access(self) view fun emitDeposited(
        type: String,
        beforeBalance: UFix64,
        afterBalance: UFix64,
        fromUUID: UInt64,
        uniqueID: UInt64?,
        sinkType: String
    ): Bool {
        if beforeBalance == afterBalance {
            return true
        }
        emit Deposited(
            type: type,
            amount: beforeBalance > afterBalance ? beforeBalance - afterBalance : afterBalance - beforeBalance,
            fromUUID: fromUUID,
            uniqueID: uniqueID,
            sinkType: sinkType
        )
        return true
    }

    /// Emits Withdrawn event if a change in balance is detected
    access(self) view fun emitWithdrawn(
        type: String,
        amount: UFix64,
        withdrawnUUID: UInt64,
        uniqueID: UInt64?,
        sourceType: String
    ): Bool {
        if amount == 0.0 {
            return true
        }
        emit Withdrawn(
            type: type,
            amount: amount,
            withdrawnUUID: withdrawnUUID,
            uniqueID: uniqueID,
            sourceType: sourceType
        )
        return true
    }

    /// Emits Aligned event if a change in UniqueIdentifier is detected
    access(self) view fun emitUpdatedID(
        oldID: UInt64?,
        newID: UInt64?,
        component: String,
        uuid: UInt64?
    ): Bool {
        if oldID == newID {
            return true
        }
        emit UpdatedID(
            oldID: oldID,
            newID: newID,
            component: component,
            uuid: uuid
        )
        return true
    }

    init() {
        self.currentID = 0
        self.AuthTokenStoragePath = /storage/authToken

        self.account.storage.save(<-create AuthenticationToken(), to: self.AuthTokenStoragePath)
        self.authTokenCap = self.account.capabilities.storage.issue<auth(Identify) &AuthenticationToken>(self.AuthTokenStoragePath)

        assert(self.authTokenCap.check(), message: "Failed to issue AuthenticationToken Capability")
    }
}
```

</details>

### Connector Examples

#### FungibleToken Connectors

Basic connectors for interacting with standard FungibleToken Vaults:

<details>

<summary>VaultSink & VaultSource implementations</summary>

```cadence
access(all) struct VaultSink : DeFiActions.Sink {
    /// The Vault Type accepted by the Sink
    access(all) let depositVaultType: Type
    /// The maximum balance of the linked Vault, checked before executing a deposit
    access(all) let maximumBalance: UFix64
    /// An optional identifier allowing protocols to identify stacked connector operations by defining a protocol-
    /// specific Identifier to associated connectors on construction
    access(contract) var uniqueID: DeFiActions.UniqueIdentifier?
    /// An unentitled Capability on the Vault to which deposits are distributed
    access(self) let depositVault: Capability<&{FungibleToken.Vault}>

    init(
        max: UFix64?,
        depositVault: Capability<&{FungibleToken.Vault}>,
        uniqueID: DeFiActions.UniqueIdentifier?
    ) {
        pre {
            depositVault.check(): "Provided invalid Capability"
            DeFiActionsUtils.definingContractIsFungibleToken(depositVault.borrow()!.getType()):
            "The contract defining Vault \(depositVault.borrow()!.getType().identifier) does not conform to FungibleToken contract interface"
        }
        self.maximumBalance = max ?? UFix64.max // assume no maximum if none provided
        self.uniqueID = uniqueID
        self.depositVaultType = depositVault.borrow()!.getType()
        self.depositVault = depositVault
    }

    /// Returns a list of ComponentInfo for each component in the stack
    ///
    /// @return a list of ComponentInfo for each inner DeFiActions component in the VaultSink
    (all) fun getComponentInfo(): DeFiActions.ComponentInfo {
        return DeFiActions.ComponentInfo(
            type: self.getType(),
            id: self.id(),
            innerComponents: []
        )
    }
    /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
    /// a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @return a copy of the struct's UniqueIdentifier
    (contract) view fun copyID(): DeFiActions.UniqueIdentifier? {
        return self.uniqueID
    }
    /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
    /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @param id: the UniqueIdentifier to set for this component
    (contract) fun setID(_ id: DeFiActions.UniqueIdentifier?) {
        self.uniqueID = id
    }
    /// Returns the Vault type accepted by this Sink
    access(all) view fun getSinkType(): Type {
        return self.depositVaultType
    }
    /// Returns an estimate of how much of the associated Vault can be accepted by this Sink
    access(all) fun minimumCapacity(): UFix64 {
        if let vault = self.depositVault.borrow() {
            return vault.balance < self.maximumBalance ? self.maximumBalance - vault.balance : 0.0
        }
        return 0.0
    }
    /// Deposits up to the Sink's capacity from the provided Vault
    access(all) fun depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault}) {
        let minimumCapacity = self.minimumCapacity()
        if !self.depositVault.check() || minimumCapacity == 0.0 {
            return
        }
        // deposit the lesser of the originating vault balance and minimum capacity
        let capacity = minimumCapacity <= from.balance ? minimumCapacity : from.balance
        self.depositVault.borrow()!.deposit(from: <-from.withdraw(amount: capacity))
    }
}

access(all) struct VaultSource : DeFiActions.Source {
    /// Returns the Vault type provided by this Source
    access(all) let withdrawVaultType: Type
    /// The minimum balance of the linked Vault
    access(all) let minimumBalance: UFix64
    /// An optional identifier allowing protocols to identify stacked connector operations by defining a protocol-
    /// specific Identifier to associated connectors on construction
    access(contract) var uniqueID: DeFiActions.UniqueIdentifier?
    /// An entitled Capability on the Vault from which withdrawals are sourced
    access(self) let withdrawVault: Capability<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>

    init(
        min: UFix64?,
        withdrawVault: Capability<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>,
        uniqueID: DeFiActions.UniqueIdentifier?
    ) {
        pre {
            withdrawVault.check(): "Provided invalid Capability"
            DeFiActionsUtils.definingContractIsFungibleToken(withdrawVault.borrow()!.getType()):
            "The contract defining Vault \(withdrawVault.borrow()!.getType().identifier) does not conform to FungibleToken contract interface"
        }
        self.minimumBalance = min ?? 0.0 // assume no minimum if none provided
        self.withdrawVault = withdrawVault
        self.uniqueID = uniqueID
        self.withdrawVaultType = withdrawVault.borrow()!.getType()
    }
    /// Returns a list of ComponentInfo for each component in the stack
    ///
    /// @return a list of ComponentInfo for each inner DeFiActions component in the VaultSource
    (all) fun getComponentInfo(): DeFiActions.ComponentInfo {
        return DeFiActions.ComponentInfo(
            type: self.getType(),
            id: self.id(),
            innerComponents: []
        )
    }
    /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
    /// a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @return a copy of the struct's UniqueIdentifier
    (contract) view fun copyID(): DeFiActions.UniqueIdentifier? {
        return self.uniqueID
    }
    /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
    /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @param id: the UniqueIdentifier to set for this component
    (contract) fun setID(_ id: DeFiActions.UniqueIdentifier?) {
        self.uniqueID = id
    }
    /// Returns the Vault type provided by this Source
    access(all) view fun getSourceType(): Type {
        return self.withdrawVaultType
    }
    /// Returns an estimate of how much of the associated Vault can be provided by this Source
    access(all) fun minimumAvailable(): UFix64 {
        if let vault = self.withdrawVault.borrow() {
            return self.minimumBalance < vault.balance ? vault.balance - self.minimumBalance : 0.0
        }
        return 0.0
    }
    /// Withdraws the lesser of maxAmount or minimumAvailable(). If none is available, an empty Vault should be
    /// returned
    access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
        let available = self.minimumAvailable()
        if !self.withdrawVault.check() || available == 0.0 || maxAmount == 0.0 {
            return <- DeFiActionsUtils.getEmptyVault(self.withdrawVaultType)
        }
        // take the lesser between the available and maximum requested amount
        let withdrawalAmount = available <= maxAmount ? available : maxAmount
        return <- self.withdrawVault.borrow()!.withdraw(amount: withdrawalAmount)
    }
}
```
</details>

#### Swap Connectors

Connectors that combine swapping with other actions:

<details>
<summary>SwapSink & SwapSource implementations</summary>

```cadence
/// SwapSink DeFiActions connector that deposits the resulting post-conversion currency of a token swap to an inner
/// DeFiActions Sink, sourcing funds from a deposited Vault of a pre-set Type.
///
access(all) struct SwapSink : DeFiActions.Sink {
    access(self) let swapper: {DeFiActions.Swapper}
    access(self) let sink: {DeFiActions.Sink}
    access(contract) var uniqueID: DeFiActions.UniqueIdentifier?

    init(swapper: {DeFiActions.Swapper}, sink: {DeFiActions.Sink}, uniqueID: DeFiActions.UniqueIdentifier?) {
        pre {
            swapper.outType() == sink.getSinkType():
            "Swapper outputs \(swapper.outType().identifier) but Sink takes \(sink.getSinkType().identifier) - "
                .concat("Ensure the provided Swapper outputs a Vault Type compatible with the provided Sink")
        }
        self.swapper = swapper
        self.sink = sink
        self.uniqueID = uniqueID
    }

    /// Returns a ComponentInfo struct containing information about this SwapSink and its inner DFA components
    ///
    /// @return a ComponentInfo struct containing information about this component and a list of ComponentInfo for
    ///     each inner component in the stack.
    (all) fun getComponentInfo(): DeFiActions.ComponentInfo {
        return DeFiActions.ComponentInfo(
            type: self.getType(),
            id: self.id(),
            innerComponents: [
                self.swapper.getComponentInfo(),
                self.sink.getComponentInfo()
            ]
        )
    }
    /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
    /// a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @return a copy of the struct's UniqueIdentifier
    (contract) view fun copyID(): DeFiActions.UniqueIdentifier? {
        return self.uniqueID
    }
    /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
    /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @param id: the UniqueIdentifier to set for this component
    (contract) fun setID(_ id: DeFiActions.UniqueIdentifier?) {
        self.uniqueID = id
    }
    /// Returns the type of Vault this Sink accepts when performing a swap
    ///
    /// @return the type of Vault this Sink accepts when performing a swap
    (all) view fun getSinkType(): Type {
        return self.swapper.inType()
    }
    /// Returns the minimum capacity required to deposit to this Sink
    ///
    /// @return the minimum capacity required to deposit to this Sink
    (all) fun minimumCapacity(): UFix64 {
        return self.swapper.quoteIn(forDesired: self.sink.minimumCapacity(), reverse: false).inAmount
    }
    /// Deposits the provided Vault to this Sink, swapping the provided Vault to the required type if necessary
    ///
    /// @param from: the Vault to source deposits from
    (all) fun depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault}) {
        let limit = self.sink.minimumCapacity()
        if from.balance == 0.0 || limit == 0.0 || from.getType() != self.getSinkType() {
            return // nothing to swap from, no capacity to ingest, invalid Vault type - do nothing
        }

        let quote = self.swapper.quoteIn(forDesired: limit, reverse: false)
        let swapVault <- from.createEmptyVault()
        if from.balance <= quote.inAmount  {
            // sink can accept all of the available tokens, so we swap everything
            swapVault.deposit(from: <-from.withdraw(amount: from.balance))
        } else {
            // sink is limited to fewer tokens than we have available - swap the amount we need to meet the limit
            swapVault.deposit(from: <-from.withdraw(amount: quote.inAmount))
        }

        // swap then deposit to the inner sink
        let swappedTokens <- self.swapper.swap(quote: quote, inVault: <-swapVault)
        self.sink.depositCapacity(from: &swappedTokens as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})

        if swappedTokens.balance > 0.0 {
            // swap back any residual to the originating vault
            let residual <- self.swapper.swapBack(quote: nil, residual: <-swappedTokens)
            from.deposit(from: <-residual)
        } else {
            Burner.burn(<-swappedTokens) // nothing left - burn & execute vault's burnCallback()
        }
    }
}

/// SwapSource DeFiActions connector that returns post-conversion currency, sourcing pre-converted funds from an inner
/// DeFiActions Source
///
access(all) struct SwapSource : DeFiActions.Source {
    access(self) let swapper: {DeFiActions.Swapper}
    access(self) let source: {DeFiActions.Source}
    access(contract) var uniqueID: DeFiActions.UniqueIdentifier?

    init(swapper: {DeFiActions.Swapper}, source: {DeFiActions.Source}, uniqueID: DeFiActions.UniqueIdentifier?) {
        pre {
            source.getSourceType() == swapper.inType():
            "Source outputs \(source.getSourceType().identifier) but Swapper takes \(swapper.inType().identifier) - "
                .concat("Ensure the provided Source outputs a Vault Type compatible with the provided Swapper")
        }
        self.swapper = swapper
        self.source = source
        self.uniqueID = uniqueID
    }

    /// Returns a ComponentInfo struct containing information about this SwapSource and its inner DFA components
    ///
    /// @return a ComponentInfo struct containing information about this component and a list of ComponentInfo for
    ///     each inner component in the stack.
    access(all) fun getComponentInfo(): DeFiActions.ComponentInfo {
        return DeFiActions.ComponentInfo(
            type: self.getType(),
            id: self.id(),
            innerComponents: [
                self.swapper.getComponentInfo(),
                self.source.getComponentInfo()
            ]
        )
    }
    /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
    /// a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @return a copy of the struct's UniqueIdentifier
    access(contract) view fun copyID(): DeFiActions.UniqueIdentifier? {
        return self.uniqueID
    }
    /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
    /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @param id: the UniqueIdentifier to set for this component
    access(contract) fun setID(_ id: DeFiActions.UniqueIdentifier?) {
        self.uniqueID = id
    }
    /// Returns the type of Vault this Source provides when performing a swap
    ///
    /// @return the type of Vault this Source provides when performing a swap
    access(all) view fun getSourceType(): Type {
        return self.swapper.outType()
    }
    /// Returns the minimum amount of currency available to withdraw from this Source
    ///
    /// @return the minimum amount of currency available to withdraw from this Source
    access(all) fun minimumAvailable(): UFix64 {
        // estimate post-conversion currency based on the source's pre-conversion balance available
        let availableIn = self.source.minimumAvailable()
        return availableIn > 0.0
            ? self.swapper.quoteOut(forProvided: availableIn, reverse: false).outAmount
            : 0.0
    }
    /// Withdraws the provided amount of currency from this Source, swapping the provided amount to the required type if necessary
    ///
    /// @param maxAmount: the maximum amount of currency to withdraw from this Source
    ///
    /// @return the Vault containing the withdrawn currency
    access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
        let minimumAvail = self.minimumAvailable()
        if minimumAvail == 0.0 || maxAmount == 0.0 {
            return <- DeFiActionsUtils.getEmptyVault(self.getSourceType())
        }

        // expect output amount as the lesser between the amount available and the maximum amount
        var amountOut = minimumAvail < maxAmount ? minimumAvail : maxAmount

        // find out how much liquidity to gather from the inner Source
        let availableIn = self.source.minimumAvailable()
        let quote = self.swapper.quoteIn(forDesired: amountOut, reverse: false)
        let quoteIn = availableIn < quote.inAmount ? availableIn : quote.inAmount

        let sourceLiquidity <- self.source.withdrawAvailable(maxAmount: quoteIn)
        if sourceLiquidity.balance == 0.0 {
            Burner.burn(<-sourceLiquidity)
            return <- DeFiActionsUtils.getEmptyVault(self.getSourceType())
        }
        let outVault <- self.swapper.swap(quote: quote, inVault: <-sourceLiquidity)
        if outVault.balance > amountOut {
            // TODO - what to do if excess is found?
            //  - can swapBack() but can't deposit to the inner source and can't return an unsupported Vault type
            //      -> could make inner {Source} an intersection {Source, Sink}
        }
        return <- outVault
    }
}
```
</details>

#### DEX Connectors

Since DFA acts as an abstraction layer above DeFi protocols on Flow across both Cadence and EVM, protocols may be
adapted for use in DFA workflows. Below are two examples - one specific to IncrementFi, the largest Cadence-based DeFi
protocol, and another generically suited for UniswapV2 EVM-based protocols.

> :information_source: The two example Swapper implementations below do not account for slippage, but could be
> configured to do so before production use.

<details>
<summary>IncrementFi Swapper implementation</summary>

```cadence
/// An implementation of DeFiActions.Swapper connector that swaps between tokens using IncrementFi's
/// SwapRouter contract
access(all) struct Swapper : DeFiActions.Swapper {
    /// A swap path as defined by IncrementFi's SwapRouter
    ///  e.g. [A.f8d6e0586b0a20c7.FUSD, A.f8d6e0586b0a20c7.FlowToken, A.f8d6e0586b0a20c7.USDC]
    access(all) let path: [String]
    /// An optional identifier allowing protocols to identify stacked connector operations by defining a protocol-
    /// specific Identifier to associated connectors on construction
    access(contract) var uniqueID: DeFiActions.UniqueIdentifier?
    /// The pre-conversion currency accepted for a swap
    access(self) let inVault: Type
    /// The post-conversion currency returned by a swap
    access(self) let outVault: Type

    init(
        path: [String],
        inVault: Type,
        outVault: Type,
        uniqueID: DeFiActions.UniqueIdentifier?
    ) {
        pre {
            path.length >= 2:
            "Provided path must have a length of at least 2 - provided path has \(path.length) elements"
        }
        IncrementFiConnectors._validateSwapperInitArgs(path: path, inVault: inVault, outVault: outVault)

        self.path = path
        self.inVault = inVault
        self.outVault = outVault
        self.uniqueID = uniqueID
    }

    /// Returns a ComponentInfo struct containing information about this Swapper and its inner DFA components
    ///
    /// @return a ComponentInfo struct containing information about this component and a list of ComponentInfo for
    ///     each inner component in the stack.
    access(all) fun getComponentInfo(): DeFiActions.ComponentInfo {
        return DeFiActions.ComponentInfo(
            type: self.getType(),
            id: self.id(),
            innerComponents: []
        )
    }
    /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
    /// a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @return a copy of the struct's UniqueIdentifier
    access(contract) view fun copyID(): DeFiActions.UniqueIdentifier? {
        return self.uniqueID
    }
    /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
    /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @param id: the UniqueIdentifier to set for this component
    access(contract) fun setID(_ id: DeFiActions.UniqueIdentifier?) {
        self.uniqueID = id
    }
    /// The type of Vault this Swapper accepts when performing a swap
    access(all) view fun inType(): Type {
        return self.inVault
    }
    /// The type of Vault this Swapper provides when performing a swap
    access(all) view fun outType(): Type {
        return self.outVault
    }
    /// The estimated amount required to provide a Vault with the desired output balance
    access(all) fun quoteIn(forDesired: UFix64, reverse: Bool): {DeFiActions.Quote} {
        let amountsIn = SwapRouter.getAmountsIn(amountOut: forDesired, tokenKeyPath: reverse ? self.path.reverse() : self.path)
        return SwapStack.BasicQuote(
            inType: reverse ? self.outType() : self.inType(),
            outType: reverse ? self.inType() : self.outType(),
            inAmount: amountsIn.length == 0 ? 0.0 : amountsIn[0],
            outAmount: forDesired
        )
    }
    /// The estimated amount delivered out for a provided input balance
    access(all) fun quoteOut(forProvided: UFix64, reverse: Bool): {DeFiActions.Quote} {
        let amountsOut = SwapRouter.getAmountsOut(amountIn: forProvided, tokenKeyPath: reverse ? self.path.reverse() : self.path)
        return SwapStack.BasicQuote(
            inType: reverse ? self.outType() : self.inType(),
            outType: reverse ? self.inType() : self.outType(),
            inAmount: forProvided,
            outAmount: amountsOut.length == 0 ? 0.0 : amountsOut[amountsOut.length - 1]
        )
    }
    /// Performs a swap taking a Vault of type inVault, outputting a resulting outVault. Implementations may choose
    /// to swap along a pre-set path or an optimal path of a set of paths or even set of contained Swappers adapted
    /// to use multiple Flow swap protocols.
    access(all) fun swap(quote: {DeFiActions.Quote}?, inVault: @{FungibleToken.Vault}): @{FungibleToken.Vault} {
        let amountOut = self.quoteOut(forProvided: inVault.balance, reverse: false).outAmount
        return <- SwapRouter.swapExactTokensForTokens(
            exactVaultIn: <-inVault,
            amountOutMin: amountOut,
            tokenKeyPath: self.path,
            deadline: getCurrentBlock().timestamp
        )
    }
    /// Performs a swap taking a Vault of type outVault, outputting a resulting inVault. Implementations may choose
    /// to swap along a pre-set path or an optimal path of a set of paths or even set of contained Swappers adapted
    /// to use multiple Flow swap protocols.
    access(all) fun swapBack(quote: {DeFiActions.Quote}?, residual: @{FungibleToken.Vault}): @{FungibleToken.Vault} {
        let amountOut = self.quoteOut(forProvided: residual.balance, reverse: true).outAmount
        return <- SwapRouter.swapExactTokensForTokens(
            exactVaultIn: <-residual,
            amountOutMin: amountOut,
            tokenKeyPath: self.path.reverse(),
            deadline: getCurrentBlock().timestamp
        )
    }
}
```
</details>

<details>

<summary>UniswapV2 Swapper implementation</summary>

```cadence
/// Adapts an EVM-based UniswapV2Router contract's primary functionality to DeFiActions.Swapper connector interface
access(all) struct UniswapV2EVMSwapper : DeFiActions.Swapper {
    /// UniswapV2Router contract's EVM address
    access(all) let routerAddress: EVM.EVMAddress
    /// A swap path defining the route followed for facilitated swaps. Each element should be a valid token address
    /// for which there is a pool available with the previous and subsequent token address via the defined Router
    access(all) let addressPath: [EVM.EVMAddress]
    /// An optional identifier allowing protocols to identify stacked connector operations by defining a protocol-
    /// specific Identifier to associated connectors on construction
    access(contract) var uniqueID: DeFiActions.UniqueIdentifier?
    /// The pre-conversion currency accepted for a swap
    access(self) let inVault: Type
    /// The post-conversion currency returned by a swap
    access(self) let outVault: Type
    /// An authorized Capability on the CadenceOwnedAccount which this Swapper executes swaps on behalf of
    access(self) let coaCapability: Capability<auth(EVM.Owner) &EVM.CadenceOwnedAccount>

    init(
        routerAddress: EVM.EVMAddress,
        path: [EVM.EVMAddress],
        inVault: Type,
        outVault: Type,
        coaCapability: Capability<auth(EVM.Owner) &EVM.CadenceOwnedAccount>,
        uniqueID: DeFiActions.UniqueIdentifier?
    ) {
        pre {
            path.length >= 2: "Provided path with length of \(path.length) - path must contain at least two EVM addresses)"
            FlowEVMBridgeConfig.getTypeAssociated(with: path[0]) == inVault:
            "Provided inVault \(inVault.identifier) is not associated with ERC20 at path[0] \(path[0].toString()) - "
                .concat("Ensure the type & ERC20 contracts are associated via the VM bridge")
            FlowEVMBridgeConfig.getTypeAssociated(with: path[path.length - 1]) == outVault: 
            "Provided outVault \(outVault.identifier) is not associated with ERC20 at path[\(path.length - 1)] \(path[path.length - 1].toString()) - "
                .concat("Ensure the type & ERC20 contracts are associated via the VM bridge")
            coaCapability.check():
            "Provided COA Capability is invalid - provided an active, unrevoked Capability<auth(EVM.Call) &EVM.CadenceOwnedAccount>"
        }
        self.routerAddress = routerAddress
        self.addressPath = path
        self.uniqueID = uniqueID
        self.inVault = inVault
        self.outVault = outVault
        self.coaCapability = coaCapability
    }

    /// Returns a ComponentInfo struct containing information about this Swapper and its inner DFA components
    ///
    /// @return a ComponentInfo struct containing information about this component and a list of ComponentInfo for
    ///     each inner component in the stack.
    access(all) fun getComponentInfo(): DeFiActions.ComponentInfo {
        return DeFiActions.ComponentInfo(
            type: self.getType(),
            id: self.uniqueID?.id,
            innerComponents: []
        )
    }
    /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
    /// a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @return a copy of the struct's UniqueIdentifier
    access(contract) view fun copyID(): DeFiActions.UniqueIdentifier? {
        return self.uniqueID
    }
    /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
    /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @param id: the UniqueIdentifier to set for this component
    access(contract) fun setID(_ id: DeFiActions.UniqueIdentifier?) {
        self.uniqueID = id
    }
    /// The type of Vault this Swapper accepts when performing a swap
    access(all) view fun inType(): Type {
        return self.inVault
    }
    /// The type of Vault this Swapper provides when performing a swap
    access(all) view fun outType(): Type {
        return self.outVault
    }
    /// The estimated amount required to provide a Vault with the desired output balance returned as a BasicQuote
    /// struct containing the in and out Vault types and quoted in and out amounts
    /// NOTE: Cadence only supports decimal precision of 8
    ///
    /// @param forDesired: The amount out desired of the post-conversion currency as a result of the swap
    /// @param reverse: If false, the default inVault -> outVault is used, otherwise, the method estimates a swap
    ///     in the opposite direction, outVault -> inVault
    ///
    /// @return a SwapStack.BasicQuote containing estimate data. In order to prevent upstream reversion,
    ///     result.inAmount and result.outAmount will be 0.0 if an estimate is not available
    access(all) fun quoteIn(forDesired: UFix64, reverse: Bool): {DeFiActions.Quote} {
        let amountIn = self.getAmount(out: false, amount: forDesired, path: reverse ? self.addressPath.reverse() : self.addressPath)
        return SwapStack.BasicQuote(
            inType: reverse ? self.outType() : self.inType(),
            outType: reverse ? self.inType() : self.outType(),
            inAmount: amountIn != nil ? amountIn! : 0.0,
            outAmount: amountIn != nil ? forDesired : 0.0
        )
    }
    /// The estimated amount delivered out for a provided input balance returned as a BasicQuote returned as a
    /// BasicQuote struct containing the in and out Vault types and quoted in and out amounts
    /// NOTE: Cadence only supports decimal precision of 8
    ///
    /// @param forProvided: The amount provided of the relevant pre-conversion currency
    /// @param reverse: If false, the default inVault -> outVault is used, otherwise, the method estimates a swap
    ///     in the opposite direction, outVault -> inVault
    ///
    /// @return a SwapStack.BasicQuote containing estimate data. In order to prevent upstream reversion,
    ///     result.inAmount and result.outAmount will be 0.0 if an estimate is not available
    access(all) fun quoteOut(forProvided: UFix64, reverse: Bool): {DeFiActions.Quote} {
        let amountOut = self.getAmount(out: true, amount: forProvided, path: reverse ? self.addressPath.reverse() : self.addressPath)
        return SwapStack.BasicQuote(
            inType: reverse ? self.outType() : self.inType(),
            outType: reverse ? self.inType() : self.outType(),
            inAmount: amountOut != nil ? forProvided : 0.0,
            outAmount: amountOut != nil ? amountOut! : 0.0
        )
    }
    /// Performs a swap taking a Vault of type inVault, outputting a resulting outVault. This implementation swaps
    /// along a path defined on init routing the swap to the pre-defined UniswapV2Router implementation on Flow EVM.
    /// Any Quote provided defines the amountOutMin value - if none is provided, the current quoted outAmount is
    /// used.
    /// NOTE: Cadence only supports decimal precision of 8
    ///
    /// @param quote: A `DeFiActions.Quote` data structure. If provided, quote.outAmount is used as the minimum amount out
    ///     desired otherwise a new quote is generated from current state
    /// @param inVault: Tokens of type `inVault` to swap for a vault of type `outVault`
    ///
    /// @return a Vault of type `outVault` containing the swapped currency.
    access(all) fun swap(quote: {DeFiActions.Quote}?, inVault: @{FungibleToken.Vault}): @{FungibleToken.Vault} {
        let amountOutMin = quote?.outAmount ?? self.quoteOut(forProvided: inVault.balance, reverse: true).outAmount
        return <-self.swapExactTokensForTokens(exactVaultIn: <-inVault, amountOutMin: amountOutMin, reverse: false)
    }
    /// Performs a swap taking a Vault of type outVault, outputting a resulting inVault. Implementations may choose
    /// to swap along a pre-set path or an optimal path of a set of paths or even set of contained Swappers adapted
    /// to use multiple Flow swap protocols.
    /// Any Quote provided defines the amountOutMin value - if none is provided, the current quoted outAmount is
    /// used.
    /// NOTE: Cadence only supports decimal precision of 8
    ///
    /// @param quote: A `DeFiActions.Quote` data structure. If provided, quote.outAmount is used as the minimum amount out
    ///     desired otherwise a new quote is generated from current state
    /// @param residual: Tokens of type `outVault` to swap back to `inVault`
    ///
    /// @return a Vault of type `inVault` containing the swapped currency.
    access(all) fun swapBack(quote: {DeFiActions.Quote}?, residual: @{FungibleToken.Vault}): @{FungibleToken.Vault} {
        let amountOutMin = quote?.outAmount ?? self.quoteOut(forProvided: residual.balance, reverse: true).outAmount
        return <-self.swapExactTokensForTokens(
            exactVaultIn: <-residual,
            amountOutMin: amountOutMin,
            reverse: true
        )
    }
    /// Port of UniswapV2Router.swapExactTokensForTokens swapping the exact amount provided along the given path,
    /// returning the final output Vault
    ///
    /// @param exactVaultIn: The pre-conversion currency to swap
    /// @param amountOutMin: The minimum amount of post-conversion tokens to swap for
    /// @param reverse: If false, the default inVault -> outVault is used, otherwise, the method swaps in the
    ///     opposite direction, outVault -> inVault
    ///
    /// @return the resulting Vault containing the swapped tokens
    access(self) fun swapExactTokensForTokens(
        exactVaultIn: @{FungibleToken.Vault},
        amountOutMin: UFix64,
        reverse: Bool
    ): @{FungibleToken.Vault} {
        let id = self.uniqueID?.id?.toString() ?? "UNASSIGNED"
        let idType = self.uniqueID?.getType()?.identifier ?? "UNASSIGNED"
        let coa = self.borrowCOA()
            ?? panic("The COA Capability contained by Swapper \(self.getType().identifier) with UniqueIdentifier "
                .concat("\(idType) ID \(id) is invalid - cannot perform an EVM swap without a valid COA Capability"))

        // withdraw FLOW from the COA to cover the VM bridge fee
        let bridgeFeeBalance = EVM.Balance(attoflow: 0)
        bridgeFeeBalance.setFLOW(flow: 2.0 * FlowEVMBridgeUtils.calculateBridgeFee(bytes: 128)) // bridging to EVM then from EVM, hence factor of 2
        let feeVault <- coa.withdraw(balance: bridgeFeeBalance)
        let feeVaultRef = &feeVault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault}

        // bridge the provided to the COA's EVM address
        let inTokenAddress = reverse ? self.addressPath[self.addressPath.length - 1] : self.addressPath[0]
        let evmAmountIn = FlowEVMBridgeUtils.convertCadenceAmountToERC20Amount(
            exactVaultIn.balance,
            erc20Address: inTokenAddress
        )
        coa.depositTokens(vault: <-exactVaultIn, feeProvider: feeVaultRef)

        // approve the router to swap tokens
        var res = self.call(to: inTokenAddress,
            signature: "approve(address,uint256)",
            args: [self.routerAddress, evmAmountIn],
            gasLimit: 15_000_000,
            value: 0,
            dryCall: false
        )!
        if res.status != EVM.Status.successful {
            DeFiActionsEVMConnectors._callError("approve(address,uint256)",
                res, inTokenAddress, idType, id, self.getType())
        }
        // perform the swap
        res = self.call(to: self.routerAddress,
            signature: "swapExactTokensForTokens(uint,uint,address[],address,uint)", // amountIn, amountOutMin, path, to, deadline (timestamp)
            args: [evmAmountIn, UInt256(0), (reverse ? self.addressPath.reverse() : self.addressPath), coa.address(), UInt256(getCurrentBlock().timestamp)],
            gasLimit: 15_000_000,
            value: 0,
            dryCall: false
        )!
        if res.status != EVM.Status.successful {
            // revert because the funds have already been deposited to the COA - a no-op would leave the funds in EVM
            DeFiActionsEVMConnectors._callError("swapExactTokensForTokens(uint,uint,address[],address,uint)",
                res, self.routerAddress, idType, id, self.getType())
        }
        let decoded = EVM.decodeABI(types: [Type<[UInt256]>()], data: res.data)
        let amountsOut = decoded[0] as! [UInt256]

        // withdraw tokens from EVM
        let outVault <- coa.withdrawTokens(type: self.outType(),
                amount: amountsOut[amountsOut.length - 1],
                feeProvider: feeVaultRef
            )

        // clean up the remaining feeVault & return the swap output Vault
        self.handleRemainingFeeVault(<-feeVault)
        return <- outVault
    }

    /* --- Internal --- */

    /// Internal method used to retrieve router.getAmountsIn and .getAmountsOut estimates. The returned array is the
    /// estimate returned from the router where each value is a swapped amount corresponding to the swap along the
    /// provided path.
    ///
    /// @param out: If true, getAmountsOut is called, otherwise getAmountsIn is called
    /// @param amount: The amount in or out. If out is true, the amount will be used as the amount in provided,
    ///     otherwise amount defines the desired amount out for the estimate
    /// @param path: The path of ERC20 token addresses defining the sequence of swaps executed to arrive at the
    ///     desired token out
    ///
    /// @return An estimate of the amounts for each swap along the path. If out is true, the return value contains
    ///     the values in, otherwise the array contains the values out for each swap along the path
    access(self) fun getAmount(out: Bool, amount: UFix64, path: [EVM.EVMAddress]): UFix64? {
        let callRes = self.call(to: self.routerAddress,
            signature: out ? "getAmountsOut(uint,address[])" : "getAmountsIn(uint,address[])",
            args: [amount],
            gasLimit: 5_000_000,
            value: UInt(0),
            dryCall: true
        )
        if callRes == nil || callRes!.status != EVM.Status.successful {
            return nil
        }
        let decoded = EVM.decodeABI(types: [Type<[UInt256]>()], data: callRes!.data) // can revert if the type cannot be decoded
        let uintAmounts: [UInt256] = decoded.length > 0 ? decoded[0] as! [UInt256] : []
        if uintAmounts.length == 0 {
            return nil
        } else if out {
            return FlowEVMBridgeUtils.convertERC20AmountToCadenceAmount(uintAmounts[uintAmounts.length - 1], erc20Address: path[path.length - 1])
        } else {
            return FlowEVMBridgeUtils.convertERC20AmountToCadenceAmount(uintAmounts[0], erc20Address: path[0])
        }
    }
    /// Deposits any remainder in the provided Vault or burns if it it's empty
    access(self) fun handleRemainingFeeVault(_ vault: @FlowToken.Vault) {
        if vault.balance > 0.0 {
            self.borrowCOA()!.deposit(from: <-vault)
        } else {
            Burner.burn(<-vault)
        }
    }
    /// Returns a reference to the Swapper's COA or `nil` if the contained Capability is invalid
    access(self) view fun borrowCOA(): auth(EVM.Owner) &EVM.CadenceOwnedAccount? {
        return self.coaCapability.borrow()
    }
    /// Makes a call to the Swapper's routerEVMAddress via the contained COA Capability with the provided signature,
    /// args, and value. If flagged as dryCall, the more efficient and non-mutating COA.dryCall is used. A result is
    /// returned as long as the COA Capability is valid, otherwise `nil` is returned.
    access(self) fun call(
        to: EVM.EVMAddress,
        signature: String,
        args: [AnyStruct],
        gasLimit: UInt64,
        value: UInt,
        dryCall: Bool
    ): EVM.Result? {
        let calldata = EVM.encodeABIWithSignature(signature, args)
        let valueBalance = EVM.Balance(attoflow: value)
        if let coa = self.borrowCOA() {
            let res: EVM.Result = dryCall
                ? coa.dryCall(to: to, data: calldata, gasLimit: gasLimit, value: valueBalance)
                : coa.call(to: to, data: calldata, gasLimit: gasLimit, value: valueBalance)
            return res
        }
        return nil
    }
}
```
</details>

#### Flash Loan Connectors

Since a flash loan must be executed atomically, protocols often include generic calldata and callback patterns to ensure
the loan is repaid in full plus a fee within their contract call scope. The Flasher interface design allows for that
callback to be defined in either contract or transaction context.

In the example Flasher below, an externally defined function can be passed into the the `Flasher.flashloan()` method,
but other implementations may also decide to pass in contract-defined methods conditionally directing flashloan
callbacks to pre-defined contract logic depending on the conditions and optional parameters.

<details>

<summary>IncrementFi Flasher implementation</summary>

```cadence
access(all) struct Flasher : SwapInterfaces.FlashLoanExecutor, DeFiActions.Flasher {
    /// The address of the SwapPair contract to use for flash loans
    access(all) let pairAddress: Address
    /// The type of token to borrow
    access(all) let type: Type
    /// An optional identifier allowing protocols to identify stacked connector operations by defining a protocol-
    /// specific Identifier to associated connectors on construction
    access(contract) var uniqueID: DeFiActions.UniqueIdentifier?

    init(pairAddress: Address, type: Type, uniqueID: DeFiActions.UniqueIdentifier?) {
        let pair = getAccount(pairAddress).capabilities.borrow<&{SwapInterfaces.PairPublic}>(SwapConfig.PairPublicPath)
            ?? panic("Could not reference SwapPair public capability at address \(pairAddress)")
        let pairInfo = pair.getPairInfoStruct()
        assert(pairInfo.token0Key == type.identifier || pairInfo.token1Key == type.identifier,
            message: "Provided type is not supported by the SwapPair at address \(pairAddress) - "
                .concat("valid types for this SwapPair are \(pairInfo.token0Key) and \(pairInfo.token1Key)"))
        self.pairAddress = pairAddress
        self.type = type
        self.uniqueID = uniqueID
    }

    /// Returns a ComponentInfo struct containing information about this Flasher and its inner DFA components
    ///
    /// @return a ComponentInfo struct containing information about this component and a list of ComponentInfo for
    ///     each inner component in the stack.
    access(all) fun getComponentInfo(): DeFiActions.ComponentInfo {
        return DeFiActions.ComponentInfo(
            type: self.getType(),
            id: self.id(),
            innerComponents: []
        )
    }
    /// Returns a copy of the struct's UniqueIdentifier, used in extending a stack to identify another connector in
    /// a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @return a copy of the struct's UniqueIdentifier
    access(contract) view fun copyID(): DeFiActions.UniqueIdentifier? {
        return self.uniqueID
    }
    /// Sets the UniqueIdentifier of this component to the provided UniqueIdentifier, used in extending a stack to
    /// identify another connector in a DeFiActions stack. See DeFiActions.align() for more information.
    ///
    /// @param id: the UniqueIdentifier to set for this component
    (contract) fun setID(_ id: DeFiActions.UniqueIdentifier?) {
        self.uniqueID = id
    }
    /// Returns the asset type this Flasher can issue as a flash loan
    ///
    /// @return the type of token this Flasher can issue as a flash loan
    (all) view fun borrowType(): Type {
        return self.type
    }
    /// Returns the estimated fee for a flash loan of the specified amount
    ///
    /// @param loanAmount: The amount of tokens to borrow
    /// @return the estimated fee for a flash loan of the specified amount
    (all) fun calculateFee(loanAmount: UFix64): UFix64 {
        return UFix64(SwapFactory.getFlashloanRateBps()) * loanAmount / 10000.0
    }
    /// Performs a flash loan of the specified amount. The callback function is passed the fee amount, a Vault
    /// containing the loan, and the data. The callback function should return a Vault containing the loan + fee.
    ///
    /// @param amount: The amount of tokens to borrow
    /// @param data: Optional data to pass to the callback function
    /// @param callback: The callback function to use for the flash loan
    (all) fun flashLoan(
        amount: UFix64,
        data: AnyStruct?,
        callback: fun(UFix64, @{FungibleToken.Vault}, AnyStruct?): @{FungibleToken.Vault} // fee, loan, data
    ) {
        // get the SwapPair public capability on which to perform the flash loan
        let pair = getAccount(self.pairAddress).capabilities.borrow<&{SwapInterfaces.PairPublic}>(
                SwapConfig.PairPublicPath
            ) ?? panic("Could not reference SwapPair public capability at address \(self.pairAddress)")

        // cast data to expected params type and add fee and callback to params for the callback function
        let params = data as! {String: AnyStruct}? ?? {}
        params["fee"] = self.calculateFee(loanAmount: amount)
        params["callback"] = callback

        // perform the flash loan
        pair.flashloan(
            executor: &self as &{SwapInterfaces.FlashLoanExecutor},
            requestedTokenVaultType: self.type,
            requestedAmount: amount,
            params: params
        )
    }
    /// Performs a flash loan of the specified amount. The executor function is passed the fee amount and a Vault
    /// containing the loan. The executor function should return a Vault containing the loan and fee.
    access(all) fun executeAndRepay(loanedToken: @{FungibleToken.Vault}, params: {String: AnyStruct}): @{FungibleToken.Vault} {
        let fee = params["fee"] as! UFix64
        let executor = params["callback"] as! fun(UFix64, @{FungibleToken.Vault}): @{FungibleToken.Vault}
        let repaidToken <- executor(fee, <-loanedToken)
        return <- repaidToken
    }
}
```
</details>

### AutoBalancer Component

The AutoBalancer is a sophisticated component that demonstrates advanced DFA composition, leveraging a PriceOracle and
optional rebalance Sink and/or Source. An AutoBalancer's `rebalance()` method is designed for use with Scheduled
Callbacks, allowing for automated management of the contained balance.

> :information_source: Since the AutoBalancer is intended to leverage scheduled callbacks for `rebalance()` execution,
> the method's final implementation and the resource's conformance set is contingent on the production interface for the
> scheduled callbacks feature.

```cadence
access(all) resource AutoBalancer : IdentifiableResource, FungibleToken.Receiver, FungibleToken.Provider, ViewResolver.Resolver, Burner.Burnable {
    /// The value in deposits & withdrawals over time denominated in oracle.unitOfAccount()
    access(self) var _valueOfDeposits: UFix64
    /// The percentage low and high thresholds defining when a rebalance executes
    /// Index 0 is low, index 1 is high
    access(self) var _rebalanceRange: [UFix64; 2]
    /// Oracle used to track the baseValue for deposits & withdrawals over time
    access(self) let _oracle: {PriceOracle}
    /// The inner Vault's Type captured for the ResourceDestroyed event
    access(self) let _vaultType: Type
    /// Vault used to deposit & withdraw from made optional only so the Vault can be burned via Burner.burn() if the
    /// AutoBalancer is burned and the Vault's burnCallback() can be called in the process
    access(self) var _vault: @{FungibleToken.Vault}?
    /// An optional Sink used to deposit excess funds from the inner Vault once the converted value exceeds the
    /// rebalance range. This Sink may be used to compound yield into a position or direct excess value to an
    /// external Vault
    access(self) var _rebalanceSink: {Sink}?
    /// An optional Source used to deposit excess funds to the inner Vault once the converted value is below the
    /// rebalance range
    access(self) var _rebalanceSource: {Source}?
    /// Capability on this AutoBalancer instance
    access(self) var _selfCap: Capability<auth(FungibleToken.Withdraw) &AutoBalancer>?
    /// An optional UniqueIdentifier tying this AutoBalancer to a given stack
    access(contract) var uniqueID: UniqueIdentifier?

    // ...
    
    /// Allows for external parties to call on the AutoBalancer and execute a rebalance according to it's rebalance
    /// parameters. This method must be called by external party regularly in order for rebalancing to occur.
    ///
    /// @param force: if false, rebalance will occur only when beyond upper or lower thresholds; if true, rebalance
    ///     will execute as long as a price is available via the oracle and the current value is non-zero
    (Auto) fun rebalance(force: Bool) {
        let currentPrice = self._oracle.price(ofToken: self._vaultType)
        if currentPrice == nil {
            return // no price available -> do nothing
        }
        let currentValue = self.currentValue()!
        // calculate the difference between the current value and the historical value of deposits
        var valueDiff: UFix64 = currentValue < self._valueOfDeposits ? self._valueOfDeposits - currentValue : currentValue - self._valueOfDeposits
        // if deficit detected, choose lower threshold, otherwise choose upper threshold
        let isDeficit = currentValue < self._valueOfDeposits
        let threshold = isDeficit ? (1.0 - self._rebalanceRange[0]) : (self._rebalanceRange[1] - 1.0)

        if currentPrice == 0.0 || valueDiff == 0.0 || ((valueDiff / self._valueOfDeposits) < threshold && !force) {
            // division by zero, no difference, or difference does not exceed rebalance ratio & not forced -> no-op
            return
        }

        let vault = self._borrowVault()
        var amount = valueDiff / currentPrice!
        var executed = false
        let maybeRebalanceSource = &self._rebalanceSource as auth(FungibleToken.Withdraw) &{Source}?
        let maybeRebalanceSink = &self._rebalanceSink as &{Sink}?
        if isDeficit && maybeRebalanceSource != nil {
            // rebalance back up to baseline sourcing funds from _rebalanceSource
            vault.deposit(from:  <- maybeRebalanceSource!.withdrawAvailable(maxAmount: amount))
            executed = true
        } else if !isDeficit && maybeRebalanceSink != nil {
            // rebalance back down to baseline depositing excess to _rebalanceSink
            if amount > vault.balance {
                amount = vault.balance // protect underflow
            }
            let surplus <- vault.withdraw(amount: amount)
            maybeRebalanceSink!.depositCapacity(from: &surplus as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
            executed = true
            if surplus.balance == 0.0 {
                Burner.burn(<-surplus) // could destroy
            } else {
                amount = amount - surplus.balance // update the rebalanced amount
                valueDiff = valueDiff - (surplus.balance * currentPrice!) // update the value difference
                vault.deposit(from: <-surplus) // deposit any excess not taken by the Sink
            }
        }
        // emit event only if rebalance was executed
        if executed {
            emit Rebalanced(
                amount: amount,
                value: valueDiff,
                unitOfAccount: self.unitOfAccount().identifier,
                isSurplus: !isDeficit,
                vaultType: self.vaultType().identifier,
                vaultUUID: self._borrowVault().uuid,
                balancerUUID: self.uuid,
                address: self.owner?.address,
                uuid: self.uuid,
                uniqueID: self.id()
            )
        }
    }

    // ...
}
```

### Event System

DFA components emit standardized events for operation tracing:

```cadence
/// Emitted when value is deposited to a Sink
access(all) event Deposited(
    type: String,
    amount: UFix64,
    fromUUID: UInt64,
    uniqueID: UInt64?,
    sinkType: String
)
/// Emitted when value is withdrawn from a Source
access(all) event Withdrawn(
    type: String,
    amount: UFix64,
    withdrawnUUID: UInt64,
    uniqueID: UInt64?,
    sourceType: String
)
/// Emitted when a Swapper executes a Swap
access(all) event Swapped(
    inVault: String,
    outVault: String,
    inAmount: UFix64,
    outAmount: UFix64,
    inUUID: UInt64,
    outUUID: UInt64,
    uniqueID: UInt64?,
    swapperType: String
)
/// Emitted when a Flasher executes a flash loan
access(all) event Flashed(
    requestedAmount: UFix64,
    borrowType: String,
    uniqueID: UInt64?,
    flasherType: String
)
/// Emitted when an IdentifiableResource's UniqueIdentifier is aligned with another DFA component
access(all) event UpdatedID(
    oldID: UInt64?,
    newID: UInt64?,
    component: String,
    uuid: UInt64?
)
/// Emitted when an AutoBalancer is created
access(all) event CreatedAutoBalancer(
    lowerThreshold: UFix64,
    upperThreshold: UFix64,
    vaultType: String,
    vaultUUID: UInt64,
    uuid: UInt64,
    uniqueID: UInt64?
)
/// Emitted when AutoBalancer.rebalance() is called
access(all) event Rebalanced(
    amount: UFix64,
    value: UFix64,
    unitOfAccount: String,
    isSurplus: Bool,
    vaultType: String,
    vaultUUID: UInt64,
    balancerUUID: UInt64,
    address: Address?,
    uuid: UInt64,
    uniqueID: UInt64?
)
```

Component actions are associated by their `uniqueID` event values. Since `UniqueIdentifier` structs are issued by the
contract and the interfaces limit external access, it can be assumed that DFA events sharing the same `uniqueID` relate
to the same workflow stack. For instance, in the case of a SwapSink where the outer SwapSink, inner Swapper and inner
Sink all share the same `uniqueID`, the `Deposited.uniqueID` and `Swapped.uniqueID` and final `Deposited.uniqueID`
denote that all DFA interface events relate to the same stack operation.

## Use Cases

### Automated Token Transmission

This example `Shuttle` object can be used to simply move tokens. The workflow executed depends on the DFA connectors
configured on the `Shuttle`'s initialization. Using the same prototype object below, several immediate configuration
options exist to execute different DeFi workflow: 

1. A VaultSink would simply receive the deposited tokens, transferring from the Shuttle's Source to the Sink.
2. Configured with a SwapSink, the deposited tokens would be swapped before depositing to an inner SwapSink's inner
   Sink.
3. Provided a Source tied to staking rewards and a Sink that stakes deposited tokens, this object could be used to
   optimize staking rewards by auto-claiming & re-staking.

> :information_source: This example is included to demonstrate the modularity of DFA architecture and not as an
> accompanying standard object in its own right.

<detail>

<summary>TokenShuttleService example</summary>

```cadence
/// Example auto token transmission using DFA connectors for use in DCA strategies, onchain subscriptions, and staking 
/// rewards claim + restake
///
/// Usage of Scheduled Callbacks based on current state of FLIP #331
access(all) contract TokenShuttleService {

    access(all) entitlement Set
    access(all) entitlement Schedule

    /// Moves tokens from a source to a target in conformance with CallbackHandler
    access(all) resource Shuttle : UnsafeCallbackScheduler.CallbackHandler {
        /// Provides tokens to move - composite connector so any excess withdrawals can be re-deposited
        access(self) let tokenOrigin: {DeFiActions.Sink, DeFiActions.Source}
        /// Sink to which source tokens are deposited - a SwapSink would swap into a target denomination
        access(self) let tokenDestination: {DeFiActions.Sink}
        /// Provides FLOW to pay for scheduled callbacks
        access(self) let callbackFeeSource: {DeFiActions.Source}
        /// The amount of tokens to withdraw from tokenOrigin when executed. If `nil`, transmission amount is whatever
        /// the tokenOrigin reports as available. Note that this is the amount of tokens withdrawn, not a dollar value.
        /// If this were taken to production, it might be considered to include a PriceOracle to ensure a dollar value
        /// is transferred on execution instead of a token amount.
        access(self) let maxAmount: UFix64?
        /// When the strategy was last executed
        access(self) let lastExecuted: UFix64
        /// The amount of time to elapse before auto-executing
        access(self) var interval: UFix64
        /// An authorized Capability enabling scheduled callbacks
        access(self) let selfCapability: Capability<auth(UnsafeCallbackScheduler.mayExecuteCallback) &Shuttle>
        /// The next scheduled callback - production would deal with more intelligently
        access(self) var pendingCallback: UnsafeCallbackScheduler.ScheduledCallback?

        // init( ... ) { ... }

        /// Calculates the timestamp of when the next execution should be scheduled
        access(all) view fun getNextTransitTime(): UFix64 {
            let now = getCurrentBlock().timestamp
            let slated = self.lastExecuted + self.interval
            return slated <= now ? now : slated
        }
        /// Sets the Capability of the Strategy on itself so it can be self-managed via Scheduled Callbacks
        access(Set)
        fun setSelfCapability(_ cap: Capability<auth(UnsafeCallbackScheduler.mayExecuteCallback) &Shuttle>) {
            // pre { ... }
            self.selfCapability = cap
        }
        /// CallbackHandler conformance - enables execution of Scheduled Callbacks
        access(UnsafeCallbackScheduler.mayExecuteCallback) fun executeCallback(data: AnyStruct?) {
            let now = getCurrentBlock().timestamp
            if now < self.lastExecuted + self.interval {
                return self.scheduleNextRun() // not time for execution - schedule the next callback
            }
            self.lastExecuted = now

            // withdraw tokens from source
            let sourceVault <- self.tokenOrigin.withdrawAvailable(maxAmount: self._getTransmissionAmount())
            if sourceVault.balance == 0.0 {
                return self.scheduleNextRun()
            }
            
            // deposit to inner sink
            let sourceVaultRef = &sourceVault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault}
            self.tokenDestination.depositCapacity(from: sourceVaultRef)
            self._handleRemainingVault(<-sourceVault)
            
            // schedule next transmission before closing
            self.scheduleNextRun()
        }
        /// Schedules the next scheduled callback
        access(Schedule) fun scheduleNextRun() {
            let prio = UnsafeCallbackScheduler.Priority.Medium
            let effort = 2000 // could be configured - varies based on complexity of source & sink
            let timestamp = self.getNextTransitTime()
            let estimate = UnsafeCallbackScheduler.estimate(
                    data: nil,
                    timestamp: timestamp,
                    priority: prio,
                    executionEffort: effort,
                )!
            let fees <- self.callbackFeeSource.withdrawAvailable(maxAmount: estimate.flowFee)
            let scheduledCallback = UnsafeCallbackScheduler.schedule(
                callback: self.selfCapability,
                data: nil,
                timestamp: timestamp,
                priority: prio,
                executionEffort: effort,
                fees: <-feesVault
            )
            self.pendingCallback = scheduledCallback
        }

        /* INTERNAL */
        //
        /// Calculates the amount to transmit on execution based on the 
        access(self) view fun _getTransmissionAmount(): UFix64 {
            let capacity = self.tokenDestination.minimumCapacity()
            var amount = self.tokenOrigin.minimumAvailable()
            if self.maxAmount != nil {
                amount = amount <= self.maxAmount! ? amount : self.maxAmount! 
            }
            amount = amount <= capacity ? amount : capacity
            return amount
        }
        /// Handles any remaining vault that was withdrawn in excess of what could be handled by the Sink
        access(self) fun _handleRemainingVault(_ remainder: @{FungibleToken.Vault}) {
            if sourceVault.balance > 0.0 {
                self.tokenOrigin.depositCapacity(from: sourceVaultRef)
            }
            assert(sourceVault.balance == 0.0, message: "Could not handle remaining withdrawal amount \(sourceVault.balance)")
            Burner.burn(<-sourceVault)
        }
    }
}
```

</detail>

To drive the example home, below is a transaction that would configure the `Shuttle` for a DCA strategy:

<detail>

<summary>Shuttle DCA configuration</summary>

```cadence
import "FungibleToken"
import "FlowToken"

import "DeFiActions"
import "FungibleTokenStack"
import "IncrementFiConnectors"

import "TokenShuttleService"

/// Example transaction configuring a TokenShuttleService Shuttle to execute a dollar cost averaging DCA strategy via
/// IncrementFi's swap protocol, executing swaps at preset intervals via scheduled callbacks
///
/// @param originStoragePath: the storage path for the Vault providing dca funds
/// @param destinationPublicPath: the public path for the Vault capability where the swapped funds will be directed
/// @param maxAmount: the max amount of tokens to with draw from the origin Vault on interval execution. If `nil`, the
///     full available balance of the Vault will be transferred
/// @param interval: approximate amount of seconds to lapse between scheduled callback executions
/// @param incrementSwapPath: the token path along which to execute the swap. The format should follow IncrementFi's
///     protocol format for defining swap paths - e.g. [A.f8d6e0586b0a20c7.USDCf, A.f8d6e0586b0a20c7.FlowToken]
/// @param shuttleStoragePath: the storage path at which to store the Shuttle
///
transaction(
    originStoragePath: StoragePath,
    destinationPublicPath: PublicPath,
    maxAmount: UFix64?,
    interval: UFix64,
    incrementSwapPath: [String]
    shuttleStoragePath: StoragePath
) {

    let shuttle: auth(TokenShuttleService.Schedule) &TokenShuttleService.Shuttle

    prepare(signer: auth(Storage, Capabilities) &Account) {
        if signer.storage.type(at: shuttleStoragePath) != nil {
            panic("Collision at Shuttle storage path \(shuttleStoragePath)")
        }

        // init the UniqueIdentifier shared across all connectors
        let id = DeFiActions.createUniqueIdentifier()

        // configure the FLOW VaultSource which pays for scheduled callback fees
        let flowProvider = signer.capabilities.storage.issue<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(/storage/flowTokenVault)
        assert(flowProvider.check(), message: "Invalid Provider Capability issued targeting storage path \(/storage/flowTokenVault)")
        let flowSource = FungibleTokenStack.VaultSource(
                min: nil,
                withdrawVault: flowProvider,
                uniqueID: id
            )
        
        // configure the VaultSinkAndSource which provides tokens to swap
        let originProvider = signer.capabilities.storage.issue<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(originStoragePath)
        assert(originProvider.check(), message: "Invalid Provider Capability issued targeting storage path \(originStoragePath)")
        let vaultSinkSource = FungibleTokenStack.VaultSinkSource(
                min: nil,
                max: nil,
                vault: originProvider,
                uniqueID: id
            )

        // configure the VaultSink which receives the swapped tokens
        let destinationReceiver = signer.capabilities.get<&{FungibleToken.Vault}>(destinationPublicPath)
        assert(destinationReceiver.check(), message: "Invalid Receiver Capability retrieved from public path \(destinationPublicPath)")
        let vaultSink = FungibleTokenStack.VaultSink(
                max: nil,
                depositVault: destinationReceiver,
                uniqueID: id
            )
        
        // configure the Swapper
        let swapper = IncrementFiConnectors.Swapper(
            path: incrementSwapPath,
            inVault: originProvider.borrow()!.getType(),
            outVault: destinationReceiver.borrow()!.getType(),
            uniqueID: id
        )

        // configure a SwapSink with the Swapper and VaultSink
        let swapSink = SwapStack.SwapSink(
                swapper: swapper,
                sink: vaultSink
                uniqueID: id
            )
        
        // configure the Shuttle in storage, putting together all the DFA components
        let newShuttle <- TokenShuttleService.createShuttle(
                tokenOrigin: vaultSinkSource,
                tokenDestination: swapSink,
                callbackFeeSource: flowSource,
                maxAmount: maxAmount,
                interval: interval
            )
        self.signer.storage.save(<-newShuttle, to: shuttleStoragePath)

        // set the self capability so scheduled callbacks can execute
        let selfCap = signer.capabilities.storage.issue<auth(UnsafeCallbackScheduler.mayExecuteCallback) &Shuttle>(shuttleStoragePath)
        self.shuttle.setSelfCapability()
    }

    execute {
        // schedule the next run to execute the next scheduled callback
        self.shuttle.scheduleNextRun()
    }
}
```
</detail>

Generally speaking, the transaction above demonstrates how one deals with DFA connectors in a sort of backwards
approach. We start with the most deeply nested connectors, then put them all together in the surface-level component,
tying together the workflow stack.

## Considerations

### Security Model

DFA's design philosophy prioritizes graceful failure over strict guarantees, which creates both benefits and security considerations:

**Weak Behavioral Guarantees:**
- Sources may return less than requested without reverting
- Sinks may accept less than provided without warning

**Security Implications:**
- **Consumer Responsibility**: Applications must validate component outputs rather than assuming behavior
- **Asset Tracing**: Multiple connectors as would be common for complex workflow stack can make asset tracking complex, particular in cross-VM context
- **Reentrancy Risk**: Open component definitions and weak guarantees may increase reentrancy attack surface
- **Flash Loan Risks**: Flasher implementations must validate full repayment (loan + fee) to prevent exploitation; consumer logic must execute entirely within the callback function scope

**Recommended Practices:**
- Always validate returned Vault types and amounts
- Implement slippage protection for Swapper operations
- Use UniqueIdentifier for comprehensive operation tracing
- Consider implementing circuit breakers for automated strategies
- For Flasher implementations: validate exact repayment amounts before transaction completion
- For flash loan consumers: ensure all loan-related logic executes within the callback function

### Performance Implications

**Computational Complexity:**
- **Aggregation Overhead**: MultiSwapper components querying many Swappers can be expensive
- **Deep Call Stacks**: Complex compositions may hit computation limits
- **Event Volume**: DFA stacks emit extensive events, increasing transaction costs
- **Cross-VM Operations**: Future EVM bridge connectors will add computational overhead

**Optimization Strategies:**
- Limit aggregation scope for performance-critical applications
- Cache price quotes when possible to reduce oracle calls
- Monitor stack depth in complex compositions
- Consider gas costs when designing autonomous strategies

### Testing Challenges

**Environment Complexity:**
- Multi-protocol dependencies require complex test environment setup
- Protocol state dependencies make unit testing difficult
- Limited testnet protocol maintenance affects testing reliability

**Recommendations:**
- Develop protocol-specific testing frameworks and starter repositories
- Assess feasibility of improving testnet infrastructure
- Explore Mainnet fork onto local emulator for realistic integration testing
- Deploy an experiment DeFiActions contract to Mainnet to allow developers to build in a live environment with reduced hurdles

### Drawbacks

**Learning Curve:**
- Component composition patterns require new approaches, often working backwards from a desired end state
- Debugging complex stacks can be challenging as the stacks can be quite deep
- Performance characteristics are not yet well-understood

**Protocol Maturity:**
- Standards are new and may evolve based on developer feedback
- Limited ecosystem tooling and documentation
- Few connectors at the start may mean early adopters may need to implement their own connectors
- Building on scheduled callbacks prior to real-world implementation feedback could introduce unknown risks

### Interface Concerns

**Single Asset Type**
- Current [Sink](#sink-interface) and [Source](#source-interface) interfaces only support single Vault operations, which may not accommodate multi-asset actions such as liquidity provision or removal that require handling multiple Vaults simultaneously.
- The [Swapper](#swapper-interface) interface currently requires slippage to be set at initialization, limiting per-trade flexibility.

**Recomendations:**
- Consider whether to introduce a multi-asset Sink/Source interface, extend existing interfaces to accept multiple Vaults, or define a new connector type for these scenarios.
- Clarify expected behavior when slippage exceeds the configured threshold, especially in the context of graceful failure requirements.

## Compatibility

DeFiActions is a new standard that maintains full compatibility with existing Flow infrastructure. The standard exists as a layer of abstraction above new and existing infrastructure, so there should be no concerns of breaking changes. Any realized incompatibility should be reported as feedback on this FLIP and considered in the component interface design.

**No Migration Required:**
- Existing DeFi protocols continue operating unchanged
- Current applications can gradually adopt DFA components
- No modifications needed to deployed contracts

**Integration Approach:**  
- Protocols can add DFA support via integrating connectors
- Applications may mix DFA components with direct protocol calls
- Developers can choose adoption level based on use case complexity

**Forward Compatibility:**
- Breaking changes will be announced well in advance during beta period
- Production release will commit to backwards compatibility standards

## Future Extensions

**Planned Enhancements:**
- **FLOW-EVM Bridge Connectors**: Support for Flow EVM DeFi protocols
- **Staking Rewards Connectors**: Integrate Source & Sink for protocol staking & claiming rewards

**Ecosystem Development:**
- **Protocol Partner Program**: Collaboration framework for connector development
- **Developer Tooling**: Protocol integration scaffolds, component mocks, and development tooling
- **Educational Resources**: Comprehensive guides, tutorials, and best practices
- **Community Components**: Registry of community-developed connectors and patterns

**Research Areas:**
- **Agent-Assisted Development**: AI-driven component usage and transaction generation
- **Visual Composition Tools**: GUI-based component composition for non-technical users
