# Attachments

| Status        | (Proposed)                                           |
:-------------- |:---------------------------------------------------- |
| **FLIP #**    | [NNN](Link to FLIP)                                  |
| **Author(s)** | Daniel Sainati (daniel.sainati@dapperlabs.com)       |
| **Sponsor**   | Daniel Sainati (daniel.sainati@dapperlabs.com)       |
| **Updated**   | 2022-10-12                                           |

## Objective

This FLIP proposes to add a new `attachment` feature to Cadence, allowing users to extend existing
composite types (structs and resources) with additional fields and methods, without modifying the
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
`<access modifier> attachment <Name> for <Type> { ... }`, where the attachment methods and fields are declared in the body. As such, 
the following would be examples of legal declarations of attachments:

```cadence
pub attachment Foo for MyStruct {
    ...
}

priv attachment Bar for MyResource {
    ...
}
```

Specifying the kind (struct or resource) of an attachment is not necessary, as its kind will necessarily be the same as the type it is extending.

The access modifier defines the scope in which the `attachment` can be used: a `pub attachment` can be attached to its original type anywhere that imports it, 
while an `access(contract) attachment` can only be used within the contract that defines it. Note that this access is different than the access 
of the fields or methods within the `attachment` itself; a `pub` attachment can declare a `priv` field, for example. It is also worth noting that this access
modifier only applies to the "attaching" of the `attachment`; an `attachment` can be removed from a resource by the owner of that resource in any context. 

Within the attachment declaration, fields and methods can be defined the same way they would be in a struct or resource declaration, with an access modifier and 
a declaration kind. The fields of the base type for which the attachment are accessible to the attachment using the `super` value, which is an implicit field 
of the attachment that is a reference to the base type. So, for an attachment declared `pub attachment Foo for Bar`, the `super` field of `Foo` would have type `&Bar`.
The fields and methods defined on the attachment itself would be accessible using the `self` value as normal. Note, however, that attachments only have access to the same fields and functions on the `super` field as other code declared in the same place would have; i.e. an attachment defined in the same contract as its original type would have access to `pub` and `access(contract)` fields and methods on `super`, but not `priv` fields or methods, while an attachment defined in a different contract and account to its original type would only be able to reference `pub` fields and methods on the `super` field. 

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
        self.derivedX = super.x * scalar
    }

    pub fun foo() {
        super.foo()
    }
}

