# Change Cadence for-loop semantics

| Status        | Implemented                                          |
:-------------- |:---------------------------------------------------- |
| **FLIP #**    | [13](https://github.com/onflow/flips/pull/13)        |
| **Author(s)** | Bastian Müller (bastian@dapperlabs.com)              |
| **Sponsor**   | Bastian Müller (bastian@dapperlabs.com)              |
| **Updated**   | 2022-12-06                                           |

## Objective

This FLIP proposes a change of the semantics of for-loops:
Iteration variables should be defined for each loop iteration, instead of once for all iterations.

## Motivation

Currently, Cadence introduces one variable for the value, and one variable for the index (if given), for the whole loop.

Capturing these variables has surprising behaviour.
For example, this program repeatedly logs `3`, instead of the items of the array, as `3` is the last value assigned to the value variable:

```cadence
let fs: [((): Int)] = []
for x in [1, 2, 3] {
    fs.append(fun (): Int {
        return x
    })
}

for f in fs {
    log(f())
}
```

This has been a problem in many other languages. Some languages and users mostly agree changing the semantics is a good idea:

- [Go is planning to change their for-loop semantics](https://github.com/golang/go/discussions/56010)
- [C# already changed their for-loop semantics which caused little problems](https://github.com/golang/go/discussions/56010#discussioncomment-3788526)

Likely only few developers experience this surprising behaviour, which is usually discovered as a bug.
It is very unlikely that existing code relies on the current semantics.

Changing the semantics will affect few users and will avoid further potential bugs in developers' programs.

## User Benefit

Developers will benefit from this change by not getting surprised, and the change will likely reduce a source of potential bugs from the language.

## Design Proposal

The value and index variables of for-loops should be introduced fresh for each iteration, instead of once for all iterations.

### Drawbacks

Changing the semantics is a breaking change. However, few developers should be affected by it.

### Alternatives Considered

None

### Performance Implications

Performance will likely decrease and memory usage will increase.
However, the effects should be minimal.

### Dependencies

None

### Engineering Impact

This change is trivial in the implementation.

### Compatibility

This change has no impact on compatibility between systems (e.g. SDKs).

### User Impact

Existing contracts' and new transactions' behaviour will change.
Given that capturing variables is uncommon, this change should only affect few users' programs.

## Related Issues

None

## Prior Art

- [Go is planning to change their for-loop semantics](https://github.com/golang/go/discussions/56010)
- [C# already changed their for-loop semantics which caused little problems](https://github.com/golang/go/discussions/56010#discussioncomment-3788526)
- In JavaScript, for loops with `var` variables are function-scoped. When block scope variables (`let`) were introduced, their behaviour in for-loops was defined as per-iteration.

## Questions and Discussion Topics

None

## Implementation

An implementation is available in https://github.com/onflow/cadence/pull/2024
