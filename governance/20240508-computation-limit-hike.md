# FLIP-267: Increasing Computation Limit on Flow

| Status        | Draft                                            |
:-------------- |:---------------------------------------------------- |
| **FLIP #**    | 267
| **Author(s)** | Kshitij Chaudhary (kshitij.chaudhary@flowfoundation.com)           | 
| **Sponsor**   | Kshitij Chaudhary (kshitij.chaudhary@flowfoundation.com) |
| **Updated**   | 2024-05-08                                           |

## Introduction

Please refer to [this post](https://forum.flow.com/t/how-evm-transaction-fees-work-on-flow-previewnet/5751) for detailed insights into how transaction fees for EVM transactions will be calculated once EVM goes live on Flow Mainnet. With the introduction of the new parameter `EVMGasUsage`, the calculation of execution effort on Flow Cadence will be modified as follows:

```
Execution Effort =
    0.0239 * function_or_loop_call +
    0.0123 * GetValue +
    0.0117 * SetValue +
		43.2994 * CreateAccount +
	  EVMGasUsageCost * EVMGasUsage**
```

Here, `EVMGasUsage` represents the gas used by EVM, such as 21K gas for a simple send transaction. Meanwhile, `EVMGasUsageCost` denotes the ratio for converting EVM gas to Flow’s computation units, which may also be called “Gas-to-Compute ratio” or “G2C ratio.”

In this FLIP, we make two proposals -

1. A mechanism to convert EVM gas costs into Flow computation costs so that EVM transactions and operations can be added into the standard Flow transaction fee calculations
2. An increase in the computation limit for Flow transactions to reflect the performance improvements made to Cadence and the overall protocol, in order to allow each transaction to do more (when warranted).

## Problem Statement

EVM launch on Flow (”Crescendo”) presents an opportunity to better align with network objectives and user needs in terms of total computation required. The current execution effort (”compute”) limit of 9999 on Flow Cadence may be inadequate for larger EVM transactions. Flow should be able to match or surpass the capabilities of Ethereum, allowing transactions of up to 30M gas to be executed seamlessly.

To attain this objective, two approaches can be employed: (1) Implementing a gas-to-compute conversion ratio (`EVMGasUsageCost`) to ensure that larger contracts (30M gas) fit within the 9999 compute limit, and (2) raising the computation limit on Flow to accommodate larger transactions requiring higher compute. A combined approach leveraging both methods would offer the most effective solution.

## Solution

1. **Set Gas to compute conversion ratio at 5000:1 (or** `EVMGasUsageCost` **at 1/5000)**

Empirical data shows that G2C should not be lower than 1000:1, given the current computation limit of 9999 (or in other words, the current weights in “execution effort” calculation), otherwise we risk EVM transactions using up more computation time, than they pay for. Considering however that EVM transactions also consume compute units in cadence execution, the actual size of EVM transactions or contracts that can be executed with the ratio of 1000:1 would be much less than 10M. 

To match Ethereum’s 30M gas limit, a 5000:1 gas to compute ratio could be adopted to calculate execution effort (compute units) on Flow. Based on some tests, this ratio would allow for 30M gas, along with adequate compute allocation for cadence execution of such transactions. For instance, a 30M gas transaction with a 5000:1 conversion ratio would consume about 6000 compute units for EVM execution, leaving 3999 compute units for Cadence execution.

It’s important to note that for such sizable contracts however, even these compute units (3999 in the above example) may prove insufficient for cadence execution. Therefore, we should also consider revising Flow’s computation limit of 9999 to ensure seamless execution of the largest Ethereum contracts.

1. **Computation Limit**

Directly raising the computation limit entails substantial code modifications and collaboration with ecosystem developers. Alternatively, we can indirectly increase the limit by lowering the execution effort coefficients or weights (0.0239, 0.0123, 0.0117, and 43.2994 in the “execution effort” calculation). Decreasing these weights would effectively expand a transaction’s computation limit; in other words, more and larger operations would “fit within” the 9999 compute limit.

To determine the appropriate reduction factor for the weights, a thorough understanding of Flow’s execution capabilities in handling large transactions was necessary. This involved assessing how much “computation” could currently be accommodated in a block (on Mainnet24) and how much additional time a transaction could consume before encountering complications. For instance, if transactions in a block exceed the available block time (1.25s), they might need to be split across multiple blocks — a feature not currently supported; also, single transactions can never be split further. After evaluating these factors, it became apparent that increasing the transaction time by more than 5 times may lead to execution issues. Therefore, it is proposed to reduce the computation weights by a factor of 5.

Note that the proposed reduction in coefficients is proposed to be offset by a corresponding increase in the unit cost of execution on Flow (by 5x). This adjustment aims to ensure that there are no changes in the overall transaction fee paid for a transaction despite the higher computation limit and lower coefficients. See [this post](https://forum.flow.com/t/how-evm-transaction-fees-work-on-flow-previewnet/5751) to review the calculation method for transaction fees and to understand how a corresponding increase in the cost of execution units would negate any influence on the total transaction fee.

## Proposal (Summary)

1. Establish the “Gas to compute ratio” at 5000:1 by setting `EVMGasUsageCost` to `1/5000`.
2. Reduce the coefficients of execution effort by a factor of 5x, effectively increasing the computation limit by the same factor. Counterbalance the reduction in coefficient values with a 5x increase in the cost of execution unit on Flow (from 4.99E-08 to 2.495E-7).

```
Execution Effort =
    **0.00478** * function_or_loop_call +
    **0.00246** * GetValue +
    **0.00234** * SetValue +
		**8.65988** * CreateAccount +
	  **1/5000** * EVMGasUsage**
```

## Impact

The proposed/implemented changes aim to enhance the computation limit. For developers, the increase in effective computation limit along with the 5000:1 gas-to-compute ratio would empower them to execute larger contracts on EVM on Flow. The indirect method of increasing the compute limit (by changing the value of coefficients allowing larger transactions to “fit in”) eliminates any extra effort required to update their contracts for changes in the computation limit. Overall, this proposal would bring ‘EVM Equivalence’ in terms of contract and transaction size capabilities for EVM transactions on Flow.

## Next Steps

The community is invited to share feedback on this post. If approved, the changes will undergo testing on the testnet before deployment on Mainnet, proposed to be timed around the [Crescendo mainnet launch](https://flow.com/upgrade/crescendo).
