# Isolated Web Apps Explainer

## Introduction

This document proposes a way of building applications using web standard technologies that will have useful security properties unavailable to normal web pages. They are tentatively called Isolated Web Apps (IWAs). Rather than being hosted on live web servers and fetched over HTTPS, these applications are packaged into [Web Bundles](https://wpack-wg.github.io/bundled-responses/draft-ietf-wpack-bundled-responses.html), signed by their developer, and distributed to end-users through one or more of the potential methods described below.

## Motivating Use Cases

Content Security Policy (CSP) provides strong protection against cross-site scripting (XSS) vulnerabilities. Transport Layer Security (TLS) and Subresource Integrity (SRI) provide protection against resources being tampered with in transit or when hosted on third-party servers. However, the threat model for some particularly security sensitive applications includes the main application server itself being compromised and serving malicious content. This goes beyond the protections that current policies can provide. An environment stricter than [[SECURE-CONTEXT]](https://w3c.github.io/webappsec-secure-contexts/) is therefore required.

For example, developers of the private messaging application Signal [concluded](https://github.com/signalapp/Signal-Desktop/issues/871) that it was more secure to distribute their application as a versioned and signed package through an application store. They were concerned that self-hosting a web app would put their users at risk if their servers were compromised to serve malicious code. When Chrome Apps were deprecated, they chose to move their application to the Electron framework and accept the increase the download size and complexity of their application rather than put their users at risk. The developers of WhatsApp, another secure messaging application, have similar concerns and have produced a [browser extension](https://chrome.google.com/webstore/detail/code-verify/llohflklppcaghdpehpbklhlfebooeog) to perform out-of-band validation of the code served to users' browsers. A comparison between that work and this proposal is included at the end of this document.

A user agent may also force an application to adopt this threat model if the developer needs access to APIs which would make the application an appealing target for XSS or server-side attacks. For example, the [Direct Sockets API](https://wicg.github.io/direct-sockets/) allows a site to make arbitrary network connections. The Fetch Standard defines the [CORS protocol](https://fetch.spec.whatwg.org/#cors-protocol) to carefully control the network requests that sites can make as users browse the web and execute potentially untrustworthy Javascript. Nevertheless, some applications have legitimate needs to make connections that would violate CORS and use protocols other than HTTP. The tradeoff the application developer can make in order to get access to this capability is to accept that user agents need to establish a higher standard for the trustworthiness of the application code and better protection against its compromise by malware.

## Non-goals

This proposal should _not_ be considered a desirable model for most web-based applications. Developers who choose the additional security provided by this proposal give up a number of desirable properties that come from building a web app. An example is the ability for users to discover their application by following a link that drops them into a complete experience rather than having to first install the application. They also lose the ability to quickly deploy updates from their own servers. Most applications do not need the enhanced security that this model offers. Existing methods can protect against many types of attacks without the additional burden of bundling and signing their resources.

## Proposed Solution

The core of this proposal is making application updates explicit. Unlike TLS keys, which have to be available online to establish new connections, the key used to sign the Web Bundle can be kept securely offline and is used infrequently. The channel through which updates are distributed creates another point where the new resources can be checked for potentially malicious content. We propose a new [Integrity Block](https://github.com/WICG/webpackage/blob/main/explainers/integrity-signature.md) format for signing an entire Web Bundle. This is different from bundling [Signed HTTP Exchanges](https://wicg.github.io/webpackage/draft-yasskin-http-origin-signed-responses.html) because we don’t intend to create a verifiable mirror of a subset of a site’s resources, but a holistically verifiable version of an entire application. For this reason Isolated Web Apps should use a [new scheme](./Scheme.md) for content served from these bundles.

The reason for this is both practical and philosophical. If the identity of the site were still based on a DNS name, then it would still be vulnerable to a temporary loss of control over that domain or the infrastructure used to validate ownership of the domain. Philosophically, we also want to avoid building an alternative to certificate authorities which shares the same namespace. Isolated Web Apps therefore use a new scheme (tentatively, `isolated-app://`) where the authority section of the URL is based on the public key used to sign the Web Bundle containing the application resources. More details available in the [Scheme Explainer](./Scheme.md).

An application can be upgraded by replacing its Web Bundle with a new version signed by the same key. Since the key hash is the same, the application retains any local storage associated with the previous version. To prevent downgrade attacks, implementations may require either a `"version"` field in the [Web Application Manifest](https://www.w3.org/TR/appmanifest/), or the signature timestamp to be monotonically increasing.

The protection against server compromise would be no good if the application could be tricked into loading malicious content from outside its Web Bundle, and so a rigorous Content Security Policy is applied,

```
Content-Security-Policy: base-uri 'none';
                         default-src 'self';
                         object-src 'none';
                         frame-src 'self' https: blob: data:;
                         connect-src 'self' https: wss: blob: data:;
                         script-src 'self' 'wasm-unsafe-eval';
                         img-src 'self' https: blob: data:;
                         media-src 'self' https: blob: data:;
                         font-src 'self' blob: data:;
                         style-src 'self' 'unsafe-inline';
                         require-trusted-types-for 'script';
```

In this policy `'self'` refers to resources loaded from the application’s Web Bundle since its origin only addresses resources from within the bundle. `'self'` also excludes `blob:`, `filesystem:`, and other local schemes, as well as inline script, which makes it more difficult to use external resources gathered through `fetch()` to change the application's behavior. Cross-origin images, media, iframes, HTTP requests from JavaScript, and WebSocket connections are still allowed so that the application can interact with network resources.

To further protect these applications from interference from potentially malicious third-party content, they must be [cross-origin isolated](https://web.dev/why-coop-coep/) and so these headers are also applied:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Resource-Policy: same-origin
Content-Security-Policy: frame-ancestors 'self'
```

Applying these policies can be accomplished in a couple of ways. Initially we plan to inject these headers into all resources loaded from the bundle. However, since Web Bundles support capturing the HTTP response headers for individual resources, validating that the headers are already present is also an option and would allow applications to customize these protections as long as the modifications can still be considered safe, for example, to add additional `unsafe-hashes` CSP directives so that web frameworks that rely on inline script can still be used.

The policies above already restrict these applications to loading as the top-level document. However, malicious third-party content can create a confusing and potentially exploitable user experience by navigating to one of the application’s documents in an unexpected way (e.g. navigating directly to an internal settings page). Such sequence breaking attacks are prevented by disallowing cross-origin navigations to the application. The application may only be launched by navigating to its [start\_url](https://developer.mozilla.org/en-US/docs/Web/Manifest/start_url) or similar well-defined entry point such as a [protocol handler](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/URLProtocolHandler/explainer.md) or [Share Target](https://github.com/w3c/web-share-target/). [Launch handling](https://github.com/WICG/sw-launch/blob/main/launch_handler.md) may also provide a safe method to allow more dynamic control over incoming navigations.

Implementations may choose to make an isolate app behave more “app-like” by only allowing them to be launched in a standalone window and assigning them a separate [storage shed](https://storage.spec.whatwg.org/#storage-shed) so that third-party storage from the user’s normal browsing session is not available. Proposed changes to the web platform in general to reduce access to third-party storage could eventually make the latter the default behavior for any origin.

Once bundled, an application could be distributed to users in a number of ways:

*   A raw signed Web Bundle.
*   Packaged into a platform-specific installation format such as an APK, MSI or DMG.
*   Distributed through an operating system, browser or third-party “app store”.
*   Automatically installed by enterprise system configuration management infrastructure.

Implementations may assign different levels of trust depending on the source of the application. For example, an application installed by the system administrator may be implicitly trusted, while one downloaded by the user in the form of an MSI or macOS app bundle may be required to carry the same code signature that a native application for Windows or macOS would. The level of trust required to launch an application should be configurable by the user, with safe defaults.

## Related work

Designing more trustworthy web contexts and packaging applications based on web technologies has been proposed and implemented many times before,

*   Mike West proposed “[Securer Contexts](https://github.com/mikewest/securer-contexts/blob/master/README.md)” upon which the isolation and injection protection presented here are based.
*   Google uses signed ZIP archives for both Chrome Apps and Extensions.
*   Mozilla's Firefox OS [used](https://wiki.mozilla.org/Apps/Security) a similar packaged app solution with signed ZIP archives and an `app://` protocol scheme. Further planned work on a “[new security model](https://wiki.mozilla.org/FirefoxOS/New_security_model)” was designed to support streaming these packages from HTTPS URLs to improve overall webiness and linkability.
*   The [Browser Extension Community Group](https://www.w3.org/community/browserext/) [suggests](https://browserext.github.io/browserext/#packaging) packaging Web Extensions in ZIP archives but leaves the details up to the implementation.
*   The [MiniApps Working Group](https://www.w3.org/2021/miniapps/) also [defines](https://w3c.github.io/miniapp-packaging/) a format based on ZIP archives.
*   Electron defines a [custom](https://github.com/electron/asar) archive format for efficient loading of application resources.
*   Microsoft previously supported building UWP applications using JavaScript which were packaged using MSIX.
*   [LG webOS platform](https://www.webosose.org/docs/tutorials/web-apps/developing-external-web-apps/) has supported their own web app packaging format (ipk) with [appinfo.json](https://www.webosose.org/docs/guides/development/configuration-files/appinfo-json/).
*   [Delta Chat](https://delta.chat/) and [Cheogram](https://cheogram.com) implement the [webxdc](https://webxdc.org/) archive format for embedding web apps in the context of an encrypted messenger, with all network activity blocked by default. The only way to send and receive data from the web app is by delegating message relay to the (e2e encrypting) messenger.
*   [Peergos](https://peergos.org) allows users to install 3rd party web apps from a signed snapshot. Those web apps are run in an [isolated unique-origin iframe](https://peergos.org/posts/a-better-web) locked down so no external communication is possible using COOP,COEP,CORP,CSP. Apps then communicate with the outer Peergos context via post messages where permissions are enforced, like sending E2EE messages to friends, or reading/persisting data. 

### Comparison to Code Verify

Meta and Cloudflare have [announced](https://blog.cloudflare.com/cloudflare-verifies-code-whatsapp-web-serves-users/) a collaboration to resolve the same threat of tampering with the code for the WhatsApp web app that this proposal addresses. Their approach is similar to [Binary Transparency](https://binary.transparency.dev/), using [a browser extension](https://github.com/facebookincubator/meta-code-verify) to compare the running code against a manifest of expected hashes that is distributed out-of-band with the application. In this case, the manifest is served by Cloudflare while the site is served by Meta. This has the advantage of not requiring a new way to distribute their application. Users who don’t have the extension installed visit the site as usual, while users with the extension have additional protection. One can imagine a future in which the validation logic is embedded into the browser, bringing this protection to more users. As with [OSCP](https://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol) however, an attacker who can compromise the user’s connection to the site may also be able to compromise their connection to the validation endpoint and the current design fails open.

This proposal addresses that vulnerability by defining a single moment at which the application is vulnerable to a network attacker, that is, the moment when a bundle is downloaded, and afterwards resources are not fetched over the network. The Code Verify system could be improved by blocking access to the site until the manifest has been fetched, so that only verified resources can be loaded rather than warning the user that the resources have been tampered with after the fact. At that point however, the benefit of individually fetching resources over the network is weakened and using a signed Web Bundle, as proposed here, allows fetching and verification to be completed as a single step.

## Discuss & Help

If you'd like to discuss the Isolated Web Apps proposal itself, please use [GitHub Issues](https://github.com/WICG/isolated-web-apps).

For discussions related to Isolated Web Apps in general, or Chromium-specific implementation and development questions, please use the [iwa-dev@chromium.org](https://groups.google.com/a/chromium.org/g/iwa-dev) mailing list.

## Acknowledgements

*   Alex Russell &lt;alexrussell@microsoft.com>
*   Andrew Whalley &lt;awhalley@google.com>
*   Mike West &lt;mkwst@google.com>
*   Penny McLachlan &lt;pjmclachlan@google.com>
