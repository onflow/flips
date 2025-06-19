---
status: draft 
flip: 314
authors: Raymond Zhang (raymond.zhang@flowfoundation.org)
sponsor: Supun Setunga (supun.setunga@flowfoundation.org)
updated: 2024-12-17
---

# FLIP 314: Import Aliasing

## Objective

The goal is to allow developers to import two contracts with the same name from different addresses (e.g. `FUSD` from `0x1` and `FUSD` from `0x2`). This would reduce code size as well as improve composibility in general.

## Motivation

From previous discussions:
- https://github.com/onflow/Flow-Working-Groups/blob/main/cadence_language_and_execution_working_group/meetings/2024-12-10.md#survey-feedback
- https://github.com/onflow/cadence/issues/1171

There can be two contracts operating on separate pools (e.g. `FlowUSDT` vs `FUSDUSDT`) which each expose a contract `UniswapPair`. In current contracts or transactions, developers cannot reference both these pools in the same code. There are workarounds, but these involve duplication of code. An additional use case is to shorten the name of imported types.

## User Benefit

This will reduce code duplication, code size and help developers avoid conflicts.

## Design Proposal

The proposed syntax is an `import ... as` alias. Consider the following example

```cdc
import FUSD as FUSD1 from 0x02
import FUSD as FUSD2 from 0x03

transaction {
  prepare(acct: AuthAccount) {
    let typeFUSD = Type<FUSD1>()
    let typeKibble = Type<FUSD2>()
    log(typeFUSD == typeKibble)
    log(typeFUSD.identifier)
    log(typeKibble.identifier)
  }
}
```

### Drawbacks

This could lead to more obfuscation, for example purposefully confusing aliases.

### Performance Implications

None.

### Dependencies

None.

### Engineering Impact

This change should be simple to implement.

### Compatibility

Feature addition, no impact.

### User Impact

Feature addition, no impact.

## Related Issues

Consider extension for type aliasing in the future. 

## Implementation
https://github.com/onflow/cadence/pull/4033

## Prior Art

- [JS](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import)
- [Python](https://docs.python.org/3/reference/simple_stmts.html#import)

