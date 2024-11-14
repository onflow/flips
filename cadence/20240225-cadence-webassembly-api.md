---
status: proposed
flip: 255
authors: darkdrag00n (darkdrag00n@proton.me), Bastian Müller (bastian.mueller@flowfoundation.org)
updated: 2024-11-13
---

# FLIP 255: Cadence WebAssembly API

## Objective

This FLIP proposes the addition of an API to allow the execution of  WebAssembly programs in Cadence.

## Motivation

More and more projects are putting non-trivial computationally expensive logic in Cadence.
Examples of this are games or zero-knowledge proof checks.
Developers would also like to utilize code written in other programming languages.

Support for WebAssembly in Cadence could enable such workloads.
For example, same way Python developers often import numerical libraries written in C,
Cadence developers might also want to import such and other libraries into Cadence.

The motivation is explained in further details in
[this forum post](https://forum.flow.com/t/idea-wasm-execution-engine-in-cadence/5164) by Dieter Shirley.

## User Benefit

Adding support for WebAssembly to Cadence has several benefits.

First, it allows developers to implement and speed up computationally expensive pure logic calculations,
which can be efficiently executed in WebAssembly.

Developers will also benefit from using existing non-Cadence programs, such as Rust or C,
by compiling such code to WebAssembly.

## Proposal

### Scope

This proposal targets a first, small iteration on adding a WebAssembly API to Cadence,
mostly focusing on allowing developers to utilize the algorithmic throughput provided by WebAssembly.

The scope of this proposal is as follows:

1. Allow Cadence programs to instantiate a WebAssembly module.
2. Allow Cadence programs to call functions exported from the WebAssembly module.

The WebAssembly module must:
- Not define any imports, i.e. the WebAssembly module has no access to Cadence,
  and other imports such as memories, globals, and tables are not supported
- Only export functions, i.e. other exports such as memories, globals, and tables are not supports
- Use the functionality defined in the [WebAssembly Core Specification 1.0](https://www.w3.org/TR/wasm-core-1/).
  Other extensions, such as the following, but not limited to, are not supported:
    - Bulk memory
    - Sign extension
    - Multi value
    - Multi memory
    - 64-bit memor
    - Mutable globals
    - Reference types
    - Tail calls
    - Extended const
    - Saturating float to int
    - Atomics
    - SIMD
- NaNs need to be canonicalized

Function exports must only have argument and return value types `i32` or `i64`.
Floats (`f32` or `f64`) are not supported.

### Design

The WebAssembly API will be implemented as a Cadence contract and will expose functions and structs to use them.

The API is inspired by the [WebAssembly JavaScript Interface](https://www.w3.org/TR/wasm-js-api-2/#webassembly-namespace).

There are two main structs that will be exposed via the contract:

1. `InstantiatedSource`: Represents a WebAssembly module and its instance.
   The instantiated source only provides access to the instance.
   The WebAssembly module cannot be introspected.

2. `Instance`: Represents an instance of an instantiated WebAssembly module.
   The instance only provides access to the exported functions.

The `InstantiatedSource` can be created using the `compileAndInstantiate` function.
It takes a WebAssembly module in binary format and compiles and instantiates it.

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

Example usage of the WASM API to load a simple `add(i32, i32) -> i32` function,
calling it with Cadence `Int32` values, and getting back the result as a Cadence `Int32`:

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

#### Implementation

An implementation of this proposal will likely use an existing WebAssembly execution environment
that satisfies the requirements.

#### Metering

Like regular Cadence programs,
code executed and the memory consumed by the WebAssembly execution environment must be limited and metered.

##### Computation

The specifics of exactly how computation usage is limited and metered is an implementation detail.

Popular WebAssembly execution environments support some form to limit and meter the amount of executed code.
Such a mechanism is required and will be used to implement the computation metering.
The mechanism must be compatible and integrate with the metering of the FVM.

The high level idea of computation metering is:
1. The FVM needs a mechanism to provide the number of remaining computation units to Cadence.
2. Computation units are converted to the equivalent of the WebAssembly execution environment equivalent ("engine units").
   For example, the wasmtime engine has the concept of "fuel", and has a cost associated with each executed instruction.
3. The WebAssembly code is executed with an upper limit of engine units determined from the available FVM computation units.
4. If the execution exceeds the limit, execution is aborted and the Cadence program is aborted.
5. If the execution succeeds, the amount of engine units is converted back to FVM computation units and metered.

##### Memory

The specifics of exactly how memory usage is limited and metered is an implementation detail.

To limit the memory being consumed, some constraints will be defined
on the various ways that memory is used within the WebAssembly execution environment.

While the specifics are not defined in this proposal,
the following aspects will be limited and metered:
1. Stack size/depth
2. Linear memory (number of WebAssembly pages)
3. Number and size of tables

### Drawbacks

None

### Alternatives Considered

None

### Performance Implications

None

### Dependencies

None

### Engineering Impact

It would require 3-4 weeks of engineering effort to implement, review & test the feature.

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
