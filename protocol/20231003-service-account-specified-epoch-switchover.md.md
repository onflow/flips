---
status: draft 
flip: 204 (set to the issue number)
authors: Jordan Schalm (jordan@dapperlabs.com)
sponsor: Jordan Schalm (jordan@dapperlabs.com)
updated: 2023-10-03
---

# FLIP 204: Smart-Contract-Specified Epoch Switchover Timing

## Objective

- Increase robustness of Cruise Control System
- Create an explicit target time for epoch switchover, defined by the service account

## Motivation

[Cruise Control: Automated Block Rate & Epoch Timing (Design)](https://www.notion.so/Cruise-Control-Automated-Block-Rate-Epoch-Timing-Design-4dbcb0dab1394fc7b91966d7d84ad48d?pvs=21) defines an existing system for controlling system block production to achieve a target *block rate*, in turn to achieve a target *epoch switchover time*. This system has been deployed on Mainnet since May.

At the time of writing, the target epoch switchover time is inferred based on a baked-in assumption of week-long epochs, and a configurable weekly switchover time. Therefore, each node’s **Process Variable** (switchover time) is determined by a heuristic, which has several downsides: 

- In extreme edge cases (timing off by several days), different nodes may disagree about the target Process Variable value.
- Networks with different-length epochs (Canary, Testnet) can not use Cruise Control at all.

## User Benefit

- Chain-queriable target epoch switchover time
- Increased robustness of Cruise Control System (reliable and consistent epoch and block timing)

## Design Proposal


- The `FlowEpoch` smart contract determines and broadcasts a `TargetEndTime` for each epoch, within the `EpochSetup` event.
- The `cruisectl.BlockTimeController` component reads this `TargetEndTime` and uses it as the Process Variable value for its PID controller, rather than the current heuristic method.

### `TargetEndTime` Definition

Below are two options for how to configure and compute the `TargetEndTime`. Overall the author is in favour of **Option 2.**


#### Option 1: Duration-Only

The configuration consists only of the epoch duration. Each epoch’s `TargetEndTime` is computed based on a reference time/view pair obtained via the `getBlock` API. 

```swift
pub struct EpochTimingConfig {
	duration: UInt64 // in seconds
}
```

```swift
// Compute the target switchover time based on the current time/view.
// Invoked when transitioning into the EpochSetup phase.
pub fun getTargetEndTimeForEpoch(
	curBlock: Block,
	epoch: EpochMetadata,
	config EpochTimingConfig,
): UInt64 {
	let now = curBlock.timestamp
	let viewsToEpochEnd = nextEpoch.finalView - curBlock.view
	let estSecondsToNextEpochEnd = UFix64(viewsToNextEpochEnd) / UFix64(nextEpoch.lengthInViews) * config.duration
	return UInt64(estSecondsToNextEpochEnd)
}
```

```swift
// Memorize the end time of each epoch.
// Invoked when transitioning into a new epoch.
pub fun memorizeEpochEndTime(curBlock: Block, epoch: EpochMetadata) {
	epoch.endedAt = curBlock.timestamp
}

// Compute the switchover time based on the last memorized reference timestamp.
pub fun getTargetEndTimeForEpoch(
	refEpoch: EpochMetadata,
	targetEpochCounter: UInt64,
	config EpochTimingConfig,
): UInt64 {
	return refEpoch.endedAt + config.duration * (targetEpochCounter-refEpoch.counter)
}
```

##### Pros

- Simpler configuration
- Does not require manual config changes to account for a durable switchover timing change.

##### Cons

- Drift can accumulate over time
- **Approach 1.2** Does not work well with `resetEpoch` process, as that involves an epoch transition at a non-target time
- Depends on block time API
- Approach 2 requires additional storage/logic in smart contract changes

#### Option 2: Duration & Reference Switchover

The configuration consists of the epoch duration and a reference counter/end-time pair. Each epoch’s `TargetEndTime` is computed solely based on the target epoch’s counter, the reference counter/end-time pair, and the duration.

```swift
pub struct EpochTimingConfig {
	duration: UInt64     // in seconds
	refCounter: UInt64   // the counter of a reference epoch
	refTimestamp: UInt64 // the UNIX timestamp (UTC) at which refCounter ended
}
```

```swift
// Compute target switchover time based on offset from reference counter/switchover.
pub fun getTargetEndTimeForEpoch(
	targetEpochCounter: UInt64,
	config EpochTimingConfig,
): UInt64 {
	return config.refTimestamp + config.duration * (targetEpochCounter-refCounter)
}
```

##### Pros

- Simple computation
- Drift cannot accumulate over time
- Does not use block time API
- Compatible with `resetEpoch` process

##### Cons

- More complex configuration specification
- Requires manual config changes for durable switchover time changes


### Implementation Plan

#### Smart Contract

- Add `targetEndTime` field to `EpochSetup` event, `EpochMetadata`
- Add config for determining `targetEndTime` to smart contract `ConfigMetadata`
- Add logic to compute `targetEndTime` to `startEpochSetup`
- Add function for service account to adjust new config
- Testing
    - Validate field is set as expected
    - Validate field is computed correctly
    - Validate setter/getter for new config values

#### Core Protocol

- Add `TargetEndTime` field to `EpochSetup` event, `Epoch` API
- Update `EpochSetup` service event conversion function
    - Read `TargetEndTime` field
    - Ensure conversion is backward-compatible
- Update `cruisectl.BlockRateController`
    - Remove `EpochTransitionTime` inference heuristic
    - Replace `EpochTransitionTime` with `time.Duration`, retrieved from `EpochSetup` event
- Add mechanism to set `TargetEndTime` in bootstrapping/sporking process
    - *Comment: currently Cruise Control is disabled by default.*
    - Option 1: Add an optional flag to explicitly the desired epoch duration (seconds). We can compute a reference counter/timestamp.
    - Option 2: Compute an initial `duration` config value, based on the committee size and epoch length and expected view rate.
- Update network instantiation
    - Default value for deploying `FlowEpoch`

### Deployment Plan

As usual, deploy to Canary → Testnet → Mainnet

1. Upgrade `FlowEpoch`
    
    <aside>
    ⚠️ Caution: First, ensure service event conversion logic is tolerant of additional fields (ignores additional fields)
    
    </aside>
    
2. Upgrade Consensus Nodes
    
    *Comment: Since we are modifying the EpochSetup model, this will likely require a spork.*
    


### Drawbacks

This proposal assumes that the service account is more reliable than the heuristic currently in use for determining target epoch switchover times. 
Many more important system functions already depend on correct operation of the service account. 
However, Cruise Control will become susceptible to faults in the service account's defined switchover time, rather than faults in the current heuristic.

### Alternatives Considered

See 2 options above.

### Performance Implications

None anticipated.

### Dependencies

There are internal dependencies that will need to be updated in lockstep as part of this FLIP. No external dependencies.

### Engineering Impact

* Do you expect changes to binary size / build time / test times?
* Who will maintain this code? Is this code in its own buildable unit? 
Can this code be tested in its own? 
Is visibility suitably restricted to only a small API surface for others to use?

### Best Practices

N/A.

### Tutorials and Examples

N/A.

### Compatibility

The change will not break compatibility.

### User Impact

N/A.

## Related Issues

N/A.

## Prior Art

N/A.

## Questions and Discussion Topics

- Do you prefer Option 1 or Option 2?
- Do you foresee any significant hurdles beyond those outlined in the Implementation Plan?

### Q&A
#### What happens during a `resetEpoch`?

In ****************Option 1.1**************** and **2**, the timing of a particular epoch transition does not affect the target timing for other epochs. Therefore, the `TargetEndTime` computation of an epoch during a spork will not behave differently from any other epoch.

#### Why do we set `TargetEndTime` rather than `TargetStartTime`?

The time information is specified in the `EpochSetup` event, which occurs partway through the current epoch. If we specified a `TargetStartTime`, then the PID Controller’s Process Variable would have an undefined value for part of the epoch, and the Cruise Control system would be unable to function. On Mainnet, this corresponds to about 90% of the duration of an epoch.