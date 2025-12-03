---
status: Accepted
flip: 336
authors: Vishal Changrani
sponsor: Vishal Changrani
updated: 2025-11-19
---

# FLIP 351: Transaction Fee Update to Enable Inflation-Neutral Tokenomics

# Objective

The objective of this FLIP is to update Flow’s **execution effort unit cost** and inclusion fees to align transaction fees with the recalibrated execution effort weights introduced in [FLIP 346: Execution Effort Calibration II](https://github.com/onflow/flips/pull/347).

With the proposed calibration in [FLIP 346: Execution Effort Calibration II](https://github.com/onflow/flips/pull/347), the average measured computation per transaction would decrease substantially, reducing total fees collected by the network by approximately 71% (section “Conversion Factor Adjustment for Constant Execution Effort” in FLIP 346). This FLIP adjusts the unit cost of computation to restore fee levels to their intended range and ensure that fees accurately reflect the updated execution cost model.

The specific goals of this FLIP are to:

1. **Update the execution effort unit cost** so that total transaction fees remain consistent with the intended economic targets (assuming adoption of FLIP 346).
2. **Progress toward an inflation-neutral network** by aligning transaction fee revenues more closely with validator rewards and reducing the protocol’s reliance on token issuance.
3. **Rebalance inclusion fees** to maintain their relevance relative to computation fees and ensure effective transaction prioritization and spam protection.

Overall, this FLIP serves as a necessary companion to FLIP 346, ensuring that Flow’s transaction fee parameters are correctly priced under the new execution effort model and continue to support sustainable network economics.

# Motivation

Transaction fees on Flow continue to be extremely low, and the problems highlighted in [FLIP 74: Revisiting Flow Transaction Fee](https://github.com/onflow/flips/blob/main/governance/20230323-transaction-fee.md) are still relevant today. The total transaction fees collected remains a small fraction of validator rewards, requiring continued token issuance by the protocol.

The main issues driving this proposal are:

- **Low transaction fees pose threats of attack.**

  The cost of submitting large volumes of transactions to Flow is orders of magnitude lower than on comparable networks (see [FLIP-74](https://github.com/onflow/flips/blob/main/governance/20230323-transaction-fee.md#impact)). This makes the network more susceptible to spam or denial-of-service attacks that exploit cheap transaction costs. [FLIP 346: Execution Effort Calibration II](https://github.com/onflow/flips/pull/347) further exacerbates this.

- **Continued token issuance by the protocol (inflation).**

  Because transaction fees do not cover validator rewards, new FLOW tokens must be minted each epoch to meet the target reward rate leading to inflation.


# Proposal

Currently, transaction fees are calculated as:

`Transaction fee = {inclusion fee + (execution effort * unit cost)} x surge`

where,

- Inclusion fee = 1E-6 FLOW
- Execution Effort Unit Cost = 2.495E-7 FLOW
- Surge = 1.0 (can vary as per load, see [FLIP-336](https://github.com/onflow/flips/blob/main/governance/20250727-dynamic-transaction-fees.md))
- Execution Effort is variable based on transaction type, as [shown here](https://github.com/onflow/flow/blob/c05d847adf2f6fb509e42c17020484d7dd3e89bd/flips/20220111-execution-effort.md). Execution effort will be updated by FLIP 346.

  (On-chain values can be viewed [here](https://www.flowview.app/account/0xf919ee77447b7497/storage/FlowTxFeeParameters))


It is proposed that,

1. Unit Cost of Execution Effort be set to 4E-05 .
2. Inclusion Fee be set to 1E-4 FLOW .

# Path to zero inflation

Lets look at how these changes get us to close to zero inflation.

For the first few years of Flow’s growth, transaction fees were set extremely low to remove friction and make it easy for builders to onboard consumers. This approach supported rapid adoption and helped the network mature with a broad set of applications.

Today the network operates with a [reward rate of 5%](https://developers.flow.com/protocol/staking/schedule#rewards-distribution) of total supply per year. With approximately 43% of the supply currently staked, validators earn an annual yield of about 12%. Under the existing fee parameters, transaction fees cover only a small share of these rewards. The recalibration introduced in [FLIP 346](https://github.com/onflow/flips/pull/347) lowers measured computation per transaction even further with most transactions being metered at less than 1/20th the computation effort of Crescendo. As a result, the network continues to rely heavily on new token issuance to fund operator rewards.

This proposal sets the transaction fees to a level that will offset all token issuance ("inflation") when the network sees throughput of approximately 50,000 computation per second.

This 50,000 target is very ambitious, but also plausible when you combine normal user transactions with the new scheduled transaction feature in Forte:

- User transactions:
    - A typical transaction on Flow today comes in between 50-100 computations using the FLIP 346 metering calculations
    - We assume that transactions will become more sophisticated over time, moving that average closer to 100 computation.
    - Many popular chains see usage in the 100-200 tps range, implying that user transactions can easily take up 10,000-20,000 computation per second.
- Scheduled transactions:
    - Scheduled transactions fees require a 2-10x premium over regular transactions, thus are likely to average around 300 computation each.
    - If anything, the number scheduled transactions is likely to exceed the number of user transactions on the network. For example, a consumer DeFi application with just 1 million users that uses one hourly scheduled transaction per hour would generate approximately 280 tps by itself.
    - This implies scheduled transactions could hit 40,000 - 60,000 computation per second.

Current performance measurements indicate that this throughput is realistic on the current mainnet, without significant optimizations or hardware scaling

Under the proposed fee parameters, transaction fee revenue can be sufficient to cover all validator rewards on an epoch-to-epoch basis, eliminating the need for new FLOW issuance and achieving zero inflation.

# Comparison with other chains

Here is how Flow will compare to other chains after this change:

| Network | 30-Day Average Transactions Fees (USD) |
| --- | --- |
| BTC | $0.7781 |
| ETH | $0.50 |
| Base | $0.046 |
| Solana | $0.015 |
| SUI | $0.004 |
| Flow | $0.001 |

# Conclusion

This FLIP recalibrates transaction fees to restore economic balance after FLIP 346 and represents a return to first-principles ensuring that transaction fees once again reflect actual resource consumption, sustain validator rewards without inflation, and uphold Flow’s long-term economic integrity.

# Related FLIPS

1. [FLIP-74: FLIP: Revisiting Flow Transaction fee](https://github.com/onflow/flips/blob/main/governance/20230323-transaction-fee.md)
2. [FLIP-267: Increasing the transaction computation limit](https://github.com/onflow/flips/blob/main/governance/20240508-computation-limit-hike.md)
3. [FLIP 753: Variable Transaction Fees - Execution Effort I](https://github.com/onflow/flips/blob/main/protocol/20220111-execution-effort.md) and [FLIP 346: Variable Transaction Fees - Execution Effort II](https://github.com/onflow/flips/pull/347)

# References

1. Source for Average Transaction Fees:

- BTC: [https://ycharts.com](https://ycharts.com/indicators/bitcoin_average_transaction_fee)
- Ethereum: https://etherscan.io/chart/avg-txfee-usd
- BASE: https://tokenterminal.com/explorer/projects/base/metrics/transaction-fee-average?interval=90d
- SUI: https://suiscan.xyz/mainnet/analytics/fees
- Solana: https://solanacompass.com/statistics/fees