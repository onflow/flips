---
status: draft 
flip: NNN (set to the issue number)
authors: Andrii Slisarchuk (andriyslisarchuk@gmail.com)
sponsor: AN Expert (core-contributor@example.org) 
updated: 2023-11-14 
---

# [FLIP DRAFT] Access Streaming API Expansion

## Objective

This FLIP document aims to logically extend the functionality proposed in [FLIP73](https://github.com/onflow/flips/blob/main/protocol/20230309-accessnode-event-streaming-api.md).

## Motivation and User Benefit

The newly implemented Access Streaming API offers limited functionality, featuring a single gRPC subscription, `SubscribeExecutionData`, and one gRPC/REST subscription, `SubscribeEvents`. On the other hand, the regular polling Access API version has a variety of endpoints for use. This FLIP identifies essential subscriptions within the current Access Subscription API, crucially based on user proposals, aiming to expand its functionality. Furthermore, it addresses recommended improvements to existing REST WebSocket connections, as suggested by the community, aimed at resolving issues arising from a large number of connections while maintaining granularity within the subscription API.

The enhanced version seeks to provide a simpler and more efficient REST WebSocket connection, offering a broader range of useful subscription endpoints for obtaining blockchain updates.

## Design Proposal

The Access Node Subscription API would introduce the following new streaming endpoints:

- SubscribeBlocks
- SubscribeBlocksHeaders
- SubscribeBlocksLightweight
- SendAndSubscribeTransactionStatuses
- SubscribeAccountStatuses - ?
- SubscribeResourcesMovement - ?
- SubscribeResourcesChanges - ?
- SubscribeTransactionResults - ?

Additionally, it proposes an improved version of the REST WebSocket Subscription with a single connection.

### WebSocket Subscriptions with a Single Connection

In the current version of REST subscription, [a new WebSocket connection is created](https://github.com/onflow/flow-go/blob/9577079fd24aad4c1fc1fc8710ef2686ead85ca4/engine/access/rest/routes/websocket_handler.go#L288) for each subscription request. However, with the potential growth in the number of endpoints available for subscription, this behavior becomes undesirable due to several reasons:

- It results in a system for connection handling on the client side that is difficult to maintain.
- It causes unnecessary network load, impacting the overall performance of the application.
- Eventually, it will hit the connection limit, posing constraints.
- It complicates future expansion efforts.

To address this, an improvement in WebSocket behavior is proposed by implementing a single connection for each subscription. This improvement is commonly referred to as the "WebSocket Components Subscription Pattern", and an example of its React implementation can be found [here](https://blog.stackademic.com/websockets-and-react-wscontext-and-components-subscription-pattern-4e580fc67bb5).

The proposed enhancement involves the following aspects:

- The `WSHandler` will initiate a connection only for the initial subscription and retain it for subsequent subscriptions.
- An unsubscribe mechanism will be introduced. With multiple subscriptions on a single connection, simply closing this connection to halt updates becomes insufficient. Therefore, a dedicated endpoint for unsubscribing from undesired updates is necessary.
- Each subscription response will now include a message type. For instance, the `SubscribeEvents` endpoint will result in a message with `type: "Events"`, while `SubscribeBlocksHeaders` will yield `type: "BlockHeader"` in messages, and so forth. This is essential to distinguish between various subscription results that will be delivered through a single connection.

### SubscribeBlocks

This endpoint enables users to subscribe to the streaming of blocks, commencing from a provided block ID or height. Additionally, users are required to specify the desired status of the streamed blocks to receive. Each block's response should include the `Block` containing `Header` and `Payload`.

- **BlockHeight** (Block height of the streamed block)
- **BlockId** (Block ID of the streamed block)
- **BlockStatus** (`BlockStatusSealed` or `BlockStatusFinalized`)

Usage example:

```go
req := &executiondata.SubscribeBlocksRequest{
    // If no start block height or ID is provided, the latest block is used
    StartBlockHeight: 1234,
    // or StartBlockID: startBlockID[:],
    BlockStatus: BlockStatus.BlockStatusFinalized, // Use BlockStatusSealed to receive only finalized or only sealed blocks
}

stream, err := client.SubscribeBlocks(ctx, req)
if err != nil {
    log.Fatalf("error subscribing to blocks: %v", err)
}

for {
    resp, err := stream.Recv()
    if err == io.EOF {
        break
    }

    if err != nil {
        log.Fatalf("error receiving block: %v", err)
    }

    block := resp.GetBlock()

    log.Printf("received block with ID: %x, Height: %d, and Payload Hash: %x",
        block.ID(),
        block.Header.Height,
        block.Payload.Hash(),
    )
}
```

### SubscribeBlocksHeaders

This endpoint enables users to subscribe to the streaming of block headers, commencing from a provided block ID or height. Additionally, users are required to specify the desired status of the streamed blocks to receive. The response for each block header should include the block's `Header`. This is a lighter version of `SubscribeBlocks` as it does not include the heavier `Payload` field.

- **BlockHeight** (Block height of the streamed block)
- **BlockId** (Block ID of the streamed block)
- **BlockStatus** (`BlockStatusSealed` or `BlockStatusFinalized`)

Usage example:

```go
req := &executiondata.SubscribeBlocksRequest{
    // If no start block height or Id is provided, the latest block is used
    StartBlockHeight: 1234,
    // or StartBlockId: startBlockID[:],
    BlockStatus: BlockStatus.BlockStatusFinalized // Use BlockStatusSealed to receive only finalized or only sealed blocks headers
}

stream, err := client.SubscribeBlocksHeaders(ctx, req)
if err != nil {
	log.Fatalf("error subscribing to blocks headers: %v", err)

}

for {
	resp, err := stream.Recv()
	if err == io.EOF {
		break
	}

	if err != nil {
		log.Fatalf("error receiving block header: %v", err)
	}

    blockHeader := resp.GetBlockHeader()

	log.Printf("received block header with ID: %x, Height: %d and Payload Hash: %x",
		blockHeader.ID(),
		blockHeader.Height,
        blockHeader.PayloadHash,
	)
}
```

### SubscribeBlocksLightweight

This endpoint enables users to subscribe to the streaming of lightweight block information, commencing from a provided block ID or height. Additionally, users are required to specify the desired status of the streamed blocks to receive. The response for each block should include only the block's `ID`, `Height` and `Timestamp`. This is the lightest version among all block subscriptions.

- **BlockHeight** (Block height of the streamed block)
- **BlockId** (Block ID of the streamed block)
- **BlockStatus** (`BlockStatusSealed` or `BlockStatusFinalized`)

Usage example:

```go
req := &executiondata.SubscribeBlocksLightweightRequest{
    // If no start block height or Id is provided, the latest block is used
    StartBlockHeight: 1234,
    // or StartBlockId: startBlockID[:],
    BlockStatus: BlockStatus.BlockStatusFinalized // Use BlockStatusSealed to receive only finalized or only sealed blocks information
}

stream, err := client.SubscribeBlocksLightweight(ctx, req)
if err != nil {
	log.Fatalf("error subscribing to lightweight blocks : %v", err)

}

for {
	resp, err := stream.Recv()
	if err == io.EOF {
		break
	}

	if err != nil {
		log.Fatalf("error receiving block lightweight: %v", err)
	}

	log.Printf("received lightweight block with ID: %x, Height: %d, parent block ID: %x and Timestamp: %s",
		resp.GetBlockId(),
		resp.GetBlockHeight(),
        resp.GetParentBlockId(),
        resp.GetBlockTimestamp().String(),
	)
}
```

### SendAndSubscribeTransactionStatuses

Streaming of a transaction and status changes

This would be great. Something like a SendAndSubscribe endpoint where you submit a tx, and the status is streamed back until the block the tx was included in is sealed.

- **TransactionMsg** (transaction that will be submited to stream status)

Usage example:

```go
txMsg, err := transactionToMessage(tx)
if err != nil {
	log.Fatalf("error converting transaction to message: %v", err)
}

req := &executiondata.SendTransactionRequest{
	Transaction: txMsg,
}

stream, err := client.SendAndSubscribeTransactionStatuses(ctx, req)
if err != nil {
	log.Fatalf("error subscribing to transaction statuses : %v", err)

}

for {
	resp, err := stream.Recv()
	if err == io.EOF {
		break
	}

	if err != nil {
		log.Fatalf("error receiving transaction status: %v", err)
	}

	txResult := resp.GetTransactionResult()

	log.Printf("received transaction status with tx ID: %x, status: %s, and error: %v",
		resp.GetTransactionID(),
		txResult.Status.String(),
        txResult.Error,
	)
}
```

### SubscribeAccountStatuses - ?

Streaming of an account state changes

This should give possibility to subscribe to account and track statuses changed (Created, Updated)

- **Address** (address of account to stream status changes)

Usage example:

```go
req := &executiondata.SubscribeAccountStatusesRequest{
	Address: "123456789abcdef0",
}

stream, err := client.SubscribeAccountStatuses(ctx, req)
if err != nil {
	log.Fatalf("error subscribing to account statuses : %v", err)

}

for {
	resp, err := stream.Recv()
	if err == io.EOF {
		break
	}

	if err != nil {
		log.Fatalf("error receiving account status: %v", err)
	}

	// ????? 
}
```

### SubscribeResourcesMovement - ?

Streaming of resource movement by type

https://github.com/onflow/flow-go-sdk/blob/master/examples/storage_usage/main.go

### SubscribeResourcesChanges - ?

Streaming of resource changes

these would be awesome. I think there’s some R&D(Research and Development) work needed before it’s possible but definitely valuable. Especially the account state changes as bluesign noted.

### SubscribeTransactionResults - ?

Streaming of transaction results starting from the provided block ID or height, and /or collection ID and/or transaction ID. The result should consist of an array of TransactionResult.

This sounds like effectively a streaming version of GetTransactionResultsByBlockID.

I can see some value in searching for transaction ID, where the behavior could be to return the result when the tx is executed/sealed. We’d have to explore this a bit more. I think there may be some open questions for this (e.g. what block is passed in to start? when does it stop searching? what happens if the tx is never finalized?). Much of that would be simplified in an endpoint like the next one.
