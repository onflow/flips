| Status        | Proposed                                            |
:-------------- |:----------------------------------------------------|
| **FLIP #**    | [259](https://github.com/onflow/flips/pull/195) |
| **Author(s)** | Yahya Hassanzadeh (yahya@dapperlabs.com) Khalil Claybon (khalil.claybon@flowfoundation.org)           |
| **Sponsor**   | Yahya Hassanzadeh (yahya@dapperlabs.com) Khalil Claybon (khalil.claybon@flowfoundation.org)           |
| **Updated**   | 8 September 2023                                    |

# Message Forensic (MF) System

## Summary
This FLIP discusses and compares two potential solutions for the Message Forensic (MF) system in the Flow protocol 
— a system that enables Flow protocol to attribute protocol violations to the original malicious sender. 
The two solutions under consideration are: (1) GossipSub Message Forensic (GMF), and (2) Flow-level Signing Policy (FSP) For All Messages. 
We delve into both, listing their pros and cons, to determine which would be more feasible given the considerations of ease of implementation, 
performance efficiency, and security guarantees.

Our analysis finds the "Flow-level Signing Policy (FSP)" to be the more promising option, 
offering a generalized solution that doesn’t hinge on a specific external protocol to send the message (e.g., GossipSub),
steering clear of the complexities tied to maintaining GossipSub envelopes and dodging the necessity of duplicating GossipSub 
router’s signature verification procedure at the engine level. Furthermore, it meshes well with the Flow protocol’s existing state.

## Review Guide
This FLIP is presented as a Pull Request (PR) in the `onflow/flips` repository. We welcome reviewers to express their opinions and share feedback directly on the PR page, aiming for a structured and productive discussion. To aid this, please adhere to one of the following response frameworks:
1. I favor the "Flow-level Signing Policy (FSP)" and here are my thoughts:
2. I support the "GossipSub Message Forensic (GMF)" approach, articulating my views as follows.
3. I find both propositions unsatisfactory, elucidating my stance with.

## Problem Overview
Within the Flow protocol, nodes converse through a networking layer, a medium which undertakes the dispatch and receipt of messages among nodes
via different communication methods: unicast, multicast, and publish. Unicast messages are transmitted to a single recipient over a direct connection.
The multicast and publish, on the other hand, utilize a pub-sub network constructed with the LibP2P GossipSub protocol for message 
dissemination. However, this protocol encounters challenges in message attribution, particularly in determining the 
initial sender of a message, which is critical for identifying protocol violations and penalizing the malicious initiators. 
Below, we break down the issues seen in different communication strategies and their implications.

