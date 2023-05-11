---
status: proposed
flip: 86
authors: Supun Setunga (supun.setunga@dapperlabs.com)
sponsor: Supun Setunga (supun.setunga@dapperlabs.com)
updated: 2023-05-19
---

# Introduce Built-in Mutability Entitlements

## Objective

The objective of this proposal is to introduce a set of built-in entitlements to control mutable operations on arrays,
dictionaries and composite types.

## Motivation

A previous version of Cadence ("Secure Cadence") restricted the potential foot-gun of mutating container-typed
`let` fields/variables via the
[Cadence mutability restrictions FLIP](https://github.com/onflow/flips/blob/main/cadence/20211129-cadence-mutability-restrictions.md).

However, there are still ways to mutate such fields by:
- Directly mutating a nested composite typed field.
- Mutating a field by calling a mutating function on the field.
- Mutating the field via a reference.

More details on this problem is described in the [FLIP for improving mutability restrictions](https://github.com/onflow/flips/pull/58).

With an aim of moving one step closer to solving these existing problems, another FLIP was proposed previous to this, to
[change member access semantics](https://github.com/onflow/flips/pull/89) for container types.

Going another step further, this FLIP proposes introducing a set of built-in entitlements, that can be used along with
the change of member-access semantics, to better control who can perform mutating operations through the references.

## User Benefit

The changes suggested in this proposal eliminate the potential for users to accidentally expose mutable fields through
references.

## Design Proposal

Before going into the details of the proposal, it is important to note that these proposed changes depend on the
[change to member-access semantics](https://github.com/onflow/flips/pull/89) and the introduction of
[Entitlements](https://github.com/onflow/flips/pull/54).

The main change proposed in this FLIP is to introduce a set of built-in entitlements:
  - `Insertable`
  - `Removable`
  - `Mutable`

<!--
Having:

```cadence
struct Foo {
    access(Insertable, Removable) fun someFunction()
}

var fooRef: auth{Insertable, Removable} Foo = ...
```

is equivalent to:

```cadence
struct Foo {
    access(Mutable) fun someFunction()
}

var fooRef: auth{Insertable, Removable} Foo = ...
```
-->

Along with that, built-in array/dictionary mutating functions such as `append`, `remove`, etc., would get a
corresponding entitlement access, instead of the `pub`/`access(all)` access that they currently possess.
Thus, only a reference (to an array/dictionary) that has the correct entitlement can call such methods.

### Arrays

Array functions would now have the below entitlements:

- `access(all) fun contains(_ element: T): Bool`
- `access(all) fun firstIndex(of: T): Int?`
- `access(all) fun slice(from: Int, upTo: Int): [T]`
- `access(all) fun concat(_ array: T): T`
- `access(Mutable | Insertable) fun append(_ element: T): Void`
- `access(Mutable | Insertable) fun appendAll(_ element: T): Void`
- `access(Mutable | Insertable) fun insert(at: Int, _ element: T): Void`
- `access(Mutable | Removable) fun remove(at: Int): T`
- `access(Mutable | Removable) fun removeFirst(at: Int): T`
- `access(Mutable | Removable) fun removeLast(at: Int): T`

Thus, for any arbitrary array:

```cadence
var arrayRef = &array as &[String]
arrayRef.contains("John")          // OK
arrayRef.append("John")            // Static Error: doesn't have the required entitlement
arrayRef.remove(2)                 // Static Error: doesn't have the required entitlement

var insertableArrayRef = &array as auth(Insertable) &[String]
insertableArrayRef.contains("John") // OK
insertableArrayRef.append("John")   // OK
insertableArrayRef.remove(2)        // Static Error: doesn't have the required entitlement

var removableArrayRef = &array as auth(Removable) &[String]
removableArrayRef.contains("John")  // OK
removableArrayRef.append("John")    // Static Error: doesn't have the required entitlement
removableArrayRef.remove(2)         // OK

var mutableArrayRef = &array as auth(Mutable) &[String]
mutableArrayRef.contains("John")   // OK
mutableArrayRef.append("John")     // OK
mutableArrayRef.remove(2)          // OK
```

### Dictionaries

Dictionary functions would now have the below entitlements:

- `access(all) fun containsKey(key: K): Bool`
- `access(all) fun forEachKey(_ function: ((K): Bool)): Void`
- `access(Mutable | Insertable) fun insert(key: K, _ value: V): V?`
- `access(Mutable | Removable) fun remove(key: K): V?`

Thus, for any arbitrary dictionary:

```cadence
var dictionaryRef = &dictionary as &{String:AnyStruct}
dictionaryRef.containsKey("John")               // OK
dictionaryRef.insert("John", "Doe")             // Static Error: doesn't have the required entitlement
dictionaryRef.remove("John")                    // Static Error: doesn't have the required entitlement

var insertableDictionaryRef = &dictionary as auth(Insertable) &{String:AnyStruct}
insertableDictionaryRef.containsKey("John")     // OK
insertableDictionaryRef.insert("John", "Doe")   // OK
insertableDictionaryRef.remove("John")          // Static Error: doesn't have the required entitlement

var removableDictionaryRef = &dictionary as auth(Removable) &{String:AnyStruct}
removableDictionaryRef.containsKey("John")      // OK
removableDictionaryRef.insert("John", "Doe")    // Static Error: doesn't have the required entitlement
removableDictionaryRef.remove("John")           // Ok

var mutableDictionaryRef = &dictionary as auth(Mutable) &{String:AnyStruct}
mutableDictionaryRef.containsKey("John")        // OK
mutableDictionaryRef.insert("John", "Doe")      // OK
mutableDictionaryRef.remove("John")             // OK
```

### Assignment

Similar to functions, the assignment operator for arrays/dictionaries would also have the `Mutable` and `Insertable`
entitlements.
Think of assignment as a built-in function with `Mutable` and `Insertable` entitlements. 
e.g:
```cadence
access(Mutable | Insertable) set(keyOrIndex, value) { ... }
```

Thus, for any arbitrary array:

```cadence
var arrayRef = &array as &[String]
arrayRef[2] = "John"               // Static Error: updating via a read-only reference

var mutableArrayRef = &array as auth(Mutable) &[String]
mutableArrayRef[2] = "John"        // OK

var insertableArrayRef = &array as auth(Insertable) &[String]
insertableArrayRef[2] = "John"     // OK

var removableArrayRef = &array as auth(Removable) &[String]
removableArrayRef[2] = "John"     // Static Error: doesn't have the required entitlement
```

This would behave the same way for dictionaries as well.

### Putting It All Together

Now let's see how this solves the mutability foot-gun. 
Let's consider the `NonFungibleToken.Collection` example:

```cadence
pub resource Collection: NonFungibleToken.Collection {

    // Requirement: This field must be read-only for outsiders.
    //
    pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}

    ...
}
```

The original intention of the `Collection` resource is to make `ownedNFTs` readable to outsiders, but must not be mutable.

```cadence    
// `collection.ownedNFTs` result in a `&{UInt64: NonFungibleToken.NFT}`
var ownedNFTsRef: &{UInt64: NonFungibleToken.NFT} = collection.ownedNFTs
```

The field `collection.ownedNFTs` is readable since it is `pub` access modifier.
It is not mutable since it has no entitlements.

```cadence 
var len = ownedNFTsRef.length         // OK: read only operation

ownedNFTsRef["someNFT"] <- someNft    // Static Error: `append` needs `Mutable` entitlement
```

#### Making the Field Mutable

Assume the author wanted to make `ownedNFTs` field mutable for some reason.
Given that field-access now always returns a non-auth reference, now the author needs to introduce a function
to return an authorized reference.

```cadence
pub resource Collection: NonFungibleToken.Collection {

    pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}

    // Provide a function that returns a mutable reference.
    pub fun getMutableNFTs() auth(Mutable) &{UInt64: NonFungibleToken.NFT} {
        return &self.ownedNFTs as auth(Mutable) &{UInt64: NonFungibleToken.NFT}
    }
}
```

Now the mutable operation is possible via the `auth(Mutable)` reference,

```cadence
// Get a mutable reference
var ownedNFTsRef = collectionRef.getMutableNFTs()

// `ownedNFTsRef` is a dictionary reference with auth{Mutable} entitlement.

var len = ownedNFTsRef.length        // OK

ownedNFTsRef["someNFT"] <- someNft   // OK
```

### Drawbacks

It is now possible to have a function that modifies the receiver, but is declared without the `Mutable` entitlement
requirements.

```cadence
pub resource Collection: NonFungibleToken.Collection {

    pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}

    pub fun delete(_ id: UInt64) {
        destroy ownedNFTs.remove(key: id)
    }
}
```

Assume the author set the access modifier of the `delete` method to be `pub` instead of `access(Mutable)`.
Now anyone with an un-entitled reference to `Collection` can modify `ownedNFTs`.

However, this is in a way a good thing to have, and eliminates the problem of the previous 
[FLIP proposed to eliminate the mutability foot-gun](https://github.com/onflow/flips/pull/58), 
where having to annotate each and every mutating function could quickly become an overkill. 
Also, certain mutating operations are in-fact safe.
For example: `deposit()` function of `Collection` does mutate the object, however, it is of no harm of anyone calling
the `deposit()` function.

### Alternatives Considered

There were three alternatives considered for the mutability problem:

#### 1) Introducing additional static checks preventing mutations

FLIP: https://github.com/onflow/flips/pull/58

The drawback was that the added checks makes it too restrictive, and having to mark each function that modifies the
receiver value as 'mutating' seems to exponentially explode in the code.

However, some of the proposed changes in current FLIP resembles to the above-mentioned FLIP, specially with the
'mutable' references concept.

On contrary, the notable difference is, not all mutating functions needs to be marked as mutating.
It is upto the author of the function to determine whether the function needs to be exposed/restricted as a
mutating operation using entitlements.

Comparing using the same example:

```cadence
pub resource Collection: NonFungibleToken.Collection {

    pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}

    // Provide a function that returns a mutable reference.
    pub fun getMutableNFTs() &{UInt64: NonFungibleToken.NFT} {
        return &self.ownedNFTs as &{UInt64: NonFungibleToken.NFT}
    }
}
```

Then, on the usage:

```cadence
// Directly updating is rejected.
collection.ownedNFTs["someNFT"] <- someNft    // Static Error

// Taking a mutable reference is rejected.
var ownedNFTsMutableRef = &collection.ownedNFTs as &{UInt64: NonFungibleToken.NFT}    // Static Error

// Taking immutable reference is OK, but updating via immutable ref is rejected.
var ownedNFTsImmutableRef = &collection.ownedNFTs as immutable &{UInt64: NonFungibleToken.NFT}  // OK
ownedNFTsImmutableRef["someNFT"] <- someNft                                                     // Static Error

// Updating via a mutable reference is OK.
var ownedNFTsRef = collectionRef.getMutableNFTs()
ownedNFTsRef["someNFT"] <- someNft    // OK
```

#### 2) Restricting usage of `let` fields

FLIP: https://github.com/onflow/flips/pull/59

```cadence
pub resource Collection: NonFungibleToken.Collection {

    priv var(set) ownedNFTs: @{UInt64: NonFungibleToken.NFT}

    // Provide functions to expose functionalities to the inner dictionary.
    pub fun size() Int {
        return self.ownedNFTs.length
    }
}
```

Drawback is author have to decide/anticipate what information to be exposed via delegator functions.
These delegator functions may add a lot of boilerplate code.


#### 3) Keeping the current behavior
Considering the added complexity of the proposed solutions, and considering that some of these solutions may only
'hide' the problem rather than completely solving them, keeping the current behavior and properly documenting it is
a possible alternative solution.

### Performance Implications

None

### Dependencies

- [Entitlements FLIP](https://github.com/onflow/flips/pull/54)
- [Changing member-access semantics FLIP](https://github.com/onflow/flips/pull/89)

### Engineering Impact

This change has medium complexity in the implementation.

### Compatibility

This is a backward incompatible change.

### User Impact

Almost all deployed contracts might be affected by this change.

## Related Issues

None

## Prior Art

None

## Questions and Discussion Topics

The naming (should be a verb/noun/adjective?) and the styling (should be camel-case/lower-case?) of the three
entitlements need to be discussed and finalized.
