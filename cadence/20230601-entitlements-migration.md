# Entitlements Migration

## Problem Description

With the release of entitlements and reference downcasting in Stable Cadence, existing references and capabilities will need to be migrated to the new entitlement model in order to continue working. In particular, because fields and functions that were previously `access(all)` on interfaces and composites will no longer be `access(all)` after the change (because this will no longer be safe), in order to maintain the same functionality that they previously had, existing references and Capabilities will need to be granted a set of entitlements that allows them to call the same set of members that were previously accessible to them.  

The challenge here is that it is not possible to determine what entitlements these might be just by scanning the code that exists today, as we can’t know which entitlements a user might create when migrating their contract, nor which entitlements will be assigned to which function. 

## Proposed Solution

NB: this solution has since been updated and iterated upon slightly to address issues that came up during its implementation. See https://cadence-lang.org/docs/cadence_migration_guide/type-annotations-guide for more details. 

### `Entitlements(T)`

The first part of this solution involves the definition of an informal “function” on types, called `Entitlements`. For some type `T` that exists in pre-Stable Cadence, `Entitlements(T)` expresses the set of entitlements necessary post-Stable Cadence contract update that are necessary to call the same set of members that were previously accessible to a reference typed `&{T}`.

The function is defined as follows: 

- For some interface or composite type `I`, `Entitlements(I)` is the set of all entitlements that appear in the access modifiers of any of `I`'s members. E.g. for some `I` defined like so:
    
    ```jsx
    resource interface I {
        access(X | Y) fun foo() 
        access(E, F) fun bar()
        access(contract) let x: Int
    }
    ```
    
    `Entitlements(I)` would be `{X, Y, E, F}`, all of the entitlements in the access modifiers of `I`'s members. 
    
- For some array or dictionary type `[T]` or `{K: V}`, `Entitlements([T])` would be `{Mutable}`, as would `Entitlements({K: V})`, assuming the behavior described in the current [entitlements mutability proposal](https://www.notion.so/External-Mutability-49baf90d70094ab0a692c0bb0ebb6706).
- For each entitlement mapping `M` used on a field in `I`, `Entitlements(I)` includes `Domain(M)`. To see why, consider that `access(M)` is effectively syntactic sugar for multiple member definitions for each element in the domain of `M`. E.g. the following two examples have effectively equivalent behavior:
    
    ```jsx
    // with mappings
    entitlement mapping M {
       X -> Y
       X -> Z
       E -> F
    }
    resource interface I {
        access(M) fun foo(): auth(M) &T {}
    }
    
    // without mappings
    resource interface I {
        access(all) fun foo0(): &T {}
        access(X) fun foo1(): auth(Y, Z) &T {}
        access(E) fun foo2(): auth(F) &T {}
    }
    ```
    
    The definition of `Entitlements(I)` effectively “desugars” the entitlement mapping `M`, breaking it into its implicit parts, and includes each of the access modifiers of those parts, hence the entire domain of `M`. 
    

Given this definition of `Entitlements(I)`, we can see that an `auth(Entitlements(I)) &{I}` reference in a post-Stable Cadence world has the same “permissions” that a `&{I}` reference had in a pre-Stable Cadence world, in that it is able to access all members on `I` that are not `access(self)`, `access(contract)` or `access(account)`. 

### Migration

Using the definition above, the proposed migration is simply to migrate any existing reference types (including those that appear inside Capability types) like so:

```jsx
&T{I1, I2, ...} ---> auth(Entitlements(I1) ∪ Entitlements(I2) ∪ ...) &T{I1, I2, ...}
```

In a world where the FLIP proposing to remove restricted types is accepted, we can just drop the `T` here from the migrated reference type. 

Note, however, that using the `Entitlements` function does require that the updated contract be available for analysis, since we need to compute the set of entitlements required by each interface in order to compute the function. This means we have two options for when to apply this migration:

1. Simultaneously upon release and spork of Stable Cadence. This would require that programmers pre-load their contract upgrades to also be applied when Stable Cadence is released, and we could combine these contract upgrades with the storage migration to migrate the existing references of those contracts that were upgraded. The downside here is that any users who miss out on their window to pre-submit a contract upgrade will be unable to fix their broken reference values.
2. Dynamically upon the load of a reference. This operates on the assumption that every contract will break when Stable Cadence is released, making existing references that use a contract’s types non-interactable until that contract is upgraded. Once it is upgraded, however, the first time that previously broken references is loaded, we can dynamically change the type from the old type to the new type computed as described above. 

### Caveats

The success of this migration depends on a key assumption: developers will update their contracts for Stable Cadence only as much as necessary to recreate the previous behavior before the update. This assumption is what makes it safe for us to grant existing references full entitlements to their interfaces, as we can assume that all the entitlements added to any interface are present to recreate the previous access control behavior where they were `pub`.

If this is not the case, e.g. if a developer adds some `Admin`-entitled member to an interface `I` that was previously not present on `I` and thus was not previously accessible to someone with an `&{I}` reference, this migration would grant holders of an `&{I}` reference access to the `Admin` entitlement. This may not be desirable. Some potential mitigations of this:

1. Just encourage people not to change their contract’s behavior during their first update. This way our assumption can hold true in more cases. However, this is fragile, as it assumes developers won’t make implementation mistakes, and also is incompatible with the dynamic type-changing option, as any future changes that do change contract behavior would need to wait until all existing references are updated in order to be safe. 
2. Ignore member definitions that didn’t previously exist as `pub` members when computing `Entitlements(I)`. This way we would only grant access to exactly the same set of functions as previously existed on `I`. The issue with this is that access to the same set of functions does not necessarily imply access to the same functionality. E.g. given some composite `I` defined like so:
    
    ```jsx
    resource I {
       pub let x: T
    }
    ```
    
    that was migrated to:
    
    ```jsx
    resource I {
       priv let x: T
       access(E) getX(): auth(F) &T {
          // ...
       }
    }
    ```
    
    This restriction would mean `E` is not included in `Entitlements(I)`, and thus existing `&{I}` references would be migrated in such a way that the `x` value would no longer be accessible. 
    
3. Allow users to specify a `#no-migrate-entitlement` (or similar name TBD) pragma on a member in their composites and interfaces that would cause a member’s entitlements to be ignored during the computation of `Entitlements(I)`. This would allow the computation of `Entitlements` to find all the necessary entitlements to recreate `I`'s functionality, but would also allow users to manually exclude certain entitlements (like the `Admin` in the original example). This relies on the user to correctly use this pragma, but is otherwise the best option because it will not implicitly break references without a user’s input. 

## Meeting Feedback

We had a breakout session for this on June 1st, and there were two primary takeaways:

- Developers should have a way to see when values are migrated and exactly what the migration that is applied to them is. This could be accomplished by having the migration be explicitly outlined in the code or the protocol somehow, but it might also be sufficient to emit an event whenever a value is migrated, specifying what the old type was and what the new type is.
- The proposed automatic solution is sufficient for the cases where developers wish to simply grant owners of their values the same authority they had before, but is not sufficient when they wish to grant less or more than that. However, these use cases are not likely to be something that can be handled by any kind of migration; the previous access control method using interfaces was more coarse than the entitlements-based method, and as such there may not be any possible mapping, automatically inferred or otherwise, that can disambiguate between two references with the same type that should end up with different entitlements. We can instead suggest that if developers wish to grant entitlements to users in addition to the ones that they receive just to replicate old functionality, they can do so by granting new capabilities. This is safer than having this additional functionality granted by default because it prevents users from being automatically granted any functionality beyond what they already had.