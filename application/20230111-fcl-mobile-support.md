---
status: draft
flip: 241
authors: Jordan Ribbink (jordan.ribbink@flowfoundation.org)
sponsor: Jeffrey Doyle (jeffrey.doyle@dapperlabs.com)
updated: 2021-01-12
---

# FLIP 241: FCL Mobile Support

## Objective

This FLIP aims to establish compatibility between the FCL protocol and native mobile applications by exposing standardized client platform and introducing mobile-specific views.

## Motivation

Currently, mobile support for the FCL protocol is very limited. The original FCL specification was designed with a primary emphasis on supporting web-based dApps via the FCL-JS client. However, the specification did not give the same level of attention to requirements of mobile-native dApps. As a result, any existing mobile FCL clients (i.e. FCL Swift, FCL Android) have been forced to knowingly violate guidelines defined by the specification in order to create an end-to-end experience. Effectively, existing mobile clients merely emulate the behaviour of a web-based FCL client. Therefore, no standardized mechanism exists for wallets to distinguish between clients from different platforms (i.e. web/mobile) and they lack the ability to implement any level of platform-specific conditional logic.

These out-of-spec integrations create a compatibility risk between clients & wallets. The FCL `LocalView` types currently available to wallet developers (iframe, popup, and tab) are web-based primitives unsupported by mobile applications. Consequentially, wallets lose specificity when interacting with these clients - mobile clients are forced to override the wallet-defined view type in favour of one supported by their environment (i.e. `ASWebAuthenticationSession` on IOS). Unaware of this replacement, wallets may draw incorrect conclusions regarding nature of the interface displayed to the user (i.e. `postMessage` is unavailable within `SFAuthenticationSession` and other equivalents & differences in appearance).

Additionally, React Native support has emerged as a popular feature request of the FCL-JS library. Ostensibly, it is possible for FCL React Native to remedy mobile compatibility issues by deploying the same techniques as existing mobile clients. However, this approach comes with substantial risks regarding stability, future compatibility, and user-experience. Notwithstanding, FCL-JS stands as the Flow Blockchain’s flagship FCL client and should strive to adhere to official guidelines set by the specification. If the largest consumer of the FCL specification were to violate these standards, it would be paradoxical to the purpose of the FCL protocol itself - defining a standardized communication channel between wallets & dApps.

More generally, formalizing platform-specific components of the FCL specification is a necessary primitive. It should be well defined to support progression of the protocol to a wide variety of clients (such as mobile), and not just those within web environments.

## User Benefit

The end-user benefit of this proposal is that it that it empowers developers to build seamless, mainstream, mobile-first experiences. These changes to the FCL specification would help solidify FCL’s role in mobile dApp development, providing a formal framework for FCL’s mobile support akin to that which already exists for web-based dApps. In order for wallets to fine-tune their user experience and leverage any mobile-native features (i.e. deep linking), a protocol-level distinction between mobile & web clients is required.

## Design Proposal

This proposal has two objectives related to mobile client integration with the FCL protocol:

1. Expose standardized client platform information during FCL connection handshaking - i.e. within FCL authentication (authn) requests. For this version of the FCL specification, two platform variants will exist: web & mobile.
2. Formalize existing local view & service compatibility with different clients (i.e. web & mobile), as well as define new mobile-native local view types.

### Authn Platform Information

FCL clients will send an additional piece of metadata, `fclClientInfo` with their authn requests moving forward.

```tsx
{
	...rest,
	fclClientInfo?: {
		platform: string // The platform the FCL client library is running on. Currently, web and mobile are the only supported values
		name: string // The name of the FCL client library. For example, fcl-js or fcl-swift. This is for diagnostic purposes only, it should be preferred to use the Fcl-Platform header to behave differently based on the platform.
		version: string // The version of the FCL client library. For example, 0.0.1 or 1.0.0. It is not recommended to use these version numbers to determine compatibility, as they are not guaranteed to follow any particular scheme. Instead, use the f_vsn field of the FCL objects to determine compatibility.
	}
}
```

Older versions of FCL clients may not send these headers, so it is recommended to not rely on them being present.

### Local Views

Within the current FCL specification, backchannel services have the ability to render a local "view" within `HTTP/POST` polling responses. This offers the wallet provider the ability to display interactive UI components to the user.

Currently, only `VIEW/IFRAME`, `VIEW/POP`, and `VIEW/TAB` exist. Unfortunately, rendering any of these view types is premised on the availability of certain Javascript APIs which would only be available on a web-based platform. This is problematic for adoption of any FCL clients wishing to build on a platform that does not have these primitives available (i.e. native mobile applications).

This proposal would introduce two new, mobile-exclusive views, `VIEW/MOBILE_BROWSER` & `VIEW/DEEPLINK`.

**VIEW/MOBILE_BROWSER**

`VIEW/MOBILE_BROWSER` is the mobile counterpart to `VIEW/IFRAME`, `VIEW/POP`, and `VIEW/TAB` in the web. It will display a secure browser window on the user's mobile device (i.e. Android Custom Tabs or iOS SFAuthenticationSession).

