# Identity

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [Irakli Gozalishvili], [Common Tools]

## Authors

- [Irakli Gozalishvili], [Common Tools]

## Abstract

The Identity system forms the foundation of the Open Ocean Protocol, establishing self-sovereign identities that enable users to own and control their digital presence. This specification defines how identities are created, verified, and used to establish ownership and control across the protocol ecosystem.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Concepts

### Self-Sovereign Identity

In the Open Ocean Protocol, identities are fully self-sovereign, meaning:

1. **User-controlled** - Identities are created and controlled by users without requiring permission from any central authority
2. **Cryptographically secure** - Identities are based on public key cryptography, allowing verifiable claims of ownership
3. **Portable** - Identities can be used across different applications, services, and devices
4. **Private by default** - Users control what information is associated with their identity and shared with others

### Decentralized Identifiers (DIDs)

The protocol uses [DID:Key](https://w3c-ccg.github.io/did-method-key/) as the primary identifier mechanism. Each identity is represented by a [did:key] identifier, which:

1. Is derived directly from a cryptographic public key
2. Can be generated offline without relying on any registry or authority
3. Enables verification of digital signatures associated with the identity
4. Can be resolved to retrieve the corresponding public key


## Implementation

### Identity Creation

The identity creation process follows a hierarchical approach that ensures both security and usability.

#### Account ID

The root of identity in the Open Ocean Protocol is the account ID:

1. Account keys are created using the [Web Authentication API]
2. This leverages platform authenticators (such as biometrics, secure enclaves, or hardware tokens)
3. The resulting public key becomes the root identifier for the user
4. A [did:key] identifier is derived from this public key
5. The private key never leaves the secure element, enhancing security

#### Session Keys

Session keys provide a practical layer for daily operations:

1. Session keys are derived from the account key using the [WebAuthn PRF extension]
2. This derivation happens within the secure element without exposing the account's private key
3. The session key can be used across multiple applications and sessions
4. Different contexts can use different session keys from the same account key
5. Ed25519 is the RECOMMENDED key type for session keys

#### Space Keys

Each user space has its own dedicated key:

1. Space keys are deterministically derived from the session key and a seed phrase (typically the space petname)
2. This ensures that each space has a unique identity while still being recoverable
3. The derivation process is deterministic, allowing recreation of all space keys when needed
4. Ed25519 is the RECOMMENDED key type for space keys

### Key Recovery

A key advantage of this hierarchical approach is simplified recovery:

1. Users don't need to manually manage or backup their cryptographic keys
2. Access to all user spaces can be recovered on a new device through WebAuthn authentication
3. The deterministic derivation process recreates the exact same keys when provided with the same inputs
4. This enables seamless device migration without requiring complex key management

### Authorization

Identities serve as the root of authority for:

1. **Space ownership** - Each personal data space is owned by exactly one identity (via its space key)
2. **Authorization delegation** - The identity owner can delegate specific capabilities to applications and services using [UCAN]
3. **Data verification** - All operations within the protocol can be cryptographically verified against the identity's public key

### Credential Issuance

The identity system supports the issuance and verification of credentials:

1. **Self-attested claims** - Users can make verifiable claims about their own identity
2. **Third-party attestations** - External parties can issue credentials attesting to various attributes of an identity
3. **Selective disclosure** - Users can choose which aspects of their identity to reveal in different contexts



[Irakli Gozalishvili]: https://github.com/gozala
[Common Tools]: https://commontool.org
[did:key]: https://w3c-ccg.github.io/did-method-key/
[UCAN]: https://github.com/ucan-wg/spec/
[Web Authentication API]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API
[WebAuthn PRF extension]: https://github.com/w3c/webauthn/wiki/Explainer:-PRF-extension