```

Any fields that are declared in an attachment must be initialized, just as any fields declared in a composite must be. An attachment
that declares fields must declare an initializer, which is run when the attachment is created. The `init` function on an attachment is run after
the attachment is attached to the base type, so `super` is accessible in the initializer.

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

Any resource fields (which are only legal in resource attachments) must also be explicitly handled in a `destroy` method, which is run when
the attachment is destroyed/removed from its base type. Like `init`, because `destroy` will be before the attachment is actually removed from the base
type, `super` will be populated and accessible in the method body. 

If a resource with attachments on it is `destroy`ed, the `destroy` methods of all its attachments are all run in an unspecified order; `destroy` should not
rely on the presence of other attachments on the base type in its implementation. The only guarantee about the order in which attachments are destroyed in this case
is that the base type will be the last thing destroyed. 

An attachment declared with `pub attachment A for C { ... }` will have a nominal type `A`.

### Adding Attachments to a Type

An attachment can be attached to a base type using the `attach e1 to e2` expression. This expression requires that `e1` have an attachment type, 
and that `e2` have the composite type to which `e1` is intended to be attached. It will fail to typecheck otherwise. 

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

### Removing Attachments from a Type

Attachments can be removed with a new statement: `remove t from e`. The `t` value here is a type name, rather than a value, 
as the attachment being removed cannot be referenced as a value. In order to typecheck, if `t` is the name of some attachment type `T2`, `e` must have some composite type `T1`
such that `T1` is the intended base type for `T2`. Before the expression executes, `T2`'s `destroy` method (if present) will be executed. After the expression executes, the composite denoted by `e` will no longer contain the attachment `T1`. If the value denoted by `e` does not contain `t`, this statement is a no-op.

Attachments may be removed from a type in any order, so users should take care not to design any attachments that rely on specific behaviors of other attachments, as there is no
way in this proposal to require that an attachment depend on another or to require that a type has a given attachment when another attachment is present. 

If a resource containing attachments is `destroy`ed, all its attachments will be `destroy`ed in an arbitrary order. 

### Accessing Attachments

Once an attachment has been added to a resource, it can be accessed using the `getAttachment` method, which is implicitly present on all resources, with the
following signature:

```cadence
fun getAttachment<T: AnyAttachment>(): &T? 
```

See the below section for a description of the new `AnyAttachment` type. `getAttachment` will query the resource for an attachment with that type, 
returning a reference to it if it is present, while returning `nil` otherwise. So, given a resource `r` with an attachment of type `A`, accessing `A`'s `foo` method
would be done like so:

```cadence
r.getAttachment<A>()!.foo()
```

### Iterating over Attachments

All resource types will contain a new function `forEachAttachment` that iterates over all the attachments present on that resource, with the following signature:

```cadence
fun forEachAttachment(_ f: ((&AnyAttachment): Void)): Void 
```

`AnyAttachment` is a new type that expresses the supertype of all attachments, 
and contains two methods (that are implicitly present on all attachments): `getField` and `getMethod`, 
with the following signatures:

```cadence
getField<T: Any>(_  name: String): &T?
getMethod<T: Any>(_  name: String): T?
```

This functions takes the `name` of a member on an attachment and checks whether a member with that `name` exists on the attachment with the provided type argument. If it does, 
`getField` will return a reference to that member is it is a field, but `nil` if it is a method, while `getMethod` will return that function if it is a method but `nil` if it is a field. If the type does not match or the member is not present, then both will return nil. These functions must be separate in order to support resource fields on attachments; 
`getField` must return a reference to prevent duplicating any resource fields, while `getMethod` cannot return a reference because references to functions cannot be called. 

So, for example, if the creator of a resource would like to have a method that returns the a descriptive string describing that resource and all its attachments, 
they may implement it this way:

```
pub resource R {
    ...
    priv let descriptionString: String
    pub fun description(): String {
        let description = descriptionString
        self.forEachAttachment(f: fun ((attachment: &AnyAttachment): Void {
            let descriptionFunction = attachment.getMethod<(():String)>("description")
            if descriptionFunction != nil {
                description.concat(descriptionFunction!())
            }
        }))
    }
}

### Drawbacks

Adding a new language feature has the downside of complexity: users have to learn yet another 
concept, and it also complicates the language implementation.

However, this language feature can be disclosed progressively, users can discover and use it when needed, 
it is not necessary to be understood for core use-cases of the language, i.e. the target audience is mostly “power-users”.

### Tutorials and Examples

* TODO: fill in later once the details of the design are decided

### Compatibility

This is backwards compatible, as it does not invalidate any existing Cadence code. 

## Related Issues

* The previous extensions proposal (see the Alternatives section below) had support for extensions/attachments
implementing interfaces. Is this a valuable feature to have, or is it not necessary? Support for it could be added
in the future, but would require separate discussion. 

* This proposal does not include a static type to express the type of a resource with attachments, despite this existing
in the previous proposal. This could be added later, along with a method to get a non-optional reference to an attachment
on resources that we can statically guarantee have that attachment. 

## Alternatives

In a previous [FLIP](https://github.com/onflow/flow/pull/1101), a solution to the extensibility
problems was proposed that suggested adding extensions to Cadence a la Swift or Scala. In this proposal, 
extending a base type would create a subtype of that original type with the fields and methods of the 
extension added to it, and had strong static typing guarantees at the expense of strict rules about
field/method name conflicts. The feedback about this proposal (summarized in comments on that FLIP as 
well as in the [notes](https://github.com/onflow/cadence/blob/master/meetings/2022-09-20-Extensions.md) 
from a meeting about it raised a number of concerns about this 
design; in particular concerns about how name conflicts would be handled if contracts are updated, 
how metadata for resources with extensions would be displayed, and the limitations posed by having 
extended types be substitutable for their base types. This FLIP is intended to be an alternative to that
FLIP that has better handling for these specific issues, and contains much of the language and details of that
proposal, with some fundamental changes that result in a different system. 

## Prior Art

This has the same prior art mentioned in https://github.com/onflow/flow/pull/1101, although the style of 
inspecting attachment methods and types is inspired by runtime reflection in langauges like Java or JavaScript.  

## Questions and Discussion Topics