However, its implementation is nuanced by limitations of mobile platforms. Views for web clients are managed by the parent FCL client, but on mobile platforms, the FCL client is unable to control the mobile browser window. This means that wallet providers bear the responsibility of internally dismissing the mobile browser when the user has completed their interaction using Javascript APIs.

Clients should execute all `ViEW/MOBILE_BROWSER` views with a `fcl_redirect_uri` query parameter, which the wallet should use to return to the dApp. If this is not provided, wallets will be unable to dismiss a view and will rely on user interaction in order to return to the dApp.

**VIEW/DEEPLINK**

`VIEW/DEEPLINK` is a view responsible for redirecting to another application on the user's device via a universal link. The wallet's universal link should be provided as the `endpoint` parameter for the view. Only a private-use URI scheme is supported (i.e. universal links on IOS) for security reasons (see [https://datatracker.ietf.org/doc/html/rfc8252](https://datatracker.ietf.org/doc/html/rfc8252)).

Like `VIEW/MOBILE_BROWSER`, this view (external dApp) is responsible for dismissing itself when the user has completed their interaction with the application. A redirect URL should be passed as a query parameter by the client, `fcl_redirect_uri`, which the wallet should use to return to the dApp. If this is not provided, wallets will be unable to dismiss a view and will rely on user interaction in order to return to the dApp.

**Compatibility Matrix**

The following table outlines the proposed compatibility matrix of local views with client platforms:

| Service Method      | Web | Mobile |
| ------------------- | --- | ------ |
| VIEW/IFRAME         | ✅  | ⛔     |
| VIEW/POP            | ✅  | ⛔     |
| VIEW/TAB            | ✅  | ⛔     |
| VIEW/MOBILE_BROWSER | ⛔  | ✅     |
| VIEW/DEEPLINK       | ⛔  | ✅     |

Naturally, wallets would be required to be cognizant of the client’s platform when attempting to display a local view. The types of the views displayed by a wallet should be types which are supported by the client platform.

### Drawbacks

The obvious drawback to this proposal is added wallet complexity to wallets - they now must be aware of client platform types and act appropriately. However, this is a necessary distinction, as wallets must have the ability to consider the idiosyncrasies of each platform/client in order to act appropriately. Without such differentiation, the wallet would face ambiguity regarding the client environment, making it difficult to create a smooth and cohesive user experience.

### Alternatives Considered

The existing alternative available within the Flow Ecosystem is for any non-web-based FCL clients to emulate a web-based FCL client, violating aspects of the FCL specification. As aforementioned in Motivation, this is a potentially dangerous direction to pursue. Without the ability to differentiate between client platforms and act accordingly, the FCL protocol risks wallet instability, future incompatibilities, and user experience degradation. It is only natural that, in order to become a truly multi-platform protocol, FCL adopts a paradigm like this one proposed, or similar, moving forward.

### Performance Implications

N/A

### **Dependencies**

This affects wallet developers building on Flow as well as any FCL clients (i.e. FCL-JS, FCL Swift, FCL Android). This also means the Flow Dev Wallet will need to be updated.

### Engineering Impact

Will not significantly affect binary size, build, or test times. The engineering impact of this is that it adds new functionality to the FCL specification, meaning all FCL-compliant clients & wallets must adopt these changes.

### Best Practices

This definitionally changes best practices for wallet & SDK developers on Flow as it is a change to the FCL specification. The changes will be communicated through updates the the FCL specification document.

### Tutorials & Examples

Tutorials shouldn’t be necessary as this is a specification change only used by a small subset of developers. Examples are not necessary beyond generalized examples using the mobile SDKs in question (i.e. this proposal is all abstracted away from the dApp developer through their client library’s API).

While it is not a production-grade example, the Flow Dev Wallet can serve as a protocol reference to wallet developers wishing to implement these changes. For developers to test their mobile dApps locally, the Dev Wallet will need to implement these FCL spec changes as well.

### Compatibility

These changes will be backward-compatible as they will follow the recently proposed FCL Protocol Versioning Specification.

Two components of the FCL protocol will need version changes.

1. `Authentication` service will need a version bump to include client platform information.
2. `LocalView` service will need a version bump to include the newly added local views for mobile development.

### User Impact

Negligible. The dApp developer will have to update their FCL client to use the latest version of the protocol as they would with any other feature.

## Related Issues

Mobile support within the FCL protocol is limited to only backchannel services within this proposal. Future work may look to extend this support to front-channel services as well. However, this requires in depth performance, security, and compatibility considerations.

## Prior Art

There are existing implementations of non-web-based FCL clients, however, these do not have robust protocol support as is proposed in this document. See:

- [https://github.com/outblock/fcl-swift](https://github.com/outblock/fcl-swift)
- [https://github.com/Outblock/fcl-android](https://github.com/Outblock/fcl-android)

See the original [V1.0 FCL specification (FLIP 45)](https://github.com/onflow/flips/blob/main/application/20221108-fcl-specification.md) for more information on the existing FCL protocol.

This proposal relies on [FLIP 240 (FCL Versioning Standards)](https://github.com/onflow/flips/pull/240), which is currently in draft.

## Questions and Discussion

N/A
