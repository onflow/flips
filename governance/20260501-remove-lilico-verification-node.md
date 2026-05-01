---
Status: Draft
Flip: 367
Authors: Vishal Changrani
Sponsor: Vishal Changrani
Updated: 2026-05-01
---

# FLIP 367: Remove unresponsive Lilico verification node from approved list

## Objective

Remove the Lilico-operated verification node from the approved list of nodes so that it gets unstaked in the next epoch.

## Motivation

Verification node `3c6519ba8be35e338df7273a895ad3abaeb0c232eb908ee7b05462018c112fe1` operated by Lilico has not been updated to the latest node software. The Flow Foundation has made multiple attempts to contact the node operator, all of which have gone unanswered. The node is running with tokens leased by the Foundation.

An outdated verification node poses a risk to network health and reliability. Since the operator is unresponsive and the node runs on Foundation-leased tokens, the appropriate action is to remove it from the approved list.

## User Benefit

Improved network reliability by ensuring all verification nodes are running up-to-date software.

## Design Proposal

Remove node ID `3c6519ba8be35e338df7273a895ad3abaeb0c232eb908ee7b05462018c112fe1` from the approved list of nodes. This will cause the node to be unstaked at the next epoch boundary.

### Drawbacks

The network loses one verification node, marginally reducing verification redundancy. However, an outdated node provides limited value and may actively harm network operations.

### Alternatives Considered

- **Continue outreach**: The Foundation has already made multiple attempts with no response.
- **Do nothing**: Leaves an outdated node running on the network, which is undesirable.

### Performance Implications

No negative performance impact. Removing an outdated node may improve network performance.

### Dependencies

None.

### Engineering Impact

Minimal. This is a configuration change to the approved node list.

### Best Practices

Node operators running on leased tokens are expected to keep their nodes updated and remain responsive to the Foundation.

### Tutorials and Examples

Not applicable.

### Compatibility

No compatibility concerns. This is a governance action.

### User Impact

The Lilico verification node will be unstaked at the next epoch. No impact on other users or node operators.

## Related Issues

- [FLIP Tracker Issue](https://github.com/onflow/flips/issues/367)

## Prior Art

- [FLIP 365: Removing Blocto as a multi-signer](https://github.com/onflow/flips/blob/main/governance/20260413-removing-blocto-from-multi-sig.md) — similar governance action for an unresponsive community member.

## Questions and Discussion

Discussion will take place on the PR itself.
