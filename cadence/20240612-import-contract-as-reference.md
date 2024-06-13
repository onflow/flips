---
status: proposed
flip: 277
authors: Supun Setunga (supun.setunga@flowfoundation.org)
sponsor: Supun Setunga (supun.setunga@flowfoundation.org)
updated: 2024-06-11
---

# FLIP 134: Import contracts as references

## Objective

Change the import statement semantics to import a reference to the contract, instead of importing the concrete value.
Thus, accessing the fields of a contract externally would return references to fields, preventing any unintended external mutations.

## Motivation

A huge thanks goes to Deniz Mert Edincik (@bluesign) for pointing out the below security foot-gun in the existing design.

In the [external mutability FLIP](https://github.com/onflow/flips/pull/89), the behaviour for accessing fields of a
composite value was changed, to prevent external-mutation foot gun.
You can read more about the overall vision [here](https://github.com/onflow/flips/blob/main/cadence/vision/mutability-restrictions.md).
The essence of the change is that, if someone owns the composite value, they would have the full access (read/write/mutate) to its fields.
Otherwise, if someone only got a reference to the composite value, then they can access the fields also as references,
and the entitlements would kick-in to limit what they can do with those fields.

Unfortunately, one edge-case that was not considered during the evaluation of the [external mutability improvements FLIP](https://github.com/onflow/flips/pull/89) is that contracts are singletons, and importing a contract provides owned access to that contract value.
The imported contract value behaves like a reference, but is not represented using a reference in the language semantics. 
Because of that, if the contract has a container-typed field (e.g., array, dictionary, composite)  defined with public access (i.e., `access(all)`),
then anyone could modify the content of that field, such as inserting/removing elements, etc.

```cadence
access(all) contract Foo {
    access(all) var array : [Int] // Can't set a new value to the array, but can "update" the content (insert/remove/etc) 
}
```

Here, the author of the contract may have intended it to be "read-only", however, it is not truly read only.

```cadence
import Foo from 0x1

access(all) fun main() {
    Foo.array[0] = 3
    Foo.array.append(42)
    Foo.array.remove(at: 0)
}
```

This is exactly the type foot-gun that was intended to get prevented by the changes proposed in the [external mutability improvements FLIP](https://github.com/onflow/flips/pull/89).
But unfortunately, given the contract import results in a non-reference value, the suggested changes in that FLIP
do not apply here.

## User Benefit

Prevent the foot-gun of unintended external mutations to read-only contract fields.

## Design Proposal

The proposal is to change the import semantics to import contracts as a reference, instead of the concrete value.

```cadence
import Foo from 0x1

access(all) fun main() {
    var foo: &Foo = Foo  // The type of the imported contract value `Foo` will be `&Foo`
}
```

Then, accessing a container typed field will also result in a reference (because of the changes added in [FLIP 89](https://github.com/onflow/flips/pull/89)),
limiting the operation that can be performed on them.
For example, mutating the `Foo.array` would result in an error.

```cadence
import Foo from 0x1

access(all) fun main() {
    var array: &[Int] = Foo.array  // Accessing field would also return a reference.

    // All of the below operation would be statically rejected,
    // since `Foo.array` would be an unauithorized reference.
    Foo.array[0] = 3
    Foo.array.append(42)
    Foo.array.remove(at: 0)
}
```

One thing to note is that, this proposal would **not** change the way the contract value is accessed within the contract itself.

```cadence
contract Foo {
    access(all) array: [Int]

    init() {
        self.array = []
    }
    
    access(all) fun bar() {
        // Accessing the contract within itself would return the concrete value, not a reference.
        
        var foo: Foo = Foo  // Type the contract value is `Foo` (not `&Foo)`
        
        var array: [Int] = Foo.array  // Accessing field would also return a copy of the concrete value.
        
        // Modifying fields within the contract is allowed.
        Foo.array[0] = 3
        Foo.array.append(42)
        Foo.array.remove(at: 0)
    }
}            
```
### Drawbacks

This is a breaking change. Certain users may have to update their code.

### Alternatives Considered

A non-breaking alternative solution is to keep current language semantics as-is, and warn about contract fields that
has `access(all)` access-modifier, through a tooling like the linter.
However, this doesn't guarantee the foot-gun would be fixed.

### Performance Implications

None

### Dependencies

None

### Engineering Impact

This change is trivial in the implementation.

A draft implementation can be found at: https://github.com/onflow/cadence/pull/3417

### Compatibility

This is a breaking change.

### User Impact

This would impact the Cadence 1.0 staged contracts. I ran an analyzer for the updated core contracts and the
staged-contracts, and as of 12th June 2024, only three (3) contracts out of 456 are impacted by this change.
The changes needed for those contracts are fairly simple, and would only require changing a couple of lines of code.

## Related Issues

None

## Prior Art

None

## Questions and Discussion Topics

None

