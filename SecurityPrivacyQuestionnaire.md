## [2.1 What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?](https://www.w3.org/TR/security-privacy-questionnaire/#purpose)

The Isolated Web Apps infrastructure on its own exposes less information to site authors because resources are primarily loaded from a local file rather than over the network. While it is not currently in-scope, this proposal provides a big piece of what would be necessary for developers to opt out of network access entirely and build truly "offline-only" applications. [Issue 1](https://github.com/WICG/isolated-web-apps/issues/1) covers the kinds of applications which would benefit from being able to make such a strong privacy guarantee.

Application updates however still require periodically fetching the [update manifest](Updates.md#web-application-update-manifest) from a developer-controlled server which could allow a developer to track their users by IP address. This can be mitigated by limiting the frequency of update checks or by proxying them through a trusted intermediary to provide IP privacy as discussed in the [Privacy Considerations](Updates.md#privacy-considerations) section of the updates explainer.

As mentioned in the explainer, user agents may restrict certain APIs or features to IWAs only at the additional integrity guarantees and stronger trustworthiness signals provide a mitigation for their particular security or privacy considerations. This is mentioned here for completeness even though each of those proposals should include their own more thorough justification for why this restriction is an effective mitigation in their own answers to this questionnaire.

## [2.2 Do features in your specification expose the minimum amount of information necessary to enable their intended uses?](https://www.w3.org/TR/security-privacy-questionnaire/#minimum-data)

Yes. The [permissions explainer](Permissions.md) describes that the packaged nature of Isolated Web Apps makes it practical for the developer to declare a [Permissions Policy](https://www.w3.org/TR/permissions-policy/) which applies to the entire application. This allows user agents to inform the user of the permissions the application may request before it is installed. Because updates are explicit, changes to the permission policy can also be inspected when they occur.

## [2.3 How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?](https://www.w3.org/TR/security-privacy-questionnaire/#personal-data)

Beyond IP addresses, discussed above, this feature does not deal with personal information.

## [2.4 How do the features in your specification deal with sensitive information?](https://www.w3.org/TR/security-privacy-questionnaire/#sensitive-data)

The goal of this proposal is to avoid accidentally exposing sensitive information or capabilities to the web. The required content security, and cross-origin opener, embedder and resource policy headers enforce a higher security baseline than the usual "secure context" definition which should help protect users against XSS attacks and potential leaks of sensitive information. The application-wide Permissions Policy similarly enforces a more secure baseline for what information the application could request.

Since each Isolated Web App has its own origin on a separate scheme from normal HTTP sites its data is partitioned by the normal storage partitioning mechanism.

## [2.5 Do the features in your specification introduce new state for an origin that persists across browsing sessions?](https://www.w3.org/TR/security-privacy-questionnaire/#persistent-origin-specific-state)

The installed version of the app is a new piece of state. If unique versions of the application's Web Bundle were generated for each user this could be used to track a particular installation over time. This is comparable to concerns over user tracking through the Web Application Manifest start\_url field. Similar to other forms of URL-based tracking this is difficult to mitigate by technical means as it is impossible to automatically detect what part of the URL is the tracking information.

If a User Agent requires applications to carry a signature indicating that the current version has been reviewed by a third-party then the process for obtaining that signature could be rate limited to prevent abuse.

## [2.6 Do the features in your specification expose information about the underlying platform to origins?](https://www.w3.org/TR/security-privacy-questionnaire/#underlying-platform-data)

Not directly, but as mentioned in the answer to 2.1 above user agents may require developers to build an Isolated Web App to access API features which require a greater level of site security and trustworthiness. The [Direct Sockets API](https://wicg.github.io/direct-sockets/), a proposed IWA-only feature, may expose information about the local network by providing the ability to make network connections not normally allowed over HTTP. 

## [2.7 Does this specification allow an origin to send data to the underlying platform?](https://www.w3.org/TR/security-privacy-questionnaire/#send-to-platform)

See the answers to 2.6, 2.10 and 2.16 for mentions of proposed IWA-only APIs which allow access to low-level platform capabilities. 

## [2.8 Do features in this specification enable access to device sensors?](https://www.w3.org/TR/security-privacy-questionnaire/#sensor-data)

Not directly, but as mentioned in the answer to 2.1 above user agents may require developers to build an Isolated Web App to access API features which require a greater level of site security and trustworthiness. Currently, there is no proposal for an IWA-only API which enables access to device sensors.

## [2.9 Do features in this specification enable new script execution/loading mechanisms?](https://www.w3.org/TR/security-privacy-questionnaire/#string-to-script)

Yes, this proposal builds on top of Web Bundles to define [a new scheme](Scheme.md) from which resources can be loaded. Since bundled resources are served from a new scheme they are cross-origin to any existing web content. A combination of CSP and additional user agent restrictions also prevent this bundled content from being navigated to or loaded as a subresource by existing web content.

## [2.10 Do features in this specification allow an origin to access other devices?](https://www.w3.org/TR/security-privacy-questionnaire/#remote-device)

Not directly, but as mentioned in the answer to 2.1 above user agents may require developers to build an IWA to access API features which require a greater level of site security and trustworthiness. There is a proposal (explainer to-be-written) to allow an Isolated Web App using the WebUSB API to claim USB device interfaces which are normally [protected](https://wicg.github.io/webusb/#has-a-protected-interface-class).

## [2.11 Do features in this specification allow an origin some measure of control over a user agent’s native UI?](https://www.w3.org/TR/security-privacy-questionnaire/#native-ui)

As with installable web apps, an IWA may run in a minimal UI mode which hides the address bar and other native UI. While presentation details are up to user agents, the explainer recommends user agents use the standalone display mode to emphasize the app-like nature of these sites to establish the right mental model and expectations for users.

Since the origin (see the more detailed [scheme explainer](Scheme.md)) is not human-readable it is expected that the user agent’s native UI will display the name or short\_name from the Web Application Manifest instead. This is potentially untrustworthy data. Protection against spoofing another application is provided by the installation process confirming the name of the application being installed and user agent policies on trustworthy installation sources.

Similarly, the short name would appear in settings related to the IWA. For example, settings to view storage consumption and clear storage, or to allow/deny access to web features such as Geolocation.

As mentioned in the answer to 2.1 above user agents may require developers to build an IWA to access API features which require a greater level of site security and trustworthiness. The [Borderless](https://github.com/sonkkeli/borderless/blob/main/EXPLAINER.md) display mode proposal allows further control over the user agent’s native UI by allowing the window title and borders to be hidden entirely.

## [2.12 What temporary identifiers do the features in this specification create or expose to the web?](https://www.w3.org/TR/security-privacy-questionnaire/#temporary-id)

This feature does not create temporary identifiers.

## [2.13 How does this specification distinguish between behavior in first-party and third-party contexts?](https://www.w3.org/TR/security-privacy-questionnaire/#first-third-party)

This feature only creates a new kind of first-party context. Third-party content may be embedded within an Isolated Web App but it is covered by the web’s normal integrity guarantees rather than the stronger integrity guarantees provided by this proposal for first-party content. First-party content (i.e. content from the developer of the app) which is served dynamically (not part of the Web Bundle) is treated identically to third-party content since any HTTPS origin is cross-origin to an IWA origin. The mandatory Content Security Policy rules are designed to protect an app from exploits launched from compromised third-party content it embeds. 

## [2.14 How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?](https://www.w3.org/TR/security-privacy-questionnaire/#private-browsing)

An Isolated Web App could be launched inside a "Private Browsing" or "Incognito" mode which would clear state between sessions and prevent embedded third-party content from setting cookies.

## [2.15 Does this specification have both "Security Considerations" and "Privacy Considerations" sections?](https://www.w3.org/TR/security-privacy-questionnaire/#considerations)

The whole set of explainers are basically security considerations. The [updates explainer](Updates.md) specifically calls out security and privacy considerations for the update process.

## [2.16 Do features in your specification enable origins to downgrade default security protections?](https://www.w3.org/TR/security-privacy-questionnaire/#relaxed-sop)

While the intent of distributing an application as a signed Web Bundle is to allow auditing and inspection of the code there are nevertheless ways to inject dynamic behavior such as, fetching content over the network and interpreting it (eval() is blocked, but developers can always implement their own interpreter), loading dynamic content in an iframe, or “laundering” dynamic resources through a Service Worker fetch event handler so that they appear same-origin when evaluating Content Security Policy. The only way to mitigate these workarounds is to inspect the signed Web Bundle and detect these patterns.

As mentioned in the answer to 2.1 above, user agents may require developers to build an Isolated Web App to access API features which require a greater level of site security and trustworthiness. Three IWA-only APIs have been proposed which downgrade default security protections:

*   The [Direct Sockets API](https://wicg.github.io/direct-sockets/) allows an app to bypass CORS protections when making network requests, does not require connections to use TLS (bypassing mixed-content restrictions), and allows an app to connect to non-HTTP endpoints (bypassing protections against protocol confusion attacks).
*   The [Controlled Frame API](https://github.com/chasephillips/controlled-frame/blob/main/EXPLAINER.md) defines an element similar to an &lt;iframe> which allows the app to have a level of control over cross-origin embedded content equivalent to the control the app would have over same-origin embedded content.
*   [WebAuthn Remote Desktop Support](https://github.com/w3c/webauthn/wiki/Explainer:-Remote-Desktop-Support) defines an extension to the WebAuthn API which allows trusted remote desktop applications to impersonate another origin in a credential request.

## [2.17 How does your feature handle non-"fully active" documents?](https://www.w3.org/TR/security-privacy-questionnaire/#non-fully-active)

Probably not applicable, this feature doesn’t add any new JavaScript APIs which would have to react to non-"fully active" documents. While it defines a new method for loading resources, navigation between those resources should follow the existing page lifecycle behavior.

## [2.18 What should this questionnaire have asked?](https://www.w3.org/TR/security-privacy-questionnaire/#missing-questions)

It should have asked, "Does this feature depend on user agents or third-parties providing additional security services?"

As mentioned above some security and privacy issues are mitigated by the fact that a bundled and signed application can be inspected before installation, but who does that inspection? User agents could require that the Web Bundle's Integrity Block contain an additional signature that indicates the version has been submitted for an audit. Browser vendors could run their own audit services or trust third-party services. To avoid centralization, a system similar to Certificate Transparency logs could be used to provide public accountability when developers publish a new version of their application.
