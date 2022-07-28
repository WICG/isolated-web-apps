# High Watermark Permissions Explainer

## Introduction

The dynamic nature of permissions on the web makes reviewing their use more difficult, and makes it impossible to provide guarantees about which capabilities a site might use. Unlike other platforms like Android or iOS, the web doesn’t force developers to declare required permissions up-front. While Permissions Policy can be used to restrict or delegate permissions to child frames, top-level frames can request access to any permission their context is compatible with.

[Isolated Web Apps](./README.md) rely on packaging and third-party distribution as a central piece of their security model. The inability to audit permissions on the web makes it more difficult for distribution channels (e.g. an enterprise admin or store) to verify that apps aren’t abusing the powerful capabilities provided to them, or provide guarantees that an app’s behavior won’t change without an explicit update.

This document proposes introducing a new Web App Manifest field that will allow Isolated Web Apps to specify the maximal set of permissions they could request. These permissions will not be automatically granted to the app, but must be specified in the manifest in order for an app’s just-in-time permission requests to be granted.

Specifying the high watermark for permissions in the manifest will allow distribution channels to more accurately review apps, and communicate to the user with more confidence and clarity what features and capabilities could be used by the app.

## Previous Proposals

### Manifest Fields

There have been several proposals to add a permissions allowlist to the manifest (see [w3c/manifest](https://github.com/w3c/manifest) issues [75](https://github.com/w3c/manifest/issues/75), [395](https://github.com/w3c/manifest/issues/395), [798](https://github.com/w3c/manifest/issues/798)), none of which progressed to a specification or prototype. Most of the concerns about these proposals stem from the fact that manifest loading is asynchronous, which raises a couple issues:

1.  If the purpose of the permissions field is to limit the capabilities of a page, then that can only be meaningfully achieved by applying the restrictions pre-install as well (otherwise uninstalling the app would bypass the restriction). This isn’t feasible because the manifest isn’t available when the page is first loaded, which introduces a race between loading the manifest and running scripts that use the features being restricted.
2.  Manifest updates are also asynchronous and aren’t guaranteed to take effect when other related code changes do. For example, if you were using the manifest to allowlist permissions for your PWA and wanted to introduce a feature that used a new policy-controlled API, the manifest change to allow the permission and the code to use it wouldn’t update at the same time. This potential divergence between the app’s scripts and its manifest makes the state of the app harder to reason about and forces developers to deal with partially updated apps.

In a packaged context we avoid these issues by tying the manifest to a specific version of the app, and preventing an app from being launched before it is installed.

Another concern with manifest-based allowlists in PWAs is that developers can create multiple apps on a single origin, but permissions apply to the entire origin. The browser can’t limit an app’s permissions only within its scope without breaking the existing permissions model, and even if the browser attempted to do so, the app could simply navigate to a same-origin URL outside its scope to request a permission it hadn’t declared. This problem doesn’t exist for Isolated Web Apps because their scope is required to be the entire origin (i.e. “/”).

All of the aforementioned problems are solved in Isolated Web Apps through packaging and manifest restrictions. This proposal relies on these Isolated Web App properties and does not attempt to solve these issues for PWAs in general.

### Origin Policy

From the [Origin Policy Explainer](https://github.com/WICG/origin-policy/tree/7354568d2492d6847f68da21df46274df50c374a#origin-policy): “Origin policy is a web platform mechanism that allows origins to set their origin-wide configuration in a central location, instead of using per-response HTTP headers.” One of the policies that can be configured within an Origin Policy is [Permissions Policy](https://github.com/WICG/origin-policy/blob/7354568d2492d6847f68da21df46274df50c374a/policy-format.md#feature-policy), which would provide a mechanism for pre-declaring which permissions are allowed within an app. We considered using this to implement high watermark permissions, but abandoned it when Origin Policy was [put on hold](https://github.com/WICG/origin-policy/blob/2683f92226052dd363b84620247cdb5f949a4b75/README.md), primarily due the same [asynchronicity issues](https://docs.google.com/document/d/1jptq14gPpuBt3933-Hns2wwrHIW6qpo3xu6Xkfs12N4/edit#) the aforementioned manifest proposals had.

## Proposal

There are two mechanisms to control permissions on the Web today: [Permissions](https://www.w3.org/TR/permissions/) and [Permissions Policy](https://www.w3.org/TR/permissions-policy-1/). Permissions allow the *user* to control whether an origin has access to a permission-controlled feature, while Permissions Policy allows *developers* to control whether an origin can access or request access to a policy-controlled feature. An API is only accessible to a page if both permission mechanisms allow it.

It is a non-goal to automatically grant access to all permissions listed in the manifest – we want the user to remain in control of the app’s capabilities. Because of this, along with the fact that manifest restrictions represent a developer choice rather than a user choice, Permissions Policy is better suited as a mechanism to enforce these permission restrictions.

We propose adding a new `permissions_policy` field to the Web App Manifest spec, which would contain a [policy directive](https://www.w3.org/TR/permissions-policy-1/#policy-directive) that maps [policy-controlled features](https://www.w3.org/TR/permissions-policy-1/#policy-controlled-feature) to an allowlist of origins. This field would define the default Permissions Policy for all top-level frames in an Isolated Web App. The [default allowlist](https://www.w3.org/TR/permissions-policy-1/#default-allowlists) for all policy-controlled features in Isolated Web Apps should default to 'none', and can only be expanded through the new permissions_policy manifest field. Child frame Permission Policies would inherit from their parent frame as they do today.

```
"permissions_policy": {
   "geolocation": [ "self", "https://map.example.com" ],
   "fullscreen": [ "*" ]
}
```

In this example, geolocation would be made available to any of the app’s top level documents, embedded content on the same origin as the app, and to embedded content loaded from map.example.com. The fullscreen permission would be available for request to the app’s top level documents, and any embedded content. These two permissions would not automatically be granted on app install, and any request for a permission other than geolocation or fullscreen will be automatically denied.

While the goal of this proposal is to create a high watermark for the permissions that an app can request, the Permissions Policy specified in the manifest may also be used by the browser to inform the installation UI, or customize site settings pages with the relevant permissions in much the same way [First Run Permissions Prompt](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/InstallTimePermissionsPrompt/Explainer.md) proposes (discussed more below).

### Interactions with the Permissions-Policy Header

WebBundles, the packaging format used by Isolated Web Apps, supports specifying response headers, which means developers could send a Permissions-Policy header in their response. These headers could differ from the manifest’s declared policy in both the permissions specified, and the allowlists. If a Permissions-Policy header is received, the header and manifest policies should be combined as follows:

*   For the **set of permissions** declared, the browser should use the intersection of the permissions specified in the manifest and in the headers, and ignore anything that falls outside of that.
*   For the **origins in the allowlist** for each permission, the browser should use the intersection of the origins specified in the manifest and in the headers and ignore anything that falls outside of that. It’s worth noting that if an iframe of a certain origin has access to a permission, it is able to grant access to that same permission to any of its own embedded content of any origin it decides.

### Default Allowlists

As stated above, this proposal changes the default allowlist for all policy-controlled features in Isolated Web Apps from ‘\*’ or ‘self’ to ‘none.’ This ensures that any new permissions added after an app is published will not automatically be granted to the app. However, it also means that any new permissions added for existing features would break apps until they’re updated to request the permission. There is precedence for this; the ability to send SharedArrayBuffers via postMessage was placed behind a new [cross-origin-isolation policy](https://github.com/whatwg/html/issues/5435) in 2020. This was safe given how Permissions Policy worked at the time, but could have broken Isolate Web App main frames with these proposed changes.

It is an open question whether we want to update the Permissions Policy spec to allow 'none' as a default allowlist for Isolated Web Apps, and have the defaults be configurable on a per-feature basis rather than denying all by default. This approach would allow us to enable a subset of existing features by default in Isolated Web Apps (such as `ch-ua-\*`, `sync-xhr`, or `cross-origin-isolation`), while letting us make backwards compatibility decisions on a case-by-case basis for future features.

## Related Work

WebExtensions is another packaging format for web-like content that includes permissions in its manifest. It splits permissions into two fields: [permissions](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/manifest.json/permissions) and [optional_permissions](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/manifest.json/optional_permissions). permissions specifies mandatory permissions that will automatically be granted upon extension installation, while optional_permissions specifies permissions that will not automatically be granted, but may be requested by the extension when needed. The permissions_policy field proposed here is similar to the optional_permissions field, except that it operates on policy-controlled features rather than permissions-controlled features.

Microsoft recently published an explainer for an API called [First Run Permissions Prompt](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/InstallTimePermissionsPrompt/Explainer.md). This proposal also introduces a new permissions-related manifest field, but instead of limiting the app to only the permissions specified in the manifest, it would be used by the browser only to provide a richer install UI and potentially allow the user to pre-grant access to the specified permissions, and would be usable by all PWAs. It is orthogonal to this proposal.
