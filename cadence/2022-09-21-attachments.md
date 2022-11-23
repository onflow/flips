# Attachments

| Status        | (Approved)                                           |
:-------------- |:---------------------------------------------------- |
| **Author(s)** | Daniel Sainati (daniel.sainati@dapperlabs.com)       |
| **Sponsor**   | Daniel Sainati (daniel.sainati@dapperlabs.com)       |
| **Updated**   | 2022-11-23                                           |

## Objective

This FLIP proposes to add a new `attachment` feature to Cadence, allowing users to extend existing
composite types (structs and resources) with additional fields and functions, without modifying the
original declaration. This feature is purely additive, i.e. no existing functionality is changed or removed.

## Motivation

It is currently not possible to extend existing types unless the original author explicitly made provisions for future functionality.

For example, to make a resource declaration extensible, its author may add a field that allows any other code to store an attachment. However, this requires a lot of boilerplate and is brittle. The original type must be prepared to store additional data with potentially additional functionality.

Instead, this would allow users to extend existing types whether or not the original author planned for that use case. 

## User Benefit

This enables a number of uses cases that were previously difficult or impossible, including [adding autographs to TopShot moments](https://github.com/onflow/cadence/issues/357#issuecomment-683387179)(or generally adding signature/edition info to existing NFTs), or [adding apparel to CryptoKitties](https://kittyhats.co/#/). 

## Design Proposal

### Attachment Declarations

The new attachment feature would be used with a new `attachment` keyword, which would be declared using a new form of composite declaration:
`pub? attachment <Name> for <Type>: <Conformances> { ... }`, where the attachment functions and fields are declared in the body. As such, 
the following would be examples of legal declarations of attachments:

```cadence
pub attachment Foo for MyStruct {
    ...
}

attachment Bar for MyResource: MyResourceInterface {
    ...
}

attachment Baz for MyInterface: MyOtherInterface {
    ...
}
```

Specifying the kind (struct or resource) of an attachment is not necessary, as its kind will necessarily be the same as the type it is extending. Note that
the base type may be either a concrete composite type or an interface. In the former case, the attachment is only usable on values specifically of that
base type, while in the case of an interface the attachment is usable on any type that conforms to that interface. 

As with other type declarations, attachments may only have a `pub` access modifier (if one is present). A future proposal may define behavior for private type declarations,
but until such a proposal exists and is accepted, the access modifier on an attachment declaration must be `pub`. 

Within the attachment declaration, fields and functions can be defined the same way they would be in a struct or resource declaration, with an access modifier and 
a declaration kind. The fields of the base type for which the attachment are accessible to the attachment using the `base` value, which is an implicit field 
of the attachment that is a reference to the base type. So, for an attachment declared `pub attachment Foo for Bar`, the `base` field of `Foo` would have type `&Bar`.
The fields and functions defined on the attachment itself would be accessible using the `self` value as normal. Note, however, that attachments only have access to the same fields and functions on the `base` field as other code declared in the same place would have; i.e. an attachment defined in the same contract as its original type would have access to `pub` and `access(contract)` fields and functions on `base`, but not `priv` fields or functions, while an attachment defined in a different contract and account to its original type would only be able to reference `pub` fields and functions on the `base` field. 

So, for example, this would be a valid declaration of an attachment:

```
pub resource R {
    pub let x: Int

    init (_ x: Int) {
        self.x = x
    }

    pub fun foo() { ... }
}

pub attachment A for R {
    pub let derivedX: Int 

    init (_ scalar: Int) {
        self.derivedX = base.x * scalar
    }

    pub fun foo() {
        base.foo()
    }
}

```

Any fields that are declared in an attachment must be initialized, just as any fields declared in a composite must be. An attachment
that declares fields must declare an initializer, which is run when the attachment is created. The `init` function on an attachment is run after
the attachment is attached to the base type, so `base` is accessible in the initializer.

The same checks on normal initializers apply to attachment initializers; namely that all the fields declared in the attachment must be initialized with a value in the
attachment's initializer. So, the following would be a legal attachment:

```cadence
pub resource R {}

pub attachment A for R {
    pub let x: String
    init(_ x: String) {
        self.x = x
    }
}
```

while this would not:

```cadence
pub resource R {}

pub attachment E for R {
    pub let x: String
    pub let y: String
    init(_ x: String) {
        self.x = x
    }
}
```

Any resource fields (which are only legal in resource attachments) must also be explicitly handled in a `destroy` function, which is run when
the attachment is destroyed/removed from its base type. Like `init`, because `destroy` will be before the attachment is actually removed from the base
type, `base` will be populated and accessible in the function body. 

If a resource with attachments on it is `destroy`ed, the `destroy` functions of all its attachments are all run in an unspecified order; `destroy` should not
rely on the presence of other attachments on the base type in its implementation. The only guarantee about the order in which attachments are destroyed in this case
is that the base type will be the last thing destroyed. 

An attachment declared with `pub attachment A for C { ... }` will have a nominal type `A`.

If an attachment `A` is declared to conform to an interface `I`, `A` will be a subtype of `{I}`, and the checker will enforce that `A` implements all the functions 
and fields of `I`. Note that an attachment must implement all the functions and fields of `I` itself; it may reference its base type with `base` but it cannot inherit 
from that type. 

Important note: because attachments are not first class values in Cadence, their types cannot appear outside of a reference type. So, for example, given an 
attachment declaration `attachment A for X {}`, the types `A`, `A?`, `[A]` and `((): A)` are not valid type annotations, while `&A`, `&A?`, `[&A]` and `((): &A)` are valid. 
For this reason as well, `self` inside an attachment has a reference type. 
In the attachment declaration `A` above, the type of `self` would be `&A`, rather than `A` like in other composite declarations.

### Adding Attachments to a Type

An attachment can be attached to a base type using the `attach e1 to e2` expression. This expression requires that `e1` have an attachment type, 
and that `e2` be a subtype of `e1`'s intended base type. It will fail to typecheck otherwise. 

```cadence
resource R {}
attachment A for R {}
```

The following would be valid ways to attach `A` to `@R`:

```cadence 
let r <- create R()
let r2 <- attach A() to <-r 
...
```

or 

```cadence 
let r2 <- attach A() to <-create R()
```

An attachment can only be created in the same statement in which it is attached; so the `A()` expression is only legal inside an `attach` expression. 
In general, attachments are not first class values in Cadence, they can only be added, removed and accessed from composite values, and are not true values themselves. 
For this reason, resource attachments do not need an expliict `<-` move operator when they appear in an `attach` expression. 
If the attachment has an initializer, the arguments to that initializer are provided in the creation of the attachment like so:

```cadence
struct S {
    pub let x: String
    init(x: String) {
        self.x = x
    }
}

attachment A for S {
    pub let y: Int 
    init(y:Int) {
        self.y = y
    }
}

let sa = attach A(y: 3) to S(x: "foo")
```

If a users attempts to `attach` an attachment to a value that already possesses an attachment of that type, Cadence will raise an error at runtime. For this
reason, if a user has reason to believe that an attachment of the type they wish to add already exists on a value, they should use the `remove` statement 
first to guarantee that the value does not have that attachment before they attempt to attach it again.

The `attach` expression creates a new value, rather than modifying the old one; thus if the base is a struct, the result of the expression will have the attachment,
while the original value will not. In the case of a resource, the old resource will be moved into the new one, which will have the attachment present. It is also worth
noting that while `base` does point to the base during the attachment's constructor, that base does not yet have the attachment present on it during the execution
of that constructor; that only occurs after the attachment expresion successfully executes. To see why, consider what would happen if a user accessed that attachment 
through `base`; if in `A`'s initializer they wrote `base[A]!.foo?`. This could potentially allow `A`'s `foo` field to be accessed before it has been initialized, and
would require analysis of statement ordering within the initializer itself to determine when it was safe to access which fields of `A` through `base`. Rather than add
this additional complexity, we simply do not attach `A` to the base until after the initializer has finished executing. 

### Removing Attachments from a Type

Attachments can be removed with a new statement: `remove t from e`. Here, `t` refers to an attachment type name, rather than a value, 
as the attachment being removed cannot be referenced as a value. In order to typecheck, if `t` is the name of some attachment type `T2`, `e` must have some composite type `T1`
such that `T1` is the intended base type for `T2`.

Before the statement executes, `T2`'s `destroy` function (if present) will be executed. After the statement executes, the composite denoted by `e` will no longer contain the attachment `T1`. If the value denoted by `e` does not contain `t`, this statement is a no-op.

Attachments may be removed from a type in any order, so users should take care not to design any attachments that rely on specific behaviors of other attachments, as there is no
way in this proposal to require that an attachment depend on another or to require that a type has a given attachment when another attachment is present. 

If a resource containing attachments is `destroy`ed, all its attachments will be `destroy`ed in an arbitrary order. 

### Accessing Attachments

Once an attachment has been added to a composite value, it can be accessed using indexing syntax: `v[T]`, where `v` is a resource or struct composite value, or a reference to a struct or resource composite value, and `T` is the name of an attachment type. This indexing syntax returns an optional non-auth reference to the attachment of type `T`: `&T?`; if no such attachment exists on `v`, the expression will return `nil`. So, given a composite `r` with an attachment of type `A`, accessing `A`'s `foo` function would be done like so:

```cadence
r[A]!.foo()
```

This syntax is only valid if `A` is a valid attachment type for `v`; i.e. if `A`'s declared base type is a supertype of the type of `v`. What this means is that the owner
of a resource can restrict which attachments can be accessed on references to their resource using restricted types, much like they would do with any other field or function. E.g.

```
struct R: I {}
struct interface I {}
attachment A for R {}
fun foo(r: &{I}) {
    r[A] // fails to type check, because `{I}` is not a subtype of `R`
}
```

### Drawbacks

Adding a new language feature has the downside of complexity: users have to learn yet another 
concept, and it also complicates the language implementation.

However, this language feature can be disclosed progressively, users can discover and use it when needed, 
it is not necessary to be understood for core use-cases of the language, i.e. the target audience is mostly “power-users”.

### Compatibility

This is backwards compatible, as it does not invalidate any existing Cadence code. 

## Related Issues

There is a plan to add support for iterating over and reflecting on attachments in a future FLIP. 
A summary of the future proposed changes (which used to be part of this FLIP) are included below:

### Iterating over Attachments

All composite types will contain a new function `forEachAttachment` that iterates over all the attachments present on that composite. On a resource, this function 
will have the following signature:

```cadence
fun forEachAttachment(_ f: ((&AnyResourceAttachment): Void)): Void 
```

On a structure, it will have 

```cadence
fun forEachAttachment(_ f: ((&AnyStructAttachment): Void)): Void 
```

`AnyResourceAttachment`/`AnyStructAttachment` are new types that expresses the basetypes of all struct/resource attachments.
They contains two functions (that are implicitly present on all attachments): `getField` and `invokeFunction`, 
with the following signatures:

```cadence
getField<T>(_  name: String): &T?
invokeFunction<(Args...): Ret>(_  name: String, args: Args...): Ret?
```

The difference between these two types is only in the kind; as might be expected from the name `AnyResourceAttachment` is resource-kinded, while
`AnyStructAttachment` is struct-kinded. They must be separate types as attachments themselves are resource or struct kinded depending on their base type. This distinction
is important because only resource-kinded attachments may contain resource-kinded fields.

Here, `invokeFunction` should take some function type as its type parameter, which will be used to typecheck the variadic arguments passed to the function call
after the `name`. If a function member exists on the attachment with that `name` and type, it will be called with the provided arguments, and the return value
of that function will be returne from `invokeFunction`. Otherwise, `invokeFunction` returns `nil`. 

`getField` takes the `name` and a type of a field on an attachment, returning a reference to that field if it is present with that type, but `nil` if the member
does not exist or is a function. 

Note that in order to prevent `getField` and `invokeFunction` from being used to circumvent access control, these functions can only be used to access 
attachment members that were declared to have `pub` access. 

For similar reasons, `forEachAttachment` will only iterate over those attachments on the receiver that could be accessed on that receiver using the normal 
access syntax; that is, if a receiver has a static type that is larger than its runtime time, any attachments present on it declared for a type smaller than 
its static type will be skipped during the call. For example, if an attachment `A` is created for `Vault` and attached to a `Vault` object, if a `&{Receiver}` 
reference is exposed to that `Vault`, when iterating over that restricted reference's attachments, `A` will be skipped, as it should not be possible to access 
`Vault` attachments with only a `Receiver` value.  

So, for example, if the creator of a resource would like to have a function that returns the a descriptive string describing that resource and all its attachments, 
they may implement it this way:

```
pub resource R {
    ...
    priv let descriptionString: String
    pub fun description(): String {
        let description = descriptionString
        self.forEachAttachment(f: fun (attachment: &AnyResourceAttachment) {
            let attachmentDescription = attachment.invokeFunction<(():String)>("description")
            if attachmentDescription != nil {
                description.concat(attachmentDescription!)
            }
        }))
    }
}
```

Attempting to attach new attachments to `R` or remove attachments from `R` while iterating over it is a runtime error. 
## Alternatives

In a previous [FLIP](https://github.com/onflow/flow/pull/1101), a solution to the extensibility
problems was proposed that suggested adding extensions to Cadence a la Swift or Scala. In this proposal, 
extending a base type would create a subtype of that original type with the fields and functions of the 
extension added to it, and had strong static typing guarantees at the expense of strict rules about
field/function name conflicts. The feedback about this proposal (summarized in comments on that FLIP as 
well as in the [notes](https://github.com/onflow/cadence/blob/master/meetings/2022-09-20-Extensions.md) 
from a meeting about it raised a number of concerns about this 
design; in particular concerns about how name conflicts would be handled if contracts are updated, 
how metadata for resources with extensions would be displayed, and the limitations posed by having 
extended types be substitutable for their base types. This FLIP is intended to be an alternative to that
FLIP that has better handling for these specific issues, and contains much of the language and details of that
proposal, with some fundamental changes that result in a different system. 

## Prior Art

This has the same prior art mentioned in https://github.com/onflow/flow/pull/1101, although the style of 
inspecting attachment functions and types is inspired by runtime reflection in langauges like Java or JavaScript.  

## Questions and Discussion Topics

