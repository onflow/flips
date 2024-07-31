# FLIP-284: Crescendo Path to Mainnet

| Status        | Proposed         |
:-------------- |:-----------------|
| **FLIP #**    | 284              |
| **Author(s)** | Vishal Changrani | 
| **Sponsor**   | Vishal Changrani |
| **Updated**   | 2024-07-30       |

## Introduction

This FLIP proposes a pathway in which smart contracts along with the transactions and app that use them can be validated as being staged or updated for the upcoming Crescendo release. In this rollout there will be logic deployed that ensures that all the smart contracts being used in a transaction are successfully staged for Mainnet meaning that after the Crescendo upgrade this transaction will still function. While doing this inspection if they fail the entire transaction can fail on the network so that applications and users can be aware there are missing dependencies after the Crescendo upgrade. So that we can ramp up the network and developersthe network will be building up the rate in which transactions that are inspected and fail on the network starting with a 0% rate in which failed inspected transactions fail on the network and incrementally increasing to 100% by the end of a multi-week long period. This approach ensures that by the, all transactions on the network will have been rigorously tested and validated, guaranteeing that Crescendo’s mainnet environment is robust and reliable.

## Motivation
The main goal of this FLIP is to transition smoothly to a more stable and reliable mainnet environment without causing significant disruptions to developers and users. The strategy is designed to allow for gradual adaptation, extensive testing, and iterative improvements, significantly reducing the risk of major issues when the new system is fully implemented.

## Design
- Initiation Phase: Starting with a transaction failure mechanism at 0%, allowing the network and its participants to adapt without initial disruption.
- Escalation Mechanism: Utilizing a continuous escalation strategy, where the percentage of transactions set to fail will increase with each block, reaching 100% failure rate.
- Monitoring and Evaluation: Implementing continuous monitoring of transaction failures, with a system in place to evaluate and mitigate issues if the failure rate exceeds 50% of total network transactions at any time.

## Implementation
### Tools and Technologies
- The implementation will utilize Flow’s Verifiable Random Function (VRF) to ensure randomness and fairness in the transaction failure process.

### Roles and Responsibilities
- Designated team members will monitor the upgrade process, ensuring that any necessary adjustments are made promptly.
- Actual adjustments on-chain will be made by the service committee.

## Test Cases
- Early Phase Testing: Evaluate the system’s response and developer’s adjustments when the failure rate is below 10%.
- Mid-Phase Testing: Assess network stability and data integrity when the failure rate reaches around 50%.
- Final Phase Testing: Confirm that all systems operate smoothly at 100% failure rate and that all necessary mitigations are effective.

## Dependencies
- This FLIP depends on the successful integration of Flow’s VRF and the readiness of the network’s existing infrastructure to support the incremental upgrades.

## Security and Privacy Considerations
- The upgrade process will be closely monitored for any potential security vulnerabilities that could arise from increased transaction failures.
- Privacy will be maintained according to Flow’s standard operational protocols, ensuring that no sensitive data is compromised during the testing phases.

## References
- Issue tracking this FLIP: [284](https://github.com/onflow/flips/issues/284)
- Forum Post (to be posted)
