# Isolated Web Apps Explainer

## Introduction

This document proposes a way of building applications using web standard technologies that will have useful security properties unavailable to normal web pages. They are tentatively called Isolated Web Apps (IWAs). Rather than being hosted on live web servers and fetched over HTTPS, these applications are packaged into [Web Bundles](https://wicg.github.io/webpackage/draft-yasskin-wpack-bundled-exchanges.html), signed by their developer, and distributed to end-users through one or more of the potential methods described below.

## Motivating Use Cases

Content Security Policy (CSP) provides strong protection against cross-site scripting (XSS) vulnerabilities. Transport Layer Security (TLS) and Subresource Integrity (SRI) provide protection against resources being tampered with in transit or when hosted on third-party servers. However, the threat model for some particularly security sensitive applications includes the main application server itself being compromised and serving malicious content. This goes beyond the protections that current policies can provide. An environment stricter than [[SECURE-CONTEXT]](https://w3c.github.io/webappsec-secure-contexts/) is therefore required.

For example, developers of the private messaging application Signal [concluded](https://github.com/signalapp/Signal-Desktop/issues/871) that it was more secure to distribute their application as a versioned and signed package through an application store. They were concerned that self-hosting a web app would put their users at risk if their servers were compromised to serve malicious code. When Chrome Apps were deprecated, they chose to move their application to the Electron framework and accept the increase the download size and complexity of their application rather than put their users at risk. The developers of WhatsApp, another secure messaging application, have similar concerns and have produced a [browser extension](https://chrome.google.com/webstore/detail/code-verify/llohflklppcaghdpehpbklhlfebooeog) to perform out-of-band validation of the code served to users' browsers. A comparison between that work and this proposal is included at the end of this document.


## Non-goals

This proposal should _not_ be considered a desirable model for most web-based applications. Developers who choose the additional security provided by this proposal give up a number of desirable properties that come from building a web app. An example is the ability for users to discover their application by following a link that drops them into a complete experience rather than having to first install the application. They also lose the ability to quickly deploy updates from their own servers. Most applications do not need the enhanced security that this model offers. Existing methods can protect against many types of attacks without the additional burden of bundling and signing their resources.

## Proposed Solution

The first component of this proposed solution is to decouple the integrity of the site resources from the integrity of the host serving them. Web Bundles (combined with [Signed HTTP Exchanges](https://wicg.github.io/webpackage/draft-yasskin-http-origin-signed-responses.html)) propose a way of packaging HTTP responses in a way that allows an untrusted third party to distribute them on behalf of the original server. Isolated Web Apps take this in a different direction by creating an entirely new origin for content served from these bundles.

The reason for this is both practical and philosophical. If the identity of the site were still based on a DNS name, then it would still be vulnerable to a temporary loss of control over that domain or the infrastructure used to validate ownership of the domain. Philosophically, we also want to avoid building an alternative to certificate authorities which shares the same namespace. Isolated Web Apps therefore use a new scheme (tentatively, `isolated-app://`) where the authority section of the URL is a hash of the public key used to sign the Web Bundle containing the application resources.

An application can be upgraded by replacing its Web Bundle with a new version signed by the same key. Since the key hash is the same, the application retains any local storage associated with the previous version. To prevent downgrade attacks, implementations may require either a `"version"` field in the [Web Application Manifest](https://www.w3.org/TR/appmanifest/), or the signature timestamp to be monotonically increasing.

The protection against server compromise would be no good if the application could be tricked into loading malicious content from outside its Web Bundle, and so a strict Content Security Policy is applied,

```
Content-Security-Policy: base-uri 'none';
                         default-src 'self';
                         object-src 'none';
                         frame-src 'self' https:;
                         connect-src 'self' https:;
```

In this policy `'self'` refers to resources loaded from the application’s Web Bundle since its origin only addresses resources from within the bundle. `'self'` also excludes `blob:`, `filesystem:`, and other local schemes, as well as inline script, which makes it more difficult to use external resources gathered through `fetch()` to change the application's behavior. Cross-origin iframes, HTTP requests from JavaScript, and WebSocket connections are still allowed so that the application can interact with network resources.

To further protect these applications from interference from potentially malicious third-party content, they must be [cross-origin isolated](https://web.dev/why-coop-coep/) and so these headers are also applied:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Resource-Policy: same-origin
Content-Security-Policy: frame-ancestors 'self'
```

Applying these policies can be accomplished in a couple of ways. Initially we plan to inject these headers into all resources loaded from the bundle. However, since Web Bundles support capturing the HTTP response headers for individual resources, validating that the headers are already present is also an option and would allow applications to customize these protections as long as the modifications can still be considered safe, for example, to add additional `unsafe-hashes` CSP directives so that web frameworks that rely on inline script can still be used.

The policies above already restrict these applications to loading as the top-level document. However, malicious third-party content can create a confusing and potentially exploitable user experience by navigating to one of the application’s documents in an unexpected way (e.g. navigating directly to an internal settings page). Such sequence breaking attacks are prevented by disallowing cross-origin navigations to the application. The application may only be launched by navigating to its [start\_url](https://developer.mozilla.org/en-US/docs/Web/Manifest/start_url) or similar well-defined entry point such as a [protocol handler](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/URLProtocolHandler/explainer.md) or [Share Target](https://github.com/w3c/web-share-target/). [Launch handling](https://github.com/WICG/sw-launch/blob/main/launch_handler.md) may also provide a safe method to allow more dynamic control over incoming navigations.

Implementations may choose to make an IWA behave more “app-like” by only allowing them to be launched in a standalone window and assigning them a separate [storage shed](https://storage.spec.whatwg.org/#storage-shed) so that third-party storage from the user’s normal browsing session is not available. Proposed changes to the web platform in general to reduce access to third-party storage could eventually make the latter the default behavior for any origin.

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
*   Mozilla [proposed](https://wiki.mozilla.org/Apps/Security) signed ZIP archives for applications on Firefox OS and a “[new security model](https://wiki.mozilla.org/FirefoxOS/New_security_model)” for such applications. 
*   The [Browser Extension Community Group](https://www.w3.org/community/browserext/) [suggests](https://browserext.github.io/browserext/#packaging) packaging Web Extensions in ZIP archives but leaves the details up to the implementation.
*   The [MiniApps Working Group](https://www.w3.org/2021/miniapps/) also [defines](https://w3c.github.io/miniapp-packaging/) a format based on ZIP archives.
*   Electron defines a [custom](https://github.com/electron/asar) archive format for efficient loading of application resources.
*   Microsoft previously supported building UWP applications using JavaScript which were packaged using MSIX.

### Comparison to Code Verify

Meta and Cloudflare have [announced](https://blog.cloudflare.com/cloudflare-verifies-code-whatsapp-web-serves-users/) a collaboration to resolve the same threat of tampering with the code for the WhatsApp web app that this proposal addresses. Their approach is similar to [Binary Transparency](https://binary.transparency.dev/), using [a browser extension](https://github.com/facebookincubator/meta-code-verify) to compare the running code against a manifest of expected hashes that is distributed out-of-band with the application. In this case, the manifest is served by Cloudflare while the site is served by Meta. This has the advantage of not requiring a new way to distribute their application. Users who don’t have the extension installed visit the site as usual, while users with the extension have additional protection. One can imagine a future in which the validation logic is embedded into the browser, bringing this protection to more users. As with [OSCP](https://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol) however, an attacker who can compromise the user’s connection to the site may also be able to compromise their connection to the validation endpoint and the current design fails open.

This proposal addresses that vulnerability by defining a single moment at which the application is vulnerable to a network attacker, that is, the moment when a bundle is downloaded, and afterwards resources are not fetched over the network. The Code Verify system could be improved by blocking access to the site until the manifest has been fetched, so that only verified resources can be loaded rather than warning the user that the resources have been tampered with after the fact. At that point however, the benefit of individually fetching resources over the network is weakened and using a signed Web Bundle, as proposed here, allows fetching and verification to be completed as a single step.  

## Acknowledgements

*   Andrew Whalley &lt;awhalley@google.com>
*   Mike West &lt;mkwst@google.com>
*   Penny McLachlan &lt;pjmclachlan@google.com>
