# Isolated Web App Updates

## Purpose

Isolated Web Apps load resources from a Web Bundle that is stored on the user’s computer rather than from a server. This design necessitates a system for determining when a new version of that Web Bundle should be downloaded.

## Proposed Design

The Web Application Manifest for an Isolated Web App will include two new fields related to updates,

*   the `version` field allows the app publisher to identify a Web Bundle as containing a particular version of the application
*   the `update_manifest_url` field tells the user agent where to look for updated versions of the application

The resource served at the `update_manifest_url` is a JSON document listing versions of the application that are available and the URL from which the Web Bundle containing that version can be downloaded.

In order to check for updates the user agent follows the following steps,

1. Fetch the resource from `update_manifest_url` and parse it as a JSON document.
2. Let _selectedVersion_ and _selectedSrc_ be initialized to `null`.
3. For each object in the document’s `versions` entry,
    1. Let _version_ be the object’s `version` field, skipping this item if the field does not exist.
    2. Let _src_ be the object’s `src` field, skipping this item if the field does not exist.
    3. If _version_ is not a valid version identifier or _src_ is not a valid URL, skip this item.
    4. If _selectedVersion_ is `null` or _selectedVersion_ is less than or equal to _version_, set _selectedVersion_ to _version_ and _selectedSrc_ to _src_. 
4. If _selectedVersion_ is `null`, abort these steps.
5. If _selectedVersion_ is less than or equal to the currently installed version, abort these steps.
6. Fetch the resource from _selectedSrc_.
7. If the resource is not a valid Signed Web Bundle, abort these steps.
8. If the Web Bundle ID of the resource is not equal to that of the currently installed app, abort these steps.
9. Load `/manifest.webmanifest` from the Web Bundle and parse it as a Web Application Manifest.
10. If the resource is not found or fails to parse, abort these steps.
11. If the `version` field in the manifest from the Web Bundle is not equal to _selectedVersion_, abort these steps.
12. Replace the Web Bundle for the current installed app with the downloaded resource.

## Extensions to the Web Application Manifest

### `version` field

The `version` field contains a string indicating the version of the application. This field is only used and mandatory for Isolated Web Apps, but has no meaning for normal Web Applications. For the purposes of updating version numbers only need to be ordered. For example, a plain integer versioning scheme (e.g. 1, 2, 3, etc.) would be sufficient however for ease of use and to provide for more complex comparisons to be performed on versions a structured format such as “A.B.C.D”, where A, B, C and D are positive integers (or zero) is much more useful. A format such as [Semantic Versioning](https://semver.org/) could also be adopted as it provides even more expressivity.

Whatever the structure of the version string, a comparison algorithm must be defined so that sorting of and comparisons between versions can be performed in the algorithms defined in this specification. Lexical comparison between version strings is inappropriate.

### `update_manifest_url` field

The optional `update_manifest_url` field indicates the location where the Web Application Update Manifest can be found. It must be an absolute HTTPS URL, or for testing purposes a localhost HTTP URL.

## Web Application Update Manifest

The Web Application Update Manifest is a document listing all application versions available for download. It consists of a single JSON object with a single field called `versions` which is a list. Each entry in the list is an object containing two fields, `version` and `src`. The `version` field is a string in the same format as the proposed Web Application Manifest `version` field. The `src` field is a URL and is resolved relative to the location of the manifest. It must be an HTTPS URL or a localhost HTTP URL, for testing purposes. For example a manifest hosted at `https://developer.example.com/app/updates.json` could contain,

```
{
  "versions": [
    {
      "version:" "5.2.17",
      "src": "https://cdn.example.com/app-package-5.2.17.swbn"
    },
    {
      "version": "6.1.13",
      "src": "v6.1.13/package.swbn"
    },
    {
      "version": "7.0.6",
      "src": "v7.0.6/package.swbn"
    }
  ] 
}
```

For forward compatibility, implementations MUST ignore keys they do not recognize. When multiple objects in the `versions` have the same `version` field the algorithm above selects the last instance to allow future versions to specify more precise matching rules based on currently undefined properties (e.g. package language).

## Alternatives Considered

### Direct link to Web Bundle

A significantly simpler solution which mimics how normal site resources are updated would be to have the Web Application Manifest link directly to a Web Bundle. The problem with this approach is that,

*   Without version numbers it doesn’t provide robust protection against rollbacks, as the value of the Last-Modified header is not part of the Web Bundle.
*   Enterprise administrators want the ability to control when new versions are rolled out, which requires multiple versions to be available at once. The update steps above could be augmented to filter available versions based on administrator-defined policy.

## Security Considerations

The Integrity Block prepended to a Web Bundle to form a Signed Web Bundle provides integrity for a single version of an application, however an application which can’t be updated is not useful and so this document proposes a mechanism for doing so. The security of this mechanism relies on the secrecy of the private key used to generate the Integrity Block in order to provide assurance that the real developer of the application has provided the new version of the Web Bundle and on the comparability of the proposed version field in the Web Application Manifest to provide rollback protections. While the Web Application Update Manifest must be delivered over HTTPS to provide some protection against network-based attackers it is insufficient to trust any Web Bundle loaded over HTTPS from the developer’s origin as the threat model for Isolated Web Apps includes the possibility for temporary compromise of the developer’s website.

## Privacy Considerations

The update algorithm requires fetching resources over the network and so care should be taken to do so in a privacy-preserving way. The first step is to ensure all fetches are done in credentialless mode. Even so, requesting the Web Application Update Manifest reveals the public IP addresses of any systems where an application is installed to a developer-controlled server. We can safely assume that when the application itself is launched it may also make network requests and so some privacy can be achieved by trying to match the frequency of update requests to how often the user actually uses the application to prevent an unused application from being used as a tracking vector. This must be balanced with ensuring that application updates are installed in a timely manner so that security vulnerabilities can be patched. A user agent could also request the manifest via a proxy service however this may prevent updates to internally-deployed applications which are not available outside of a private network.
