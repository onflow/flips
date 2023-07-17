---
status: proposed 
flip: NNN (do not set)
authors: Satyam Agrawal (satyam.agrawal@dapperlabs.com)
sponsor: Jonny (Increment.Fi) (hsyjonny@gmail.com) 
updated: 2023-06-09 
---

# Pool-Based DEX Swap Standard

## Objective

The Flow ecosystem is experiencing continued growth and with this expansion comes a need to expand the DeFi ecosystem on the Flow blockchain. Expansion of DeFi on Flow requires new liquidity - typically through cross-chain bridges - as well as straightforward integration for wallets and developers to provide a simple and intuitive user experience. While the macro topic of liquidity is beyond the scope of this discussion, the initial focus of the nascent standard aims to enable ease of interoperability with, and composability of, DEXs which are a cornerstone concept serving as gateways to any DeFi ecosystem. This FLIP is focused on the pool based DEXs like Uniswap v2 style AMMs.

## Motivation

As the FLOW ecosystem continues to expand, the incorporation of Decentralized Finance (DeFi) becomes increasingly essential. Serving as a portal, DeFi facilitates the introduction of a multitude of associated projects, thereby widening the horizon of customer engagement and services provided to FLOW blockchain users. Central to this vision is the role of decentralized exchanges, which form the bedrock upon which the FLOW DeFi ecosystem will be constructed. To strengthen this foundation, the standardization of decentralized exchanges is imperative. Standardization fosters an environment of openness and permissionless innovation, ensuring optimal composability of protocols developed on the FLOW blockchain. Consequently, it sets the stage for a robust and dynamic DeFi ecosystem within the FLOW framework, opening a new chapter of possibilities and opportunities.

Standardization of Decentralized Exchange (DEX) Interfaces is a crucial aspect of the broader DeFi (Decentralized Finance) ecosystem due to several compelling reasons. Primarily, it fosters interoperability, enhancing the capacity of various applications, wallets, and protocols to seamlessly interact with each other, thereby promoting a more cohesive and efficient DeFi environment. Secondly, standardized interfaces mitigate the risk of user errors that can stem from dealing with disparate systems, thereby improving overall user experience and encouraging wider adoption. Moreover, standardization aids in creating a transparent, competitive marketplace, as it levels the playing field by ensuring that all DEXs adhere to the same functional and operational parameters. Finally, a standardized interface could expedite the development of new tools and services, as developers would have a common framework to build upon, leading to more innovation within the DeFi ecosystem.

