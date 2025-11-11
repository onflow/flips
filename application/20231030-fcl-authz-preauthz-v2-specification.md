---
status: proposed
flip: 215
authors: Jordan Ribbink (jordan.ribbink@dapperlabs.team)
sponsor: Jeffrey Doyle (jeffrey.doyle@dapperlabs.com)
updated: 2023-10-30
---

# FLIP 215: FCL Authz/Pre-Authz v2.0.0 Specification

## Objective

This FLIP outlines a proposed update to the FCL (Flow Client Library) specification which would introduce v2.0.0 authorization (authz) and pre-authorization (pre-authz) services to address limitations of the existing design.

## Motivation

It has been 3 years since the original FCL specification outlining authorization services (known as authz & pre-authz) was defined. Together, these two services aimed to resolve one or more signatures for a transaction using keys whose cumulative signatory weight for each account was ≥1000. This combination of signatures would confirm that the accounts participating in each transaction role had approved the transaction’s execution.  FCL's authorization services have demonstrated exceptional robustness. However, it appears that these services have now surpassed their initially envisioned use cases.

Currently, key IDs used to authorize accounts may only be defined prior to signing during the pre-authz phase or statically in an authz service definition; authz services are responsible for generating signatures for a distinct account-key pair.  While a signature from any sufficiently-weighted account key would be considered a valid authorization at the protocol level, FCL strictly requires signatures from a predefined key that it anticipates an authorization service to use.

