---
status: draft
flip: 255
authors: darkdrag00n (darkdrag00n@proton.me)
sponsor: Bastian Müller (bastian@dapperlabs.com)
updated: 2024-02-25
---

# FLIP 255: Cadence Webassembly API

## Objective

This FLIP proposes addition of a WebAssembly (WASM) execution engine API to Cadence.

## Motivation

More and more projects are putting non-trivial computationally expensive logic in Cadence. Examples of this are games or zero-knowledge proof checks. 

WASM could offer a great algorithmic throughput for such workloads by providing a way to offload the pure logic calculation to it without exposing Resource objects to it. 

The motivation is explained in further details in [this forum post](https://forum.flow.com/t/idea-wasm-execution-engine-in-cadence/5164) by Dete.

## User Benefit

Users can benefit from the raw algorithmic throughput of WASM by offloading computationally expensive pure logic calculations to it while still keeping the security benefits of Cadence.

## Proposal

### Scope

As mentioned earlier, the feature is targeted towards utilizing algorithmic throughput provided by WASM. So the scope of this proposal is as follows:

1. Allow instantiating a WebAssembly module, without imports (i.e. the WebAssembly module has no access to Cadence)
2. Function exports with argument and return value types as i32 or i64. Floats (f32 or f64) will not be supported
3. The function exports could be called from Cadence, and receive the results back

### Design

The WebAssembly API will be implemented as a Cadence contract and will expose functions and structs to use them.

There are two main structs that will be exposed via the contract:
1. InstantiatedSource: Represents a source that has been compiled and instantiated via the Webassembly module.
2. Instance: Represents an instance of an instantiated webassebmly module that can be used to get the exports and use them in Cadence code.

The `InstantiatedSource` can be created using the `compileAndInstantiate` function which takes the WebAssembly binary code and compiles it into a Module.

```cadence
access(all)
contract WebAssembly {

    /// Compile WebAssembly binary code into a Module 
    /// and instantiate it. Imports are not supported.
    access(all)
    view fun compileAndInstantiate(bytes: [UInt8]): &WebAssembly.InstantiatedSource

    access(all)
    struct InstantiatedSource {

        /// The instance.
        access(all)
        let instance: &WebAssembly.Instance
    }

    struct Instance {

        /// Get the exported value.
        access(all)
        view fun getExport<T: AnyStruct>(name: String): T
    }
}
```

#### Example

Example usage of the WASM API to load a simple `add(i32, i32) -> i32` function, calling it with Cadence `Int32` values, and getting back the result as a Cadence `Int32`:

```cadence

access(all)
func main() {
    // A simple program which exports a function `add` with type `(i32, i32) -> i32`,
    // which sums the arguments and returns the result:
    //
    //  (module
    //    (type (;0;) (func (param i32 i32) (result i32)))
    //    (func (;0;) (type 0) (param i32 i32) (result i32)
    //      local.get 0
    //      local.get 1
    //      i32.add)
    //    (export "add" (func 0)))
    //
    let addProgram: [UInt8] = [
       0, 97, 115, 109, 1, 0, 0, 0, 1, 7, 1, 96,
       2, 127, 127, 1, 127, 3, 2, 1, 0, 7, 7, 1,
       3, 97, 100, 100, 0, 0, 10, 9, 1, 7, 0, 32,
       0, 32, 1, 106, 11,
    ]

    let instance = WebAssembly.compileAndInstantiate(bytes: program).instance
    let addFn = instance.getExport<fun(Int32, Int32): Int32>(name: "add")

    let sum1 = addFn(1, 2) // 3
    let sum2 = addFn(1, -100) // -99
}
```

#### Metering

TODO

### Drawbacks

None

### Alternatives Considered
Initial proposal considered defining operators such as `..` or `downTo` for defining `Range`. It also proposed adding another type named `Progression` for allowing non-default values of `step`. 

Due to the readability concerns associated with the inclusive vs exclusive behavior of the operators, it was considered better to have types and constructor functions with self-explanatory names to avoid ambiguity.

It was also proposed that since `Progression` is essentially a `Range` with non-default value of `step`, the two types can be merged into one.

### Performance Implications

None

### Dependencies

None

### Engineering Impact

It would require 2-3 weeks of engineering effort to implement, review & test the feature.

### Compatibility

This change has no impact on compatibility between systems (e.g. SDKs).

### User Impact

The proposed feature is a purely additive.
There is no impact on existing contracts and new transactions.

## Related Issues

None

## Questions and Discussion Topics

None

## Implementation
Will be done as part of https://github.com/onflow/cadence/issues/2853.

A proof of concept has already been done as part of https://github.com/onflow/cadence/pull/2760.
