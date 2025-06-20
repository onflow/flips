---
status: draft 
flip: 332
authors: Vishal Changrani
sponsor:
updated: 2025-06-19
---
# FLIP 332: Enforcing Domain-Based Networking Addresses for Nodes

## Objective

This FLIP proposes stricter validation rules for node networking addresses during registration and updates. Currently, the staking contract allows the use of both IP addresses and domain names, with or without a port. This flexibility introduces security and operational concerns.

The FLIP proposes to:
- Disallow IP addresses entirely as networking addresses.
- Require that networking addresses include a valid domain name and port.
- Apply these same rules when updating an existing node's networking address.

### Summary of Accepted vs Rejected Formats

❌ **Rejected**:
- `193.23.53.12:3569`
- `193.23.53.12`
- `flownode.xyz.com` _(missing port)_

✅ **Accepted**:
- `flownode.xyz.com:3569`
- `flownode.888.com:1234`

## Motivation
### Background

To register a node on the Flow network, an operator submits a [node registration transaction](https://developers.flow.com/networks/staking/staking-collection#register-stakers), which includes the networking address of the node. This address is how other nodes establish peer connections with this node.

The networking address has two parts - the TCP/IP address or domain name of the node and the port used by the node for peer-to-peer communication.q

While [best practices](https://developers.flow.com/networks/node-ops/node-operation/node-bootstrap#generate-your-node-keys) recommend using a fully qualified domain name (FQDN) and including the port, the staking contract does not enforce these recommendations.

### Drawbacks of IP-Based Addresses

1. **Unstable connectivity**: Nodes running in cloud environments may not have a static IP unless specifically provisioned. If the IP changes, other nodes can no longer connect to it, severely impacting availability.
2. **Poor portability**: Migrating a node to another machine (e.g. for maintenance or hardware upgrades) will likely change the IP, breaking peer connectivity.

### Advantages of Domain Names

3. **Resilience and flexibility**: Domain names can be mapped to new IPs by simply updating DNS records, ensuring seamless migration without disrupting node operation.
4. **Security**: Using DNS enables additional security layers like DNS-based filtering and monitoring.

Furthermore, omitting the port number causes node initialization to fail. This alone justifies enforcing its inclusion.

## Design Proposal

A validation check will be added to the [`FlowIDTableStaking` contract](https://github.com/onflow/flow-core-contracts/blob/master/contracts/FlowIDTableStaking.cdc).

### Draft Implementation

A preliminary implementation is available in this [pull request](https://github.com/onflow/flow-core-contracts/pull/484/files#diff-65336be374bb3fc9ad7b822243e065a389d73e758c7c16223e52fc3181cea59bR170).

This same validation must also be applied to updates made through the [`NetworkingAddressUpdate` function](https://github.com/onflow/flow-core-contracts/blob/master/contracts/FlowIDTableStaking.cdc#L53).

> **Note**: This change is not retroactive. Existing nodes using IP addresses will not be affected.

## Performance Implications

This change introduces no performance overhead, as the additional validation is limited to a simple format check at the time of registration or update.

## Dependencies

None.

## Compatibility

- **Backward compatibility**: This change is **not** backward-compatible for node registration or networking address updates involving IPs or missing ports. However, it **does not** affect already registered nodes.
- **Tooling implications**:
  - **FlowPort** must implement client-side validation to block invalid formats before submission.
  - **FCL**, **Emulator**, and **Flow SDKs** are unaffected unless they perform client-side validation, in which case they should mirror the new logic.

## Status

A draft implementation is available for review in [PR #484](https://github.com/onflow/flow-core-contracts/pull/484).
