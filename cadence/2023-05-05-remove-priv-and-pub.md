---
status: proposed 
flip: NNN (do not set)
authors: Daniel Sainati (daniel.sainati@dapperlabs.com)
sponsor: Daniel Sainati (daniel.sainati@dapperlabs.com)
updated: 2023-05-05 
---

# Remove `pub` and `priv` aliases for access modifiers

## Objective

This FLIP proposes to remove the `pub` alias for `access(all)` and the `priv`
modifier for `access(self)`, requiring all code to use these longer forms instead
of the shortened ones. 

This also proposes to remove the `pub(set)` access keyword, which was previously used to make
variables publicly settable. This is not a common case, and in order to simplify the language, 
we also propose to remove it. If users wish to have a field be publicly settable, they should write
a public mutator function for that field. 

## Motivation

The proposed entitlements changes in [the entitlements FLIP](https://github.com/onflow/flips/pull/54) 
will add a new `access(X)` modifier for entitled access, which will likely be used frequently 
in many contracts. It would be more consistent with this new `access(X)` syntax for `pub`
and `priv` to be `access(all)` and `access(self)` respectively. 

## User Benefit

This will increase the readability and consistency of user code. 

## Design Proposal

The proposal is simple: just remove the `pub` and `priv` aliases. `pub(set)` will 
also be removed. 

### Drawbacks

This will increase the verbosity of all code, as well as break every single contract
that exists, as it is extremely unlikely that there is any code on Mainnet currently that 
does not use at least one `pub` modifier. However, as discussed in the next section, this may be
an benefit rather than a drawback. 

### Compatibility

This is not backwards compatible, and will indeed break every existing contract. However, 
given that the aforementioned entitlements change is also going to require a rewrite of all
code currently deployed to Mainnet, a change such as this that statically breaks all of that code
will alert developers that changes are required, rather than allowing potentially newly unsafe
code to slip through undetected. 

### User Impact

This is going to require all code to be updated to not use the deprecated modifiers. It would 
be possible to write a migration assistant that would automatically replace the `pub` and `priv`
modifiers with their equivalent long forms, but it is unclear if we would want to do this, given
that a stated goal of this change is to make developers manually consider which `pub` methods are
truly `access(all)` and which should be rewritten to use entitlements. 

## Questions and Discussion Topics
