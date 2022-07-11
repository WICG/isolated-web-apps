# `isolated-app:` Scheme Explainer

This document provides a more detailed look at the `isolated-app:` scheme, which is part of the proposal for [Isolated Web Apps](./README.md).
Isolated Web Apps are a new proposal to build applications using web technologies, but with additional security properties compared to normal web pages.
Isolated Web Apps bundle all their contents inside a web bundle that is then [signed to make its integrity verifiable](https://github.com/WICG/webpackage/blob/main/explainers/integrity-signature.md).
A key difference between Isolated Web Apps and normal web pages is that Isolated Web Apps do not rely on DNS name resolution and HTTPS certificate authorities.
Instead, they need to be explicitly downloaded and installed by the user, such as from an app store or via enterprise configuration.
A user agent can verify the integrity of an Isolated Web App by checking the signature and comparing its corresponding public key to a list of known trusted public keys.

## `isolated-app:` Scheme

Since Isolated Web Apps do not use HTTPS certificate authorities and also need to be independent of domains and name resolution, a new URL scheme is needed to signify this difference.
The tentative name of this new scheme is `isolated-app:`, highlighting how Isolated Web Apps are isolated from each other and the web through enforced Content Security Policy and Cross-Origin Isolation.
When loading an `isolated-app:` URL, the user agent checks the signatures of the underlying web bundle and whether it trusts the public key that was used to sign it.
A user agent might trust, for example, public keys configured via enterprise policy, public keys of known distribution mechanisms/stores, or separately allow-listed public keys of individual Isolated Web Apps.
`isolated-app:` URLs are considered Secure Contexts, just like HTTPS pages, and thus have access to APIs like Service Workers. `isolated-app:` URLs look like so:

```
isolated-app://signed-web-bundle-id/path/inside/app.js?some-query#foo
^           ://^                   /^                 ?^         #^
scheme         opaque host          path               query      fragment
```

### Host

URLs with the `isolated-app:` scheme use a Signed Web Bundle ID (see next section) as their opaque host, as defined in the [URL standard](https://url.spec.whatwg.org/).
The Signed Web Bundle ID being the host adds the requirement that it must be a valid hostname (see [RFC 3986](https://www.rfc-editor.org/rfc/rfc3986)). This requirement is satisfied by using [base32 encoding](https://datatracker.ietf.org/doc/html/rfc4648), with padding removed.
It is the user agent’s responsibility to map the Signed Web Bundle ID contained in the opaque host to an installed Isolated Web App and its associated Web Bundle. The URL path is then used to fetch resources as relative URLs from the bundle.

### Port & Credentials

Isolated Web App URLs do not use usernames, passwords, or ports.
All three of these components are therefore always `null`.
Isolated Web App URLs with a port or credentials must not be loaded and must result in an HTTP error.

### Path, Query, Fragment

These work like one would expect from HTTP(S) URLs, as described in the URL specification.

## Signed Web Bundle IDs

Isolated Web Apps are identified by their _Signed Web Bundle ID_.
The Signed Web Bundle ID is an ASCII string which contains an identifier followed by a suffix that represents the type of identifier.
The general format of Signed Web Bundle IDs looks like this:

```
lowercase(base32EncodeWithoutPadding([identifier] [identifier type] [identifier type length]))
                                                  [ ------------ suffix ------------------ ]
```

### Suffix

The suffix consists of two parts: The last byte indicates how many bytes are used to describe the identifier type, and the `n` bytes before it are used to describe the identifier type.
Currently, the following two suffixes are defined:

`0x00 0x00 0x02`: This suffix indicates that the ID is a user-agent-specific value that can be used for testing and development purposes.
For example, loading a Web Bundle from the local filesystem which is not signed.

`0x00 0x01 0x02`: This suffix is used for [Ed25519](https://datatracker.ietf.org/doc/html/rfc8032) public keys as Signed Web Bundle ID.
It is prefixed by the 32-byte representation of the Ed25519 public key that is used to sign the web bundle (see [this explainer](https://github.com/WICG/webpackage/blob/main/explainers/integrity-signature.md) for more details about the signing process).
Since this Signed Web Bundle ID is only dependent on the signing key, it makes it possible to have the same Signed Web Bundle ID regardless of the distribution mechanism of the Isolated Web App, without requiring a central authority to manage keys.

### Encoding

Given an identifier and the suffix representing the type of the identifier, both must be concatenated and then base32-encoded.
Potential padding introduced by the base32 encoding must be discarded and the result transformed into lowercase.
The resulting string is a Signed Web Bundle ID.

### Decoding

Given a Signed Web Bundle ID, it must first be transformed into uppercase.
If the string's length is not a multiple of 8, a number of padding characters (`=`) must be added until its length is a multiple of 8.
Then, the string must be base32-decoded.
Next, the last byte must be read to determine the length of the identifier type.
Assuming that there are `n` bytes in total and that the last byte contains the number `l`, the `identifier` can be reconstructed by reading bytes `[0, n-l-1)`, and the `identifier type` can be reconstructed by reading bytes `[n-l-1, n-1)`.

### Example for Ed25519 keys

Public Ed25519 key (hex, 32 bytes):
```
01 23 43 43 33 42 7A 14 42 14
a2 b6 c2 d9 f2 02 03 42 18 10
12 26 62 88 f6 a3 a5 47 14 69
00 73
```

Public key and suffix (hex, 35 bytes):
```
01 23 43 43 33 42 7A 14 42 14
a2 b6 c2 d9 f2 02 03 42 18 10
12 26 62 88 f6 a3 a5 47 14 69
00 73 00 01 02
```

Base32-encoded public key and suffix (56 chars long):
```
AERUGQZTIJ5BIQQUUK3MFWPSAIBUEGAQCITGFCHWUOSUOFDJABZQAAIC
```

Since the length of the public key and suffix is a multiple of 5, no padding is generated that would need to be removed.

Signed Web Bundle ID (56 chars long):
```
aerugqztij5biqquuk3mfwpsaibuegaqcitgfchwuosuofdjabzqaaic
```