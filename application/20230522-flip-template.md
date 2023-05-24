---
status: draft 
flip: NNN (do not set)
authors: My Name (me@example.org), AN Other (you@example.org) 
sponsor: Satyam Agrawal (satyam.agrawal@dapperlabs.com) 
updated: YYYY-MM-DD 
---

# Pool-Based DEX Swap Standard

## Objective

The Flow ecosystem is experiencing continued growth and with this expansion comes a need to expand the DeFi ecosystem on the Flow blockchain. Expansion of DeFi on Flow requires new liquidity - typically through cross-chain bridges - as well as straightforward integration for wallets and developers to provide a simple and intuitive user experience. While the macro topic of liquidity is beyond the scope of this discussion, the initial focus of the nascent standard aims to enable ease of interoperability with, and composability of, DEXs which are a cornerstone concept serving as gateways to any DeFi ecosystem. This FLIP is focused on the pool based DEXs like Uniswap style AMMs.

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

/// Data structure that get returned after the `swapExactSourceToTargetTokenUsingPathAndReturn` function execution.
pub struct interface ExactSwapAndReturnValue {
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

Why should this *not* be done? What negative impact does it have? 

### Alternatives Considered

* Make sure to discuss the relative merits of alternatives to your proposal.

### Performance Implications

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

## Questions and Discussion Topics

Seed this with open questions you require feedback on from the FLIP process. 
What parts of the design still need to be defined?