Several Decentralized Exchanges (DEXs) exist within the FLOW ecosystem, including but not limited to [Increment.Fi](https://increment.fi/), [BloctoSwap](https://swap.blocto.app/#/), and [Metapier](https://metapier.org/). However, each DEX presents a unique interface which may instigate friction during the development of composable products on FLOW. To address these complications, this FLIP aims to propose a standardized interface, designed to mitigate such discrepancies and streamline the creation process in the ecosystem.

## User Benefit

This FLIP primarily benefits developers aiming to integrate DEXs and construct composable financial products within the FLOW blockchain. The proposal's fundamental advantage lies in establishing a unified interface, alleviating developers' concerns regarding the nuances of individual DEX interfaces. This streamlined approach ensures that the integration process becomes more efficient, promoting a plug-and-play model for developers. Consequently, even when new DEXs are introduced into the ecosystem, they will be immediately compatible and supportable, courtesy of the standardized interface. This forward-looking design promises to facilitate continual innovation, bolstering the FLOW ecosystem's growth and dynamism.

## Design Proposal

The crux of this proposal revolves around the introduction of two distinct resource interfaces. The first, `ImmediateSwap`, is dedicated to facilitating direct trades between two tokens. The second, `ImmediateSwapQuotation`, is designed to retrieve the current price data for a particular token pair. In accordance with this proposal, these two resource interfaces are expected to be adhered to within the resource implementation carried out by various DEXs. This standardization ensures a consistent, interoperable approach to both trading and price quotation across different decentralized exchanges within the ecosystem.

### Interface for Swapping.

Design decision of `ImmediateSwap` resource interface revolves around the following requirements - 

- As a user of a Decentralized Exchange (DEX), Alice possesses the ability to effortlessly swap Token A for Token B, enhancing her capacity to manage her digital assets effectively.
- Alice, utilizing the DEX platform, is able to conduct swift asset swaps where the required funds are provisioned through a vault, streamlining the process of asset exchange.
- To safeguard her trades from substantial slippage, Alice can stipulate a minimum expected amount post-swap, reinforcing her control over the transaction outcomes.
- Alice's trade, while facilitated by the DEX, is not deemed valid indefinitely. It is subject to a predefined expiration period, ensuring trades are time-bound and promoting transaction integrity within the DEX system.

In pursuit of above objectives, this proposal introduces two straightforward functions, namely, `swapExactSourceToTargetTokenUsingPath` and `swapExactSourceToTargetTokenUsingPathAndReturn`. Each function facilitates the exchange of the source token for the target token, with the choice between the two being contingent upon the specific use-case scenario. While both functions serve the primary purpose of token swapping, they offer distinct advantages depending on the operational context. The nuanced differences and their implications are elaborated further in the interface definition, ensuring a comprehensive understanding of their functional dichotomy.

The functions proposed in this document leverage an off-chain computed optimized path array. This methodology is adopted due to the limited likelihood of possessing a direct pool between a given pair of tokens. In order to execute a successful swap, a path from the source token to the target token is required, wherein all adjacent pairs within the path represent potential pool pairs.

Computing such an optimized path on-chain would not be feasible due to the potentially extensive number of variations, making it impractical and inefficient in terms of gas usage. Therefore, it falls upon the transaction initiator to provide a valid path to ensure the successful completion of the transaction. This approach not only optimizes the use of resources but also maintains the robustness and reliability of the transaction process within the ecosystem.

```cadence

/// Resource that get returned after the `swapExactSourceToTargetTokenUsingPathAndReturn` function execution.
pub resource interface ExactSwapAndReturnValue {
    /// It represents the Vault that holds target token and would be returned
    /// after a swap. 
    pub let targetTokenVault: @FungibleToken.Vault
    /// It is an optional vault that holds the leftover source tokens after a swap.
    pub let remainingSourceTokenVault: @FungibleToken.Vault?
}

pub resource interface ImmediateSwap {

    /// @notice It will Swap the source token for to target token
    ///
    /// If the user wants to swap USDC to FLOW then the
    /// sourceToTargetTokenPath is [Type<USDC>, Type<FLOW>] and
    /// USDC would be the source token
    ///
    /// Necessary constraints
    /// - For the given source vault balance, Swapped target token amount should be
    ///   greater than or equal to `exactTargetAmount`, otherwise swap would fail.
    /// - If the swap settlement time i.e getCurrentBlock().timestamp is less than or
    ///   equal to the provided expiry then the swap would fail.
    /// - Provided `recipient` capability should be valid otherwise the swap would fail.
    /// - If the provided path doesn’t exists then the swap would fail.
    ///
    /// @param sourceToTargetTokenPath: Off-chain computed path for reaching source token to target token
    ///                                 `sourceToTargetTokenPath[0]` should be the source token type while
    ///                                 `sourceToTargetTokenPath[sourceToTargetTokenPath.length - 1]` should be the target token
    ///                                 and all the remaining intermediaries token types would be necessary swap hops to swap the
    ///                                 source token with target token.
    /// @param sourceVault:             Vault that holds the source token.
    /// @param exactTargetAmount:       Exact amount expected from the swap, If swapped amount is less than `exactTargetAmount` then
    ///                                 function execution would throw a error.
    /// @param expiry:                  Unix timestamp after which trade would get invalidated.
    /// @param recipient:               A valid capability that receives target token after the completion of function execution.
    /// @param remainingSourceTokenRecipient: A valid capability that receives surplus source token after the completion of function execution.
	pub fun swapExactSourceToTargetTokenUsingPath(
		sourceToTargetTokenPath: Type[],
		sourceVault: @FungibleToken.Vault,
		exactTargetAmount: UFix64,
		expiry: UFix64,
		recipient: Capability<&{FungibleToken.Receiver}>,
        remainingSourceTokenRecipient: Capability<&{FungibleToken.Receiver}>
	)


    /// @notice It will Swap the source token for to target token and          
    /// return `ExactSwapAndReturnValue`
    ///
    /// If the user wants to swap USDC to FLOW then the
    /// sourceToTargetTokenPath is [Type<USDC>, Type<FLOW>] and
    /// USDC would be the source token.
    /// 
    /// This function would be more useful when smart contract is the function call initiator
    /// and wants to perform some actions using the receiving amount.
    ///
    /// Necessary constraints
    /// - For the given source vault balance, Swapped target token amount should be
    ///   greater than or equal to exactTargetAmount, otherwise swap would fail
    /// - If the swap settlement time i.e getCurrentBlock().timestamp is less than or equal to the provided expiry then the swap would fail
    /// - If the provided path doesn’t exists then the swap would fail.
    ///
    /// @param sourceToTargetTokenPath: Off-chain computed path for reaching source token to target token
    ///                                 `sourceToTargetTokenPath[0]` should be the source token type while
    ///                                 `sourceToTargetTokenPath[sourceToTargetTokenPath.length - 1]` should be the target token
    ///                                 and all the remaining intermediaries token types would be necessary swap hops to swap the
    ///                                 source token with target token.
    /// @param sourceVault:             Vault that holds the source token.
    /// @param exactTargetAmount:       Exact amount expected from the swap, If swapped amount is less than `exactTargetAmount` then
    ///                                 function execution would throw a error.
    /// @param expiry:                  Unix timestamp after which trade would get invalidated.
    /// @return A valid vault that holds target token and an optional vault that may hold leftover source tokens.
	pub fun swapExactSourceToTargetTokenUsingPathAndReturn(
		sourceToTargetTokenPath: Type[],
		sourceVault: @FungibleToken.Vault,
		exactTargetAmount: UFix64,
		expiry: UInt64
	): ExactSwapAndReturnValue

}

```

Events play a pivotal role in enhancing discoverability within a system; thus, this proposal extends its standardization efforts to include events as well. This concerted action aims to ensure that data aggregation or indexing platforms can readily curate data in response to user demands. By standardizing event structures, we strive to simplify data processing workflows and increase system transparency, facilitating efficient data retrieval and real-time analytics. The envisioned uniformity promises to provide a seamless experience for data aggregators and end-users alike, fostering a more intuitive and user-friendly environment.

```cadence
   /// Below event get emitted during `swapExactSourceToTargetTokenUsingPath` & `swapExactSourceToTargetTokenUsingPathAndReturn`
   /// function call.
   ///
   /// @param senderAddress:    Address who initiated the swap. // TODO: It is not possible to know as per the proposed interface.
   /// @param receiverAddress:  Address who receives target token, It is optional because in case of 
   /// `swapExactSourceToTargetTokenUsingPathAndReturn` function target token vault would be returned instead of movement of funds in
   ///  receiver capability.
   /// @param sourceTokenAmount: Amount of sourceToken sender wants to swap.
   /// @param receivedTargetTokenAmount: Amount of targetToken receiver would receive after swapping given `sourceTokenAmount`.
   /// @param sourceToken: Type of sourceToken. eg. Type<FLOW>
   /// @param targetToken: Type of targetToken. eg. Type<USDC>
   pub event Swap(
        senderAddress: Address,
        receiverAddress: Address?,
        sourceTokenAmount: UFix64,
        receivedTargetTokenAmount: UFix64,
        sourceToken: Type,
        targetToken: Type
    )

```

### Interface for Quotation

Design decision of `ImmediateSwapQuotation` resource interface revolves around the following requirements -  

- Alice as a user needs to know how much amount of targetToken she gets if she provide __x__ amount of sourceToken.
- Alice as a user needs to know how much amount of sourceToken she needs to get for __x__ amount of targetToken.
- For DEX developers, providing optimal routing to ensure the best possible quotation for users is of paramount importance. However, achieving this goal without an off-chain computed optimal path can be challenging due to several factors. For instance, liquidity may be dispersed across various pools rather than being concentrated in a single location. Additionally, there may be instances where a direct liquidity pair does not exist. Consequently, the use of an off-chain optimized path becomes integral to facilitating efficient and effective routing, thereby ensuring the provision of optimal quotations to users.

__Assumptions -__

- The end user is primarily concerned with the final output of a transaction, rather than the associated fees. The user's primary interest lies in knowing the amount of target tokens they will receive in exchange for a specified quantity __x__ of source tokens.

- The calculated quotation holds validity only at the exact moment of calculation. Due to the dynamic nature of the DeFi markets, there is no guarantee that the user will receive the same result upon submitting identical values at a later time. This uncertainty arises from potential fluctuations in pool values that may occur post-quotation calculation.


```cadence

pub resource interface ImmediateSwapQuotation {
	
    /// @notice Provides the quotation of the target token amount for the
    /// corresponding provided sell amount i.e amount of source tokens.
    ///
    /// If the source to target token path doesn't exists then below function
    /// would return `nil`.
    /// Below function would return the quoted amount after deduction of the fees.
    ///
    /// If the sourceToTargetTokenPath is [Type<FLOW>, Type<BLOCTO>]. 
    /// Where sourceToTargetTokenPath[0] is the source token while 
    /// sourceToTargetTokenPath[sourceToTargetTokenPath.length -1] is 
    /// target token. i.e. FLOW and BLOCTO respectively.
    ///
    /// @param sourceToTargetTokenPath: Offchain computed optimal path from
    ///                                 source token to target token.
    /// @param sourceAmount Amount of source token user wants to sell to buy target token.
    /// @return Amount of target token user would get after selling `sourceAmount`.
    ///
	pub fun getExactSellQuoteUsingPath(
	   sourceToTargetTokenPath: Type[],
	   sourceAmount: UFix64
    ): UFix64?


    /// @notice Provides the quotation of the source token amount if user wants to
    /// buy provided targetAmount, i.e. amount of target token.
    ///
    /// If the source to target token path doesn't exists then below function
    /// would return `nil`.
    /// Below function would return the quoted amount after deduction of the fees.
    ///
    /// If the sourceToTargetTokenPath is [Type<FLOW>, Type<BLOCTO>]. 
    /// Where sourceToTargetTokenPath[0] is the source token while 
    /// sourceToTargetTokenPath[sourceToTargetTokenPath.length -1] is 
    /// target token. i.e. FLOW and BLOCTO respectively.
    ///
    /// @param sourceToTargetTokenPath: Offchain computed optimal path from
    ///                                 source token to target token.
    /// @param targetAmount: Amount of target token user wants to buy.
    /// @return Amount of source token user has to pay to buy provided `targetAmount` of target token.
    ///
    pub fun getExactBuyQuoteUsingPath(
        sourceToTargetTokenPath: Type[],
        targetAmount: UFix64
    ): UFix64?
```


### Drawbacks

There is no real drawbacks because of this FLIP.

### Alternatives Considered

The current FLIP proposes the utilization of optimized routes for token swaps or quotations. Initially, the standard aimed to maintain a lightweight structure by explicitly specifying the source token and target token types instead of an optimized path. This approach assumes that a direct liquidity pool must exist between the provided source and target tokens for a successful swap, and if not, the transaction would fail.

A discussion took place regarding this matter on the forum post, and the community reached a consensus to incorporate support for an optimized path that satisfies the requirement of a direct pool. However, the reverse scenario is not feasible. For further details, please refer to the [forum post](https://forum.onflow.org/t/dex-standard-on-flow/4607/4).

### Performance Implications

The implementation of the current FLIP would not have any performance implications on the FLOW network or dApps. However, it would decrease the integration time between Automated Market Makers (AMMs) and wallets, thereby facilitating the creation of seamless and composable DeFi dApps.

### Dependencies

There is no external dependencies on this FLIP.

### Engineering Impact

The implementation of the proposed FLIP would have no impact on the FLOW network itself. However, existing AMMs and wallets that have already integrated with AMMs would need to make changes to support the proposed FLIP. Fortunately, since many AMMs are already utilizing a similar interface, it is anticipated that only minimal changes would be required, although these specific modifications fall outside the scope of the current FLIP.

### Best Practices

No changes in the best practices

### Tutorials and Examples

TODO

### Compatibility

TODO

### User Impact

AMMs on FLOW are required to conform to the proposed FLIP in order to offer a seamless experience to users. Nevertheless, end-users will not perceive any significant changes in the user interface. However, this implementation will simplify the lives of dApp developers and smart contract developers on the FLOW blockchain.

## Related Issues

- Oder-book based trading interface is out of the scope and it will be addressed in a separate FLIP.
- Error reporting is also out of the scope of the FLIP. It will be addressed separately.

## Prior Art

Several AMMs already exist within the FLOW ecosystem, including FLIP sponsors such as [Increment.Fi](https://increment.fi/) and other AMMs like [BloctoSwap](https://swap.blocto.app/#/), [Metapier](https://metapier.org/swap). The author of this FLIP collaborated closely with these entities, gaining valuable insights from their experiences, which informed the design of this FLIP. For additional information, please consult the corresponding [forum post](https://forum.onflow.org/t/dex-standard-on-flow/4607/4). 

## Questions and Discussion Topics

TODO