### Single Recipient (Unicast) Message Attribution
Unicast messages have an implicit attribution to the sender, i.e., they are attributed to the remote node of the connection
on which the message is received. Albeit, the situation differs when the unicast message carries the explicit signature of the
sender or other attribution data (e.g., the signature of the original sender in case of a forwarded unicast).
In such cases, the message is attributed to the sender that is specified in the attribution data. Nevertheless, the current state of
Flow protocol does not enforce the attribution data to be present in the unicast messages. 
A prime example is the [`ChunkDataResponse` messages](https://github.com/onflow/flow-go/blob/master/engine/execution/provider/engine.go#L326-L337) sent by the Execution Nodes to the Verification Nodes over a unicast. 
Hence a node in Flow protocol cannot prove that a received unicast message is sent by a specific node, unless the message carries the explicit attribution data.

### Group and All-Node (Multicast and Publish) Message Attribution
When dealing with multicast or publish messages, the GossipSub router initially [signs it](https://github.com/libp2p/go-libp2p-pubsub/blob/master/sign.go#L109-L134) with a local node's networking key 
before releasing it into the pub-sub network. The routers are structured to verify message signatures before forwarding them or making 
them accessible to the application layer (i.e., Flow protocol engines). This method does indeed block invalid messages from progressing through the network, 
also [penalizing](https://github.com/onflow/flow-go/blob/master/insecure/integration/functional/test/gossipsub/scoring/scoring_test.go#L26-L188) the immediate sender of an invalid message through a built-in scoring mechanism that influences network connections and message forwarding decisions.
Yet, a significant loophole remains: the elimination of authentication data (like signatures) by the GossipSub router during message delivery to the application layer. 
This eradication obstructs the Flow protocol's ability to trace a message back to its original sender in cases of protocol violations that happen at a level above the GossipSub layer.

## Message Structure in Flow
At broad level, the Flow-protocol engines send and receive events as `interface{}` types to each other through their `Conduit` interface of the networking layer. Upon receiving 
an event, the networking layer performs two rounds of encoding before sending it over the wire. 
- First, the event is [encoded using the CBOR encoding scheme to bytes, which becomes the payload of an internal `messages.Message` protobuf type](https://github.com/onflow/flow-go/blob/master/network/message/message_scope.go#L178-L200).
- Second, for the unicast cases, the `messages.Message` protobuf type is encoded directly to bytes, and is [written over streams](https://github.com/onflow/flow-go/blob/master/network/p2p/p2pnet/network.go#L650) to the remote node. For the pubsub cases, 
the `message.Message` protobuf type is [wrapped](https://github.com/onflow/flow-go/blob/master/network/p2p/p2pnode/libp2pNode.go#L422-L438) in a GossipSub envelope `pb.Message` type (internally by the GossipSub), and is gossiped within the network.

Hence, on the receiving path, the networking layer performs two rounds of decoding before delivering the event to the Flow-protocol engines. 
- For the unicast cases, the networking layer reads the bytes from the stream, and decodes the `messages.Message` protobuf type from the bytes. Then, the `messages.Message.Payload`
is decoded to an `interface{}` type event using CBOR, and is delivered to the Flow-protocol engines.
- For the pubsub cases, the networking layer receives the `pb.Message` type from GossipSub, and decodes the `messages.Message` protobuf type from the `pb.Message.Data` bytes. 
Then, the `messages.Message.Payload` is decoded to an `interface{}` type event using CBOR, and is delivered to the Flow-protocol engines.

In the rest of this FLIP, we refer to `pb.Message` as the GossipSub envelop, and to the `message.Message` as the Flow message. Hence, in a nutshell:
- For the unicast case, the event is wrapped in a Flow message type.
- For the pubsub case, the event is wrapped in a Flow message type, which is then wrapped in a GossipSub envelope type.
- The GossipSub envelope contains the signature of the sender over the entire envelope. The signature is generated using the networking key of the sender by the GossipSub router.

## Proposal-1: GossipSub Message Forensic (GMF)
As the first option to (partially) mitigate the gap in the network to hold the malicious senders accountable,
we introduce the GossipSub Message Forensic (GMF) mechanism designed to integrate GossipSub authentication data seamlessly with the 
Flow protocol engines. The principal aim is to enhance message authenticity verification through multicast and publish,
focusing on origin identification and event association. Here, we elucidate the method, its interface, and delve into the operational specifics, 
weighing its pros and cons against potential alternatives.
In this proposal, we propose a mechanism to share the GossipSub authentication data with the Flow protocol. We call this mechanism
GossipSub Message Forensic (GMF). 

At the current state of the code, the networking layer receives the [raw GossipSub envelope](https://github.com/onflow/flow-go/blob/master/network/p2p/p2pnet/internal/readSubscription.go#L48) from its underlying pubsub router, 
validates it, and passes the incorporated Flow message to for further decoding. The Flow message is then decoded to an `interface{}` type event,
and is delivered to the Flow protocol engines. The idea is to pass along the GossipSub envelop to the Flow protocol engines together with their `interface{}` type event.
In this way, the Flow protocol engines can have access to the GossipSub authentication data. The rough idea is depicted in the following figure. 
Note that it is not a production-ready code, we need to architect the solution in an Object Oriented Manner with proper abstraction and encapsulation of the interface. In this figure,
we intentionally pass along the envelope together with the event for sake of simplicity. In the actual implementation, we need to abstract the envelope and event in a single object 
and pass it to the Flow protocol engines. We have to also account for the unicast case that does not have a GossipSub envelope.
```go
// MessageProcessor represents a component which receives messages from the
// networking layer. Since these messages come from other nodes, which may
// be Byzantine, implementations must expect and handle arbitrary message inputs
// (including invalid message types, malformed messages, etc.). Because of this,
// node-internal messages should NEVER be submitted to a component using Process.
type MessageProcessor interface {
    // Process is exposed by engines to accept messages from the networking layer.
    // Implementations of Process should be non-blocking. In general, Process should
    // only queue the message internally by the engine for later async processing.
    Process(channel channels.Channel, originID flow.Identifier, event interface{}, envelope *pb.Message) error
}
```

In order to enable the engines with verification capabilities of the envelope signature in this proposal, we add new interface methods to the [`EngineRegistry` interface](https://github.com/onflow/flow-go/blob/master/network/network.go#L37-L56).
The `EngineRegistry` was previously known as the `Network` interface, and is exposed to individual Flow protocol engines who are interested in receiving the
messages from the networking layer including the GossipSub protocol. In this approach, the `EngineRegistry` interface is extended
by one new method: `VerifyGossipSubMessage`. The `VerifyGossipSubMessage` method takes an origin identifier, an event `interface{} type`,
as well as the GossipSub message as input, and verifies the signature of the message against the networking public key of the origin id. 
The method returns true if the signature is valid, and false otherwise.
```go

type EngineRegistry interface {
	// VerifyGossipSubMessage takes a Flow identifier as well as a GossipSub message as input, and verifies the signature of the
	// message. The method returns true if the signature is valid, and false otherwise.
	// Args:
	//  - originId: The Flow identifier of the node that originally sent the message.
	// - message: The GossipSub message that is associated with the event.
	// Returns:
	//  - bool: True if the signature is valid, and false otherwise.
	//  - error: An error if the message is not found.
	VerifyGossipSubMessage(originId flow.Identifier, event interface{}, message *pb.Message) (bool, error)

	// Other methods are omitted for brevity
	// ...
}
```

### Implementation Complexities
- This proposal requires refactoring the message processing pipeline of the networking layer to pass along the GossipSub envelope together with the event to the Flow protocol engines. We need to
  also refactor the `Engine`'s `Process` method in a way that it can accept the GossipSub envelope together with the event. The implementation should account for proper encapsulation and abstraction of the interface,
  to also support the unicast case that does not have a GossipSub envelope.
- This solution also requires replicating the signature verification logic of the GossipSub in `VerifyGossipSubMessage` so that it is accessible to the engines. 
  We need to extend the signature verification mechanism to account for translation of `originId` from `flow.Identifier` to networking key and `peer.ID` (i.e., LibP2P level identifier).
  As the engines are operating based on the `flow.Identifier`, while the GossipSub signatures are generated using the Networking Key of the node.

### Advantages
1. The GossipSub authentication data is shared with the Flow protocol, providing attribution for `Multicast` and `Publish` messages.
2. The interface is easy to use for the engines, as it abstracts the complexity of translating the origin Flow identifier to the GossipSub peer id, and
   verifying the signature of the message against the networking public key of the origin id.
3. The implementation is **not a breaking change** and is backward compatible with the current state of the Flow protocol.

### Disadvantages
1. This solution does not cover the unicast messages, as they are not sent through the GossipSub protocol.
2. To construct the GossipSub-level signature on an event, this solution requires maintaining the entire [GossipSub message (i.e., envelope)](https://github.com/libp2p/go-libp2p-pubsub/blob/master/pb/rpc.pb.go#L143-L153) which is more than the event itself. 
   This is because the GossipSub-level signature is generated on the entire message (i.e., envelope) and not just the event. Hence, this solution requires more memory footprint.
3. This solution requires replicating the signature verification logic of the GossipSub router at the engine level (i.e., `VerifyGossipSubMessage`). Changes to the GossipSub signature 
    verification procedure may pose as breaking changes and be a blocker for upgrading and keeping up with GossipSub upgrades.
4. Verification logic is not straightforward, as it requires at least two steps:
   - The first step is to ensure the event is wrapped in a GossipSub envelope. If not, the verification fails. For this we need to replicate the entire encoding path down to the GossipSub level
      as wrapping the Flow message in the GossipSub envelope is done internally at the GossipSub and is not exposed to the Flow codebase. The replication may also cause another layer of coupling 
      that causes breaking changes in the future upgrades of GossipSub.
   - The second step is to verify the signature of the GossipSub envelope against the networking public key of the origin id. This requires translating the origin id from `flow.Identifier` 
      to networking key and `peer.ID` (i.e., LibP2P level identifier), and replicating the signature verification logic of the GossipSub router, which is another layer of coupling that 
      causes breaking changes in the future upgrades of GossipSub.

## Proposal-2: Flow-level Signing Policy (FSP)
In this proposal, we introduce the policy to enforce a Flow-level signature for all messages using the staking key of the node. 
The idea is to refactor the [`Conduit` interface](https://github.com/onflow/flow-go/blob/master/network/conduit.go#L26-L57), so that instead of taking an `interface{}` type event, it takes an `Envelope` type event. 
The `Envelope` type event is a wrapper around the `interface{}` type event, and contains
the staking key signature of the sender on the event. The `Envelope` type is defined as follows:
```go
type Envelope struct {
    // The event that is wrapped by the envelope.
    Event interface{}
    // The Staking Key signature of the sender on the event.
    Signature []byte
}
```

Accordingly, the `Conduit` interface is refactored as follows:
```go
type Conduit interface {
	Publish(envelope *Envelope, targetIDs ...flow.Identifier) error

	Unicast(envelope *Envelope, targetID flow.Identifier) error

	Multicast(event *Envelope, num uint, targetIDs ...flow.Identifier) error

	// Other methods are omitted for brevity
	// ...
}
```

In this design, the engine on the sender side is required to sign the messages that are sent through the `Conduit` interface. To impose the safety mechanism, 
the `Conduit` interface rejects any message that is not signed by the sender by returning an error. 
On the other hand, the [`Engine` (aka `MessageProcessor`)](https://github.com/onflow/flow-go/blob/master/network/engine.go#L14-L60) interface is refactored to receive an `Envelope` type event instead of an `interface{}` type event. 
The `Engine` interface is refactored as follows:
```go
// MessageProcessor represents a component which receives messages from the
// networking layer. Since these messages come from other nodes, which may
// be Byzantine, implementations must expect and handle arbitrary message inputs
// (including invalid message types, malformed messages, etc.). Because of this,
// node-internal messages should NEVER be submitted to a component using Process.
type MessageProcessor interface {
	// Process is exposed by engines to accept messages from the networking layer.
	// Implementations of Process should be non-blocking. In general, Process should
	// only queue the message internally by the engine for later async processing.
	Process(channel channels.Channel, originID flow.Identifier, envelope *Envelope) error
}
```
Prior to delivering the message to the engine (i.e., `MessageProcessor`), the networking layer will verify the signature of the `Envelope` against the staking key of the
sender. If the signature is valid, the message will be delivered to the engine. Otherwise, the message will be rejected and reported to the Application Layer Spam Prevention (ALSP system) as an impersonation attempt, hence penalizing the
node who sent it over the network (the remote node in unicast case, and the RPC sender in multicast and publish cases).
In this way, when an engine's `Process` method is called, the engine can be sure that the message is signed by the sender. 
Hence, the engine can attribute a protocol-level violation to the malicious sender that originally sent the message.

### Implementation Complexities
- This proposal requires refactoring the `Conduit` interface, which is used by all the Flow protocol engines to send messages to the networking layer. 
  Currently, engines are passing arbitrary `interface{}` types as events to the conduits. This proposal requires refactoring the `Conduit` interface and 
  engines to pass `Envelope` type events to the conduits. 
- This proposal requires refactoring the `Engine` interface, which is used by all the Flow protocol engines to receive messages from the networking layer. 
  Currently, engines are receiving arbitrary `interface{}` types as events from the networking layer. This proposal requires refactoring the `Engine` interface and 
  engines to receive `Envelope` type events from the networking layer. This is a breaking change and is not backward compatible with the current state of the Flow protocol.
- This proposal requires refactoring the networking layer of the Flow blockchain to verify the signature of the `Envelope` against the staking key of the sender prior to 
  delivering the message to the engine.

### Advantages
1. This solution enables Flow protocol with attribution of a protocol-level violation to the malicious sender that originally sent the message.
2. The implementation is simple; contrary to the GMF proposal, there is no need to maintain and pass-along the GossipSub envelopes and no extra memory footprint.
3. This solution not only covers the GossipSub messages, but also covers all the messages that are sent through the `Conduit` interface. Hence, it is a more general solution
   that can be used to attribute a protocol-level violation to the malicious sender that originally sent the message, regardless of the protocol that is used to send the message.
   For example, the current state of codebase does not enforce signature policy for the `ChunkDataPack` responses that Execution Nodes send to the Verification Nodes over unicast. Hence,
   a verification node does not have any forensic evidence to attribute a protocol-level violation to the malicious Execution Node that originally sent the message.
4. The entire authentication data is at the Flow protocol level (i.e., Staking Key) and is not dependent on the GossipSub protocol. Hence, the Flow protocol can attribute a
   protocol-level violation to the malicious sender that originally sent the message, regardless of the protocol that is used to send the message, i.e., GossipSub in this case.
6. `Envelope`-based message representation provides nested authentication, i.e., an envelope generated by sender A can be wrapped as an authenticated evidence by sender B, and so on, with
   minimum overhead. This is a powerful feature that can be leveraged to build a robust forensic system.

### Disadvantages
1. The implementation is a **breaking change** and is not backward compatible with the current state of the Flow protocol. We change the `Conduit` interface, which is used by
   all the Flow protocol engines to send messages to the networking layer. Hence, all the Flow protocol engines must be refactored to sign the messages that are sent through
   the `Conduit` interface. 
2. This approach adds a computation overhead to the Flow protocol engines, as the engines must sign all the messages that are sent through the `Conduit` interface, and on the receiving side, the 
   networking layer must verify the signature of the message against the Staking Key of the sender. This overhead is not negligible, as the Flow protocol engines are the most performance critical components of the Flow blockchain. 
   Hence, we must carefully evaluate the performance overhead of this approach. Moreover, with this approach, we are extending the size of data sent over the wire by piggybacking the signature.
   Assuming that we are using ECDSA with `secp256r1` curve, the signature size is 64 bytes. Hence, we are adding 64 bytes to the size of data sent over the wire.
   Nevertheless, the performance overhead and the size of data sent over the wire may be an acceptable trade-off for the security guarantees that this approach provides.

### Is the GossipSub Router Signature Still Necessary?
Yes, maintaining the GossipSub router signature is essential, even while adopting Proposal-2. 
The chief purpose of this signature is to authenticate messages at the GossipSub layer, serving as a robust mechanism to weed out invalid messages early in the transmission process,
before reaching the application layer or being relayed to other nodes. Moreover, GossipSub needs its own signature to penalize the immediate sender of an invalid message through a built-in 
scoring mechanism that influences network connections and message forwarding decisions.
Drawing a parallel, it is akin to the ephemeral signatures employed in secure sessions like TLS and SSL. 
These signatures work diligently to verify the authenticity of the session-level communications. 
However, they are transient and do not function to attribute the message to its original sender after the session concludes. 
For post-session message attribution, application context-based signatures are used, preserving the integral role of GossipSub-level signatures for real-time session authentication.

### Is the `Entity` type still required?
In the Flow codebase, [`Entity` objects](https://github.com/onflow/flow-go/blob/master/model/flow/entity.go#L9-L26) are self-identified data structures, meaning that they are singularly identified through their unique hash value. Conversely, the newly proposed `Envelope` type operates through a 
framework of self-authentication; it relies on its intrinsic signature to confirm authenticity.
This critical distinction underscores that the `Envelope` type does not serve to replace the `Entity` type. In fact, a viable long-term strategy might involve strictly enveloping the `Entity` type within the `Envelope` type to leverage the strengths of both, as demonstrated below:
```go
type Envelope struct {
    // The event that is wrapped by the envelope.
    Event Entity
    // The Staking Key signature of the sender on the event.
    Signature []byte
}
```

### Do We Need Additional Signatures for Certain Data Structures in Flow?
In the Flow architecture, various data structures, including the [`ExecutionReceipt` structure](https://github.com/onflow/flow-go/blob/master/model/flow/execution_receipt.go#L11-L18), come with a staking key signature attributed to their creating execution node. 
However, this stipulation does not apply universally across all data structures present in Flow, with `ChunkDataResponse` being a notable exception that does not demand the signature 
of its originating execution node.

Preserving the internal signature of a data structure holds considerable importance for future reference and validation processes.
This retention is especially critical when introducing data structures to new nodes, which may either be lagging or freshly joining the network; 
the internal signature by the creator facilitates the verification process, allowing the new node to establish trust in the data structure.
On the other hand, the envelope-level signature characterizes the node wrapping the data structure, diverging from the creator’s signature found in the data structure itself.
This approach substantially enhances forensic capabilities. Taking the instance of a scenario where a sender manipulates an execution receipt, compromising its integrity, 
the system is designed to not just detect the violation via the executor's internal signature but to robustly attribute the breach to the malicious sender (via `Envelope` validation) ensuring non-repudiation.
This process, backed by irrefutable forensic evidence, strengthens the transparency and accountability to external parties.

To fortify security and streamline verification processes, the proposal at hand suggests a reevaluation of the existing signature protocols.
We envision a system where the intrinsic signatures of data structures are detached from the overarching envelope-level signature policy. This proposal serves dual purposes:
1. **Internal Signature As Part Of An Event (e.g., `ExecutionReceipt`'s executor signature):** Ensures protocol-level validation during the data structure processing, affirming that a claimed node genuinely created it. This signature retains its relevance over time, facilitating trust and verification for nodes newly integrated or lagging in the network, as it substantiates the original creator's identity and integrity of the data.
2. **Flow-level Signature:** Plays a pivotal role in tracing protocol violations back to malicious senders, hence fostering accountability and deterring fraudulent activities. Interestingly, this signature operates temporarily, losing its necessity once the engine affirms the message's validity.
This revamped policy opens up the opportunity to reconsider the necessity of internal signatures in various data structures, potentially eliminating them if deemed redundant, thereby streamlining the security process without compromising on the foundational principles of computer security.
