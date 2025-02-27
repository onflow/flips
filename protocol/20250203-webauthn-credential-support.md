---
status: Draft
flip: 264
authors: Tarak Ben Youssef (tarak.benyoussef@flowfoundation.org)
sponsor: Janez Podhostnik (janez.podhostnik@flowfoundation.org)
updated: 2024-02-03
---

# FLIP 264: WebAuthn Credential Support

# Objective

This FLIP proposes the integration of the WebAuthn standard into Flow, enabling users to leverage WebAuthn credentials such as passkeys for Flow transaction signing, in the same way they are used for authenticating to web services.

[WebAuthn](https://www.w3.org/TR/webauthn-3) is a standard provided by the [FIDO2](https://www.passkeys.com/what-is-fido2-fido-2-explained) framework to enable passwordless authentication.
It replaces passwords and multi-factor authentication with public key cryptography when logging into a service, for a more user-friendly experience and multiple security benefits (phishing resistance, no credential interception, avoid password reuse, resistance to server breaches, etc.). 

Passkeys are the most common form of WebAuthn credentials.
When a user registers for a web service, a new public passkey is generated on their device's authenticator and shared with the service's server.
The user's authenticator securely stores the private passkey and allows its use in future sessions through biometric authorizations (e.g., face ID).
The private passkey generates a cryptographic signature that authenticates the user to the web service.

Flow (like other blockchains) uses public key cryptography to authenticate account owners and authorize on-chain actions.
This is done by signing transactions using the account owner's private key, while the chain stores the public key counterpart for verification.
Flow already employs the same key infrastructure required by FIDO2 and can thus leverage the existing WebAuthn implementations for account authentication.
Flow users would inherit the high usability of passkeys provided by major platforms and ecosystems (Apple, Google, browsers, password managers, etc.).
Additionally, Flow users could also use other compatible WebAuthn credentials, such as hardware keys (e.g., Yubikey).

# Motivation

For self-custody wallets, enabling users to control their private keys introduces two main flaws.
First, users are responsible for safeguarding their secret seeds, which is prone to user error, leading to situations where users either lose access to their secrets or fall victim to exploits.
Secure elements provide a good solution to safeguard the user's private keys by preventing their leakage and unauthorized usage.
This turns devices equipped with secure elements into secure hardware wallets, eliminating the need for a secret seed and reducing the burden on users to safeguard their keys.
However, this solution introduces the second flaw in existing self-custody mechanisms: private keys are tied to specific devices making portability of the keys across devices not possible.
Losing the user's device often means losing access to their private keys.

FIDO2 is already implemented by major platforms and operating systems. These implementations use passkeys as FIDO2 credentials, which are backed by secure hardware and are very convenient to use.
On certain systems, passkey credentials are shared between the user's devices for easier access, and are securely backed up for later recovery in cases of loss.
Wallets based on passkeys would resolve the current self-custody flaws. 

# User Benefit

Flow account users would be able to use self-custody wallets based on passkeys from major platforms (Apple's Password app, Google's Password Manager, etc.). 
They would be able to register account public keys from their devices and authorize transaction signing via the device's biometric authentication.
This combination provides strong security through hardware elements (no phishing, protection at rest and in use) and high usability through the portability and recovery features provided by passkeys.
This would be possible without the need to remember seed phrases (as in hardware wallets) or master passwords (as in some browser wallets).

Flow account users would also be able to use any compatible WebAuthn credentials other than passkeys, to both register and authorize transactions. 

# Scope of the proposal

Currently, the Flow protocol cannot verify WebAuthn-generated signatures.
This FLIP outlines the changes required at the Flow protocol level in order to support WebAuthn credentials for both registration (adding public keys to new or existing Flow accounts) and authentication (Flow transaction authorization). This would enable the usage of WebAuthn-compatible credentials, including passkeys, to control Flow accounts.

The proposal focuses on the changes required at the Flow protocol level (mainly within the Access API and the Flow Virtual Machine).
It is not a complete guide for Flow client developers on how to integrate WebAuthn credentials and it does not replace the [WebAuthn specifications](https://www.w3.org/TR/webauthn-3).
Wallet developers must still refer to the original specifications on how to manage credentials and communicate with authenticators. 
That said, the proposal highlights the points where Flow client usage differs from the original WebAuthn authentication use case. 

# Design Proposal

## Terminology

The WebAuthn specifications originally provide a safe and user-friendly way to authenticate users into web applications.
A recent [variation](https://www.w3.org/TR/secure-payment-confirmation/), authored by the same [W3C](https://www.w3.org/) consortium, covers similar secure methods to authorize payments on web services.
Neither of these specifications was meant to address the case of blockchain transactions. 
This FLIP proposes using certain WebAuthn methods and functions for a different use case.
Here is how WebAuthn terminology corresponds to blockchain terms:

- [Public key Credential](https://www.w3.org/TR/webauthn-3/#public-key-credential): The cryptographic key pair used to authenticate. In Flow, this corresponds to the account key pair, with the private key controlled by the user and the public key stored on-chain.

- [Authenticator](https://www.w3.org/TR/webauthn-3/#authenticator): The cryptographic component (software or hardware) that generates and uses the credential private key. In Flow Wallets, the authenticator depends on the wallet implementation. In the common case of passkeys, it corresponds to the OS passkeys manager (for instance Apple's Passwords on iOS and macOS).

- [Client](https://www.w3.org/TR/webauthn-3/#client): The client-side components that interacts with a remote server. In Flow, this would be the wallet software interacting with the chain.

- [Relying Party server](https://www.w3.org/TR/webauthn-3/#webauthn-relying-party): The server side component of the authentication process that registers the user's public credentials and authenticates users. In Flow, the relying party runs on-chain within the Flow Virtual Machine (FVM). The transaction validation module validates the user assertions (signatures) while the execution state stores the public credentials (account public keys). While WebAuthn requires the relying party to connect to clients using a secure channel, the chain communication with clients is not confidential and does not require an encrypted channel. 

- [Authentication Assertion](https://www.w3.org/TR/webauthn-3/#authentication-assertion): The proof provided by an authenticator to the relying party (via a client) upon the user’s approval, proving possession of the private credentials. In Flow, the assertion is the transaction signature that proves ownership of the account's private key.

## Scenarios

### Account public key registration

- A wallet wants to add a new account public key to an existing Flow account, or create a new Flow account with a new public key.
- The wallet initiates the process by calling [`authenticatorMakeCredential`](https://www.w3.org/TR/webauthn-3/#sctn-op-make-cred). In most platforms this is invoked via the higher level web API [`navigator.credentials.create()`](https://www.w3.org/TR/webauthn-3/#sctn-createCredential). This requests the authenticator to create a new key pair credential with the options specified in [`PublicKeyCredentialCreationOptions`](https://www.w3.org/TR/webauthn-3/#dictionary-makecredentialoptions):
    - The relying party ID `rpID` is a string set by the wallet to identify the Flow credentials. This is a constant defined by the wallet for all Flow usage (for example `"FLOW-WEBAUTHN-MYWALLET-V0.0"`). Note that further map indices like [user handles](https://www.w3.org/TR/webauthn-3/#user-handle) and [credential IDs](https://www.w3.org/TR/webauthn-3/#credential-id) would allow the look up of the Flow private credential in the authenticator internal map.
    - Unlike the original WebAuthn registration case, the `challenge` is not sent by the server to the client. The wallet sets `challenge` to any constant string that should be stored temporarily during till receiving the authenticator response. It is not necessary to use a random challenge.
    - The list [`PublicKeyCredentialParameters`](https://www.w3.org/TR/webauthn-3/#dictdef-publickeycredentialparameters) must only contain items with the algorithm being `ES256` or `ES256k` as defined in the [COSE](https://www.iana.org/assignments/cose/cose.xhtml#algorithms) list. These are the only COSE algorithms currently supported by Flow accounts. `ES256` is ECDSA [using P-256]([)https://www.w3.org/TR/webauthn-3/#sctn-alg-identifier) with SHA2-256, while `ES256k` is ECDSA using secp256k1 curve and SHA2-256.
- Once the user confirms registration through an authorization gesture, the authenticator creates the key pair and returns an [AuthenticatorAttestationResponse](https://www.w3.org/TR/webauthn-3/#iface-authenticatorattestationresponse) to the wallet. The response contains all the data needed to verify and register the new account public key.
- The wallet extracts the new public key and the attestation signature from the authenticator response. It also constructs the [attested signed message](https://www.w3.org/TR/webauthn-3/#sctn-attestation) from the response and validates it contains the expected constant `challenge` set earlier. The wallet should also check the rest of the settings are set as expected in the attestation response such as the `rpIDHash` (should be hashed correctly from the `rpID`) and the other flags.  
- Unlike the traditional WebAuthn registration process, the attestation signature is not verified on the server side. It is highly recommended to verify it on the wallet side, against the newly created public key and the attested message. This step is important to validate the public key before finalizing its registration on-chain. It serves as a proof that the authenticator possesses the private key corresponding to the public credential. 
- If the validation succeeds, the wallet builds a new account public key using the newly generated public key, the signature algorithm returned by the authenticator, and the hash algorithm SHA2-256. 
- The wallet includes the new account public key into a transaction to create a new Flow account with the key, or to add the new account key to an existing Flow account. The transaction is then submitted by the wallet.
- The wallet stores the credential data required to look up the user private key in future calls to the authenticator. This may be a combination of the rpID, user handle and credential ID depending on the wallet implementation.
- The chain processes the transaction and no modifications are required on the Flow protocol level.

### User Transaction Submission

- A wallet needs to sign a transaction for an account previously registered with a WebAuthn credential.
- The wallet looks up the user's credential data locally, and pulls the transaction data required from the chain.
- The wallet initiates the process by calling [`authenticatorGetAssertion `](https://www.w3.org/TR/webauthn-3/#authenticatorgetassertion). On most platforms this is called via the higher level web API [`navigator.credentials.get()`](https://www.w3.org/TR/webauthn-3/#sctn-getAssertion). This requests the authenticator to generate an assertion signature using a previously generated private credential with the inputs provided in [`PublicKeyCredentialRequestOptions`](https://www.w3.org/TR/webauthn-3/#dictionary-assertion-options):
    - The relying party ID `rpID` must be the constant string set by the wallet for all Flow credential registrations.
    - Unlike the original WebAuthn assertion case, the `challenge` is not sent by the server to the client. In Flow, it is not necessary to use a challenge-response mechanism to sign transactions (more details in #replay-attacks). The wallet sets `challenge` to the signable transaction message hash. The hash used is SHA2-256, which produces a 32-bytes `challenge`. The transaction message is either the transaction [payload](https://developers.flow.com/build/basics/transactions#payload) or the [authorization envelope](https://developers.flow.com/build/basics/transactions#authorization-envelope), depending on the signer role. Note that a signable message in Flow includes a domain separation tag to scope the message to the Flow transaction context.
    - When calling `navigator.credentials.get()`, other fields of the request (including [CollectedClientData](https://www.w3.org/TR/webauthn-3/#dictionary-client-data)) are set implicitly by the web API and cannot be set by the developer.
- Once the user confirms the registration through an authorization gesture, the authenticator uses the private key to sign an internally constructed message and returns an [AuthenticatorAssertionResponse](https://www.w3.org/TR/webauthn-3/#iface-authenticatorassertionresponse) to the wallet. The response contains the signature as well as all the required data to rebuild the signed message. The exact signed message is [defined](https://www.w3.org/TR/webauthn-3/#sctn-op-get-assertion) by the WebAuthn specification as `webauthn_message = authenticatorData || Hash(json(collectedClientData))`, where [authenticator data](https://www.w3.org/TR/webauthn-3/#sctn-authenticator-data) is set by the authenticator and [`collectedClientData`](https://www.w3.org/TR/webauthn-3/#dictionary-client-data) is set by the wallet and includes the `challenge` above. 
- The wallet should sanity-check that `authenticatorData` and  `collectedClientData` returned by the authenticator are set according to the query. This includes the `challenge` value and the `rpIDhash` value (which should be hashed correctly from `rpID`). Other [authenticator flags](https://www.w3.org/TR/webauthn-3/#authdata-flags-up) should be checked as required by the WebAuthn specs and the wallet settings. In particular the User Presence `UP` must be set while the User Verification `UV` should be set according to the wallet preference.
- The wallet extracts the signature and the signed message data from the assertion response. It then builds a new transaction using the authenticator signature along with the extra signed message, forming the transaction signature (payload signature or envelope signature depending on the case).
- Once the transaction is submitted, the chain processes the transaction. The transaction signature is not verifiable on the current protocol because the signed message extends beyond the usual payload data (or authorization envelope data). The signed message includes new data added by the webauth scheme. Verifying these transactions require modifications on two different levels: the access API should be able to transmit the extra verification data as part of the transaction (changes described in #access-api-changes), and the FVM should be able to construct the WebAuthn verification message (changes described in #fvm-transaction-validation-changes).

## Access API changes

To support WebAuthn signatures, Flow's transaction signatures need to be extended to include additional verification data such as `authenticatorData` and `collectedClientData`. 
We refer to this data by `extension_data`. 
The new structure must support the WebAuthn scheme as well as continue to support the original non-webauthn scheme (also called plain scheme) without breaking changes.

Currently, the Flow transaction signature only supports the plain scheme, and is [defined](https://github.com/onflow/flow/blob/master/protobuf/flow/entities/transaction.proto#L26) as:
```protobuf
message Signature {
    bytes address = 1;
    uint32 key_id = 2;
    bytes signature = 3;
}
```
The proposed transaction signature to support additional schemes is:
```protobuf
message Signature {
    bytes address = 1;
    uint32 key_id = 2;
    bytes signature = 3;
    bytes extension_data = 4;
}
```
The  `extension_data` field represents any extra data required to verify the signature.
This will be used to support WebAuthn and to potentially support other schemes in the future.
The `signature` field continues to represent the cryptographic signature to be verified against the public key. 
The first byte of `extension_data` is a scheme identifier that scopes the signature to a defined signing scheme and specifies the signature verification process.
The signing scheme here identifies a protocol or a framework and should not be confused with cryptographic signature schemes (such as ECDSA, RSA, etc).

Here is how the bytes of `extension_data` should be set:
- The scheme identifier is a byte which encodes up to 256 possible schemes. The plain scheme identifier is `0x0`, while the WebAuthn scheme identifier is `0x1`. Only these two signature schemes will be supported in Flow for now.
- For backward compatibility, and to optimize for the plain scheme case (expected to be the commonly used scheme), the plain scheme identifier does not need to be used when using the plain scheme. The `extension_data` field is omitted when building a `Signature` struct and will be decoded to the default language value (e.g., an empty slice in Golang). `extension_data` needs to be included only when the scheme is not plain. This also avoids a malleability problem where setting `extension_data` to an array `{0x0}` would be equivalent to not setting one. 
- Any `extension_data` value that is not at least 1-byte in length or does not start with a valid scheme identifier (currently only `0x1`) makes the transaction signature invalid.

In the case of the WebAuthn scheme, `extension_data` should be encoded as:
```
 byte(webauthn_scheme_identifier) || 
		RLP({
		"authenticator_data" : bytes
		"collectedClientData_json" : bytes
		})
```

- `authenticator_data` is the [data](https://www.w3.org/TR/webauthn-3/#sctn-authenticator-data) set by the authenticator and returned in the assertion response. The data must be at least `35` bytes, and is the concatenation of the following fields:
    - the first 32 bytes represent the `rpIDHash`
    - the next byte represents the flags 
    - the following 4 bytes represent the `signCount`
    - the two remaining fields are `attestedCredentialData` and `extensions`. They are optional and usually not included. The flags byte encode the presence or not of these two fields. If included, they are both of variable size.
- `collectedClientData_json` is the json encoding of [`collectedClientData`](https://www.w3.org/TR/webauthn-3/#dictionary-client-data) (or `json(collectedClientData)`) as formed by the client when requesting an assertion from the authenticator. It is important to include the same json encoding used during the assertion request because `collectedClientData` is a dictionary and the field order of the serialization must be preserved. `collectedClientData_json` must be a valid json encoding of a dictionary with the required fields `"type"`, `"challenge"` and `"origin"`. The type field must be `"webauthn.get"` and the challenge must be decoded into exactly 32 bytes. The origin is a string of arbitrary size. This adds up to at least 93 bytes, with an average of about 145 bytes if the data are constructed honestly.

## FVM transaction validation changes

Upon receiving each transaction signature, the FVM transaction validation module performs the following steps to validate the transaction signature. It returns "valid" if the transaction signature is correct, and "invalid" otherwise.
The existing validation steps before this FLIP are not detailed.

- Decode the protobuf fields of the new proposed `Signature` structure.
- Check the public key has a valid `key_id` and is not revoked, otherwise return "invalid".
- If the `extension_data` field is not provided, set the current scheme to the plain scheme.
- If the `extension_data` field is provided, check it is of byte-size larger than `1`, otherwise return "invalid".
- Set the current scheme to `extension_data[0]`.
- Check that the current scheme is supported. Currently, only the WebAuthn scheme identifier (`0x1`) is accepted at this step. Otherwise return "invalid".
- At this point, the scheme is WebAuthn. RLP decode the remaining data `extension_data[1:]` into `authenticator_data` and `collectedClientData_json`. Return "invalid" if decoding fails. We recall the expected structure of `extension_data` in the webauthn case:
```
 byte(webauthn_scheme_identifier) || 
		RLP({
		"authenticator_data" : bytes
		"collectedClientData_json" : bytes
		})
```
- Json-decode `collectedClientData_json` into [`collectedClientData`](https://www.w3.org/TR/webauthn-3/#dictionary-client-data) and make sure the resulting dictionary contains the required keys `"type"`, `"challenge"` and `"origin"`. Return "invalid" if any of the steps fail.
- Extract the `challenge` value from the `collectedClientData` dictionary using the "challenge" key, and make sure it is exactly 32 bytes.
- Reconstruct the payload hash `SHA2-256(flow_domain_tag || payload)` from the received transaction and check that it equals the `challenge` field value. If the values do not match, return "invalid". Use the authorization envelope instead of `payload` in the case of a payer signature.
- Read the `type` value from the dictionary using the "type" key. If the value is different than `"webauthn.get"` return "invalid". 
- In WebAuthn, the server checks client from the dictionary such as `origin` and `crossOrigin` to make sure the assertion was generated for the right relying party. These checks aren't required in the Flow transaction case. Unlike the classic WebAuthn case, the wallet can connect to the chain via multiple access nodes. The domain tag scopes the assertion to the Flow transactions.
- Check `authenticatorData` has the minimum length of 37 bytes.
- Read `rpIDHash` and check it is not equal to the Flow plain transactions [Domain Separation Tag](https://github.com/onflow/flow-go/blob/bc341a46060ab2ee6c6b23d4dc5a6d4a262a9931/model/flow/constants.go#L85). 
- Extract the [user flags](https://www.w3.org/TR/webauthn-3/#authdata-flags-up) from [`authenticatorData`](https://www.w3.org/TR/webauthn-3/#sctn-authenticator-data) and check `UP` is set, `BS` is not set if `BE` is not set, `AT` is only set if [attested data](https://www.w3.org/TR/webauthn-3/#attested-credential-data) is included, `ED` is set only if [extension data](https://www.w3.org/TR/webauthn-3/#authdata-extensions) is included. If any of the checks fail, return "invalid".
- No other checks are required on the fields of `authenticatorData`. The `counter` check is not needed in Flow to mitigate replay attacks (more details in #replay-attacks). The `rpIDHash` is set arbitrarily by the wallet, while the extension [may be ignored](https://www.w3.org/TR/webauthn-3/#authn-ceremony-verify-extension-outputs) during the attestation verification, as long as they are covered by the cryptographic verification.
- Construct the message `webauthn_message = authenticatorData || Hash(collectedClientData_json)`.
- Compute the cryptographic verification of the `signature` value against `webauthn_message` and the cryptographic public key, taking into account the hashing algorithm stored in the account key. Return "valid" is the verification succeeds, and "invalid" otherwise.

### Access and Collection Validation

Access nodes and collection nodes make sanity checks on the ingested transactions.
As these nodes do not have access to the execution state (which contains the account public keys), the checks are limited to transaction format validation and contextual verification, including the signature fields.
Access node and collection nodes should integrate all the checks in the FVM validation process above, excluding the cryptographic signature verification any any other check involving the account public keys.

## Design and security considerations

### Large Transaction size

Some fields in the signature extension data have arbitrary size and can be made purposely long in malicious transactions.
Access nodes and collection nodes already implement a size check when ingesting transaction into the protocol and such malicious transactions won't pass the check.

### RP ID hash and the Flow DST

The signable message in the WebAuthn scheme begins with 32 bytes of `rpIdHash`, which is in honest cases, computed by hashing the `rpID`.
The signable message in the plain scheme also begins with 32 bytes of the [Flow Domain Separation Tag](https://github.com/onflow/flow-go/blob/bc341a46060ab2ee6c6b23d4dc5a6d4a262a9931/model/flow/constants.go#L85) (DST) constant.
The DST scopes the signable message to the Flow transaction context: `scoped_message = Flow_DST || payload`.
It would be an issue if a malicious/careless wallet chooses `rpIdHash` to match `Flow_DST`.
In that case, a signature generated by a private credential in the WebAuthn scheme would be a valid plain-scheme signature under the same private key, but with a different intent than the original transaction (or vice versa). 
FVM verification must check that the `RP_ID_HASH` attached to the signature data is different than `Flow_DST`, in the case of wallets that skip the `RP_ID` hashing step.

### Hash of the payload as a challenge

The proposal uses the `collectedClientData`'s challenge as the signable payload hash and not the payload.
It is indeed preferable to set `challenge` to the signable payload hash and not the unhashed signable payload.
The payload can be large and may cause issues in some authenticator implementations that are not expecting large challenges of more than 32 bytes.
The other reason is that `collectedClientData` is submitted as part of the transaction data. Setting the challenge to the unhashed payload would almost double the transaction size for large payloads.

### Replay attacks

The original WebAuthn scheme uses two layers of protection against replay attacks: 
- The random challenge-response mechanism: The server initiates the process by generating a random challenge and only authenticate the user when the challenge is answered (i.e. signed).
- The signing counter: This counter is privately stored on the server to track the signature count per authenticator. The authenticator must provide a valid incremented counter along with every signature (included in `authenticatorData`). The server compares the provided count in a signature against the server stored one. This counter also prevents authenticator cloning across sessions, where the local counters can get out of sync with the server's stored counter. 

The proposal does not require random challenges (the challenge is the payload hash) and does not store or check the authenticator's signing counts. 
Instead, Flow relies on the [proposal key sequence number](https://developers.flow.com/build/basics/transactions#sequence-numbers) to prevent replay attacks. 
The sequence number tracks the number of signatures per account public key, it is stored on-chain and FVM expects it to be incremented in each signed transaction (in a similar logic to WebAuthn's signing counts). 
Flow sequence numbers are not effective against authenticator cloning because the expected sequence number is publicly stored on-chain.
A cloned authenticator can look up the next valid sequence number before signing.

### Server side attestation verification

Unlike WebAuthn, Flow does not require a signature from the newly registered account key.
Account public keys can be added to new or existing accounts without providing a valid signature by the new private key (though the proposer key's signature must be valid).
WebAuthn new key registration results in the authenticator generating a signature attestation that is traditionally verified on the server.
Flow wallets are encouraged to verify the authenticator's attestation locally before registering the new key on-chain.
This would be a proof that the authenticator possesses the right user's private key. 

### Order of Cryptographic and Contextual checks

The proposed steps to validate a transaction signature by the FVM follow the same order logic used in the WebAuthn [server-side assertion verification](https://www.w3.org/TR/webauthn-3/#sctn-verifying-assertion): format and contextual checks first and cryptographic checks at the end.
This means that decoding and parsing the signature extension data is done first, while the more expensive cryptographic operation is left to the end. If any step fails, the verification aborts.
Collection nodes are supposed to have made the contextual checks. Transactions failing these checks are not supposed to reach the FVM stage. Starting by these checks on the FVM level would confirm that collection nodes behaved as expected by the protocol.
The cryptographic verification would then only impact the transaction payer. 

# Alternatives Considered

### RP ID is a protocol constant**

`RP_ID` could be a unique protocol-wide constant that is used by all wallets submitting WebAuthn transactions.
Wallets would use the Flow constant instead of arbitrarily choosing their own `RP_ID`.
`RP_ID_hash` can thus be omitted from `authenticatorData` when attached to the signature.
This reduces the size of a transaction signature by 32 bytes.

### Hash of the payload as `clientDataHash` and the high level API limitation

The `authenticatorGetAssertion` function accepts `clientDataHash` and not the full json encoding of `CollectedClientData`. 
The wallet design could have simply passed the signable payload hash as the input `clientDataHash`.
The authenticator would then sign the signable payload hash, and not the `clientDataHash`. 

This alternative not only simplifies the overall design, but it also makes the assertion verification on the FVM side much simpler. 
The payload hash does not need to be sent as an extension data with the signature because the verifier can recompute it from the transaction payload. The signable message in this case would be `authenticatorData || hash(signable_payload)`.

This method entirely omits the dictionary `CollectedClientData` and all the complexity of parsing and verifying its fields.
They will be no use of the dictionary fields `type`, `challenge`, `origin` and `crossOrigin`, which are all not necessary in the Flow integration. 
Flow transaction payloads already use a domain separation tag and do not require extra measures to scope the signature. 
The FVM wouldn't need to make the extra check of consistency between `challenge` and the payload hash. 

The last advantage of this approach is that only `authenticatorData` needs to be submitted with the signature, reducing the signature extension data size by about 150 bytes.

This alternative is not proposed because it is not easily implementable on most platforms.
`authenticatorGetAssertion` is a [CTAP](https://fidoalliance.org/specs/fido-v2.0-ps-20190130/fido-client-to-authenticator-protocol-v2.0-ps-20190130.html#authenticatorGetAssertion) function, which is the standard describing the communication between the client and the authenticator.
This is a low-level communication and is unfortunately abstracted from high level APIs in most platforms (browsers and OSs).
These platforms expose the public javascript web API which only exposes `navigator.credentials.get()` and hides the CTAP complexity from developers. 
`navigator.credentials.get()` does not accept `clientDataHash` as an input. Instead, it accepts other inputs that form the `CollectedClientData` dictionary. It JSON-encodes the dictionary internally and hashes it into `clientDataHash`, before invoking the low level `authenticatorGetAssertion`.  

Imposing this alternative makes the implementation of Flow passkeys wallets more complex, which may slow down adoption of the feature.
We expect most developers to rely on the higher level javascript API. Calling CTAP functions requires using low-level or native libraries like the [FIDO2 lib](https://github.com/Yubico/libfido2).

### Same signature struct

Attaching the extra signature data can be done by keeping the same `Signature` structure and including any extensions in the `signature` field. 
An internal structure of the field would need to be defined to encode the scheme identifier byte and the rest of the data. 
This would break with seeing the `signature` field as purely the cryptographic signature and would mix different data in the same field. Since the structure already cleanly abstracts the `address` and `key_id` from the signature bytes, adding a new separate field `extension_data` was preferred.

```protobuf
message Signature {
    bytes address = 1;
    uint32 key_id = 2;
    bytes signature = 3; // field would contain the cryptographic signature + the extension data
}
```

### Encode the scheme within account public keys

Instead of attaching the scheme information to the transaction signature (through the scheme identifier byte), the scheme information could have been attached the account public key.
The process of validation a signature would be specified to the FVM by looking at the account public key.
Different options have been considered to attach such information to the account key, and they all present disadvantages. Some of the drawbacks include:
 - An account public key would be bound to a single scheme, removing the flexibility of using the same account key in different schemes (without avoiding a new key registration). The scheme and key value are orthogonal and should ideally not be bound.
 - update the account public key format which results in Cadence language changes and breaks existing basic transactions to add keys and create accounts.
 - encode the scheme (plain, WebAuthn, and others) within the signature algorithm field of the account key, breaking with reading the algorithm field as a cryptographic signature algorithm and surcharging it with extra definitions. This would impact multiple tools and libraries in the ecosystem. 

# Drawbacks

These changes require a coordinated update across access, collection, execution and verification nodes to maintain the integrity and consistency of transaction validation.

# Performance Implications

The WebAuthn scheme transactions require larger transactions in size to include the extra verification data.
The extra data per WebAuthn transaction signature are at least 130 bytes (37 bytes for `authenticatorData` + 93 bytes for `collectedClientData_json`) and about 180 bytes in a general case for a valid signature.
The FVM execution overhead to decode the extra data and perform extra hashes is negligible.

There is no size or execution impact on the the non-WebAuthn transactions.

# Engineering Impact

The changes require code updates to the access API, FVM and collection node software.
Updates will also be required to some tooling libraries like Flow SDKs. 

# Compatibility

Proposed changes are backwards compatible.
Current plain scheme transactions (non-WebAuthn) continue to work as usual. 