Recently, the community has voiced its concerns regarding shortcomings of the current authorization model.  Specifically, the discussion [raised in FCL-JS#1679](https://github.com/onflow/fcl-js/pull/1679) has highlighted the need to reevaluate how these services dictate which key IDs are used to authorize accounts.  In this pull request, the Blocto team has identified a need to specify the key ID used by authz services at the time of signing.  These evolving wallet requirements have illuminated that V1.0.0 FCL authz services may have been unnecessarily restrictive, which motivated the creation of this FLIP.

## User Benefit

For the end user, the benefit of this proposal is access to new wallet features which were previously difficult or impossible to implement. Notably, allowing key IDs to be specified during the authorization step would streamline building self-custodial wallets sharing a single account with keys across multiple user devices.  This proposal would help surface one of Flow’s flagship features, native account abstraction, and to promote the growth of Flow's non-custodial ecosystem.  To the end user, this means real ownership of their digital assets.

## Design Proposal

This proposal outlines a refined, key-agnostic approach for resolving account authorizations.  Namely, a new configuration for FCL authz services would no longer require that authorizations are resolved using a specific signature from a predefined account key.  Instead, authorizations would be constructed using one or more signatures from any keys on the authorizing account.

In addition to adding the key-agnostic authz configuration, FCL must continue to support the key-specific authz service structure (current model).  This is because proposer authorizations remain as an edge case that is, by definition, not key-agnostic.  Transaction payload signatures are generated based on a predetermined proposer key - this cannot change during the authorization phase and must be established prior to authorizers receiving a payload to sign (i.e. pre-authz phase). There must remain an key-specific authorization strategy valid for the transaction proposer which will continue to use a static `keyId`.

In this document, we will establish the following terminology:

- **key-specific** authz services - the current authz service model
- **key-agnostic** authz services - the proposed revision to the authz service model (removes `keyId` field)

Effectively, v2.0.0 of the authz service would make the `keyId` field optional and only enforce `keyId` constraints on the response if it were from a key-specific authz service (i.e. explicit `keyId` specified in authz service definition).

Additionally, authz v2.0.0 will introduce a new response format to help better facilitate multi-signature account authorizations.  Since the requirements of explicit authz `keyId` fields will be relaxed, it is natural that authz services are to allow multiple keys signatures from the authorizing account to combine to form authorizations.  This will be outlined later in the document.

Wallets wishing to integrate authz v2.0.0 with their pre-authz services will also be required to implement the v2.0.0 pre-authz specification.  This service identical to pre-authz v1.0.0 with the only difference being added compatibility with authz v2.0.0.

### Authz Service Structure

The following would be the schema for v2.0.0 key-specific authz services.  It is identical to v1.0.0 authz services, however the `f_vsn` field is bumped to v2.0.0.

```jsx
{
    "f_vsn": 2.0.0 // currently, v1.0.0
    "f_type": "Service",
    "id": "foobar#authz-http-post",
    "type": "authz",
    "identity": {
        "address": "f086a545ce3c552d",
        "keyId": 615 // explicit keyId expected to sign
    },
    "method": "HTTP/POST",
    "endpoint": "<https://example.com/authz>",
    "params": { ... }
}

```

In the above example, FCL expects that `https://example.com/authz` will respond with a signature for `keyId=615` on account `f086a545ce3c552d`.  No other keys’ signatures would be valid responses.

Key-agnostic authz services would discard this assumption, instead conforming closer to the protocol-level definition of account authorizations.  Rather than defining authorizations in terms of account-key pairs, they would be defined as a collection signatures using any number of arbitrary keys from a given account.

For the key-agnostic v2.0.0 authz spec, `keyId` would become an optional parameter.  When describing a key-agnostic authz service, the only difference would be the removal of the `"keyId"` field.

```jsx
{
    "f_vsn": 2.0.0,
    "f_type": "Service",
    "id": "foobar#authz-http-post",
    "type": "authz"
    "identity": {
        "address": "f086a545ce3c552d", // just an address, no keyId
    },
    "method": "HTTP/POST",
    "endpoint": "<https://example.com/authz>",
    "params": { ... }
}

```

The above authz service defines an HTTP/POST request to `[https://example.com/authz](https://example.com/authz)` which should resolve one or more signatures `f086a545ce3c552d` using any keys from the signing account.

### Authz Service Response

For v1.0.0 authz services, the response was only a CompositeSignature object defined as follows:

```jsx
{
    ...polling response...
    "data": {
        "f_vsn": "1.0.0",
        "f_type": "CompositeSignature",
        "addr": "0x358529c1f0220750",
        "keyId": "1"
        "signature": "0db1718281a5c5587b5b4b026f12aed576b3e2dec5a297fab17c40a2c1e46f1b93fe537934bf0d5208c7cabe9196f24d4a0ba34a45ec958f8f9295139624cf3d"
    }
    ...
}

```

A notable limitation in the current response is that it only accommodates a single signature. The proposed revision to the authz service response would enable the inclusion of arrays of signatures.

Imagine an example authz service which is responsible for resolving an authorization for `0x358529c1f0220750`.  Assume that this account has two keys, both with 500 weight, and that this authz service is responsible for resolving signatures for both of these keys.  The response from this service could be as follows:

```jsx
{
    ...polling response...
    "data": [
        {
            "f_vsn": "1.0.0",
            "f_type": "CompositeSignature",
            "addr": "0x358529c1f0220750",
            "keyId": "1" // 500 weight
            "signature": "0db1718281a5c5587b5b4b026f12aed576b3e2dec5a297fab17c40a2c1e46f1b93fe537934bf0d5208c7cabe9196f24d4a0ba34a45ec958f8f9295139624cf3d"
        },
        {
            "f_vsn": "1.0.0",
            "f_type": "CompositeSignature",
            "addr": "0x358529c1f0220750",
            "keyId": "2" // 500 weight
            "signature": "0db1718281a5c5587b5b4b026f12aed576b3e2dec5a297fab17c40a2c1e46f1b93fe537934bf0d5208c7cabe9196f24d4a0ba34a45ec958f8f9295139624cf3d"
        }
    ]
    ...
}

```

In this case, the authz service successfully obtains a 1000-weight authorization by using a combination of two keys’ signatures.  It is natural that key-agnostic authz responses are not restricted to a single CompositeSignature.  This design is more compatible with the protocol-level definition of account authorizations: the account authorization process may require an aggregate of signatures from multiple keys as opposed to a single key.  In some situations, this may prevent redundant authz services and unnecessary API requests.

It is not a requirement that the cumulative weight of the signatures from a single authz service is ≥1000.  Multiple authz services may have to work together in order to achieve this (as was the case for v1.0.0 authz/pre-authz).  This depends on specific configuration of authz services established by the wallet.

Of course, a single CompositeSignature passed as a plain object, and not an array, would still be sufficient as well; e.g.

```jsx
"data": {
    "f_vsn": "1.0.0",
    "f_type": "CompositeSignature",
    "addr": "0x358529c1f0220750",
    "keyId": "1"
    "signature": "0db1718281a5c5587b5b4b026f12aed576b3e2dec5a297fab17c40a2c1e46f1b93fe537934bf0d5208c7cabe9196f24d4a0ba34a45ec958f8f9295139624cf3d"
}

```

### Multi-signer transactions? Is there enough weight?

One question that may arise is the following: how does the removal of explicit account-key pairs from authz services affect multi-signer transactions.  Specifically, how can it be ensured that the combined signature resolution of all authz services would result in a satisfactory weight for transaction approval (1000)?

Ultimately, these concerns are not relevant.  It’s up to the authn-provider (wallet) to issue a collection of authz services that generate signatures of combined weight ≥1000 - this concern is out of the scope of FCL.  These guarantees should hold true due to the wallet guarantees and it is not the client’s responsibility to coordinate this.

However, should these services require additional metadata for key-orchestration, this can be defined by the pre-authz step and passed through `"params"` field of the authz services returned by this step.  Required metadata may differ by the wallet’s specific use-case.

### Overlapping Keys

A key-specific authz service makes a guarantee that two distinct authz services will not produce signatures using overlapping keys.  One question may be: How can we make the same guarantee with key-agnostic authz services.

This concern can be addressed using the same rationale as mentioned earlier for multi-signer transactions. Key orchestration will be an inherent guarantee of the services the wallet chooses to advertise to clients.  The wallet will have the responsibility of ensuring that their provided authz services do not incur key collisions.  This may require that additional metadata be supplied to these services; this can be configured via the `"params"` field of an authz service.

### Drawbacks

The drawback with this approach is the added complexity for wallets to support both v1.0.0 & v2.0.0 authz/pre-authz services (see Engineering Impact).

### Alternatives Considered

This FLIP was inspired by a PR submitted to the FCL-JS repository by Blocto [https://github.com/onflow/fcl-js/pull/1679](https://github.com/onflow/fcl-js/pull/1679).  This approach did not alter the specifications of the authz or pre-authz services, but instead adjusted how FCL interprets the response received from the authz service.  If the wallet specified an alternate `keyId` as part of their authz response, it would override the expected `keyId` for the authz service.  The advantage of this approach is its apparent simplicity compared to the one outlined in this FLIP.

However, there are a few disadvantages that can be identified:

1. It lacks safeguards for backwards compatibility - wallets may try to override the authz `keyId` with an older FCL client which does not yet support this feature.
2. This is a workaround that subverts the declared intent of authz services.  FCL requests a signature for a specific key from an authz service and the service may respond with a signature for a different key - should this response be valid to FCL?
3. Unlike the design proposal in this FLIP, it maintains the limitation that an authz service may only return a signature for a single `keyId`.  Authz services lack the ability to construct a multi-signature authorization for a given account which is a natural addition when relaxing the `keyId` requirement.

### Performance Implications

Performance is generally not applicable to this FLIP.  By advertising a new version of authz/pre-authz services wallets will trivially use more network bandwidth during the authentication phase (<1kb).

### Dependencies

N/A

### Engineering Impact

FCL-compliant wallets will need to create new endpoints to upgrade to v2.0.0 authz/pre-authz services.  Additionally, existing v1.0.0 endpoints must be supported in order to maintain compatibility with older FCL clients.

### Best Practices

Wallet providers should upgrade their codebase to reflect these changes.  Future versions of FCL may not support v1.0.0 authz/pre-authz services, so it is important they upgrade as soon as possible.

### Tutorials & Examples

This feature is purely wallet-facing and abstracted from the developer who uses FCL to build their application.  As such, it is unnecessary to complete a full-fledged tutorial using this feature.

However, it will be necessary to update the FCL wallet provider specifications to reflect these changes.  Additionally, Flow Reference Wallet would integrate v2.0.0 authz/pre-authz services into their codebase, serving as an example for wallet developers looking upgrade to this feature.

### Compatibility

Ensuring backward compatibility is critical when making changes to the FCL spec. Wallets wishing to upgrade to authz/pre-authz v2.0.0 may do so as long as the FCL client they are interacting with supports this feature.

**Service Version Selection**

The wallet and the client (FCL) will negotiate whether they are using the revised authz schema using the following technique:

1. The wallet will advertise all of the FCL services & versions of these which they are compatible with during the authentication (authn) phase. For example:
    
    ```jsx
    [
    	...services...,
    	{
    		"f_vsn": "1.0.0",
    		"f_type": "Service",
    		"type": "authz"
    		...
    	},
    	{
    		"f_vsn": "2.0.0",
    		"f_type": "Service",
    		"type": "authz"
    		...
    	}
    ]
    
    ```
    
2. When executing a given service (i.e. `execService('authz')`), the FCL client will choose to execute the greatest compatible version supported the client.  This guarantees that the most recent version supported by both the client & wallet is selected.

Using these two steps, it is possible to incrementally upgrade services from both the wallet & FCL client while maintaining double-sided backwards compatibility.

So, to implement key-agnostic authz services from a wallet’s standpoint, it means adding additional versioned services for the following:

- `pre-authz` v2.0.0 - If a wallet wishes to return key-agnostic authz services during their pre-authorization step, they must provide an additional v2.0.0 pre-authz service.
- `authz` v2.0.0 - If a wallet wishes for their default authz service (no pre-authz phase) to be key-agnostic, they must provide an additional v2.0.0 authz service. However, it is important to remember that key-agnostic services are not valid for proposer authorizations. If a wallet offers a key-agnostic authz service to the user, it is necessary that the proposer authorization will be overridden by a key-specific authz service during a pre-authz step - if this does not happen, the transaction will fail.

**FCL-JS Filtering Issue**

The technique outlined above **will not work** with FCL-JS versions <1.7.0.  While service version selection was always an intended feature of FCL-JS, it was previously impossible due to an unfortunate bug that has existed since the birth of the library.  This was unbeknownst to the library maintainers as wallet service specifications have not yet been iterated beyond v1.0.0.

Two workarounds are possible to avoid breaking compatibility with older FCL-JS versions:

1. Wallets will have to ensure that the `fclVersion` field provided in the authentication (sign in) step is ≥1.7.0 in order to use v2.0.0 authz & pre-authz services.  If the FCL-JS client version is <1.7.0, wallets must continue to specify only v1.0.0 services.
2. Because of the nature of the previous design flaw in FCL-JS, the first version of a service listed in the authn response will be selected in affected versions of FCL-JS (<1.7.0).  I.e. if both v1.0.0 & v2.0.0 authz services were supplied by the wallet, listing authz v1.0.0 at an earlier position than authz v2.0.0 in the services array would ensure that older clients always select the v1.0.0 service.

## Prior Art

- See [Blocto’s initially proposed solution](https://github.com/onflow/fcl-js/pull/1679)
- See [v1.0.0 FCL Specification](https://github.com/onflow/flips/blob/main/application/20221108-fcl-specification.md)
