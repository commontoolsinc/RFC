# Space

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [Irakli Gozalishvili], [Common Tools]

## Authors

- [Irakli Gozalishvili], [Common Tools]

## Abstract

A space can be defined as a namespace for a personal data. It is created locally on a user device by generating cryptographic keypair and identified globally via [did:key] of the generated public key. This document describes "memory" protocol which can be used to store data in space and retrieved from space.

- [Capabilities](#capabilities)
  - [`/memory/transact`](#memoryttransact)
  - [`/memory/query`](#memoryquery)
  - [`/memory/subscribe`](#memorysubscribe)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Protocol

The protocol is defined in terms of [UCAN] [capabilities](capability) that can be delegated by the space [did:key] to and delegate [did:key] giving them authorization to invoke it on their behalf.

### Subject

Every [UCAN] [delegation] and consequently [invocation] defined by the "memory" protocol MUST have a `subject` that is a [did:key] of the space.

> ℹ️ This setup allows protocol provider to verify whether invoked capability has being authorized by the space owner.

### Fact

Facts represent a certain memory stored in the space. It can be either in asserted or retracted, where later represents deleted. Fact associates a state (`is` field) with some resource (`of` field) identified by a URI. More than one, semantically distinct, state can be associated with a same resource through distinct [media type] identifiers (`the` field).

```ts
type Fact<The extends MIME, Of extends URI, Is extends JSONValue> =
  | Assertion<T, Of, Is>
  | Retraction<T, Of, Is>

type Assertion<The extends MIME, Of extends URI, Is extends JSONValue> = {
  the: The
  of: Of
  is: Is
  cause: Reference<Assertion<The, Of, Is> | Retraction<The, Of, Is>>
}

type Retraction<The extends MIME, Of extends URI, Is extends JSONValue> = {
  the: The
  of: Of
  is?: undefined
  cause?: Reference<Assertion<The, Of, Is>>
}

type URI = `${string}:${string}`
type MIME = `${string}/${string}`
```

Valid implementation MUST suppport `application/json` media type. More meda types may be added in the future.

### Capabilities

Protocol is defined in terms of following [UCAN] capabilities.

#### `/memory/transact`

Capability MAY be used to assert or retract set of [fact]s in a [subject] (memory) space. Capability [command] MUST be set to `/memory/transact`. Invoked capability MUST have `args` object conforming to a following schema

```ts
type Transaction = {
  changes: Changes
}

type Changes = {
  [of: URI]: {
    [the: MIME]: {
      [cause: string]: Assert | Retract | Claim;
    };
  };
}

type Assert = { is: JSONValue }
type Retract = { is?: undefined }
type Claim = true
```

Valid capability provider MUST update state transactionally adherencing to [compare and swap (CAS)][CAS] semantics, in order to uphold consistency guarntees. That implies that all assertions and retraction MUST be applied as long as `cause` of every supplied `{the, of}` pair corresponds to [merkle-reference] of the current [Fact]. Transaction where this invariant does not hold MUST be rejected in full, that is none of the enclosed assertions or retraction can be applied.

The `cause` of the new fact MUST be [merkle-reference] to the retracted [Fact] all the optional fields omitted.

#### Example

```json
{
  "iss": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "sub": "did:key:z6MkffDZCkCTWreg8868fG1FGFogcJj5X6PY93pPcWDn9bob",
  "cmd": "/memory/transact",
  "args": {
    "changes": {
      "uuid:5d59a2ff-5aa7-419d-8b3a-547063a2fd23": {
        "application/json": {
          "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
            "is": { "hello": "world" }
          }
        }
      }
    }
  },
  "nonce": {"/": {"bytes": "TWFueSBopvcs"}},
  "meta": {},
  "exp": 1697409438,
  "prf": [
    {"/": "zdpuAzx4sBrBCabrZZqXgvK3NDzh7Mf5mKbG11aBkkMCdLtCp"}
  ]
}
```

#### `/memory/query`

Capability MAY be used to query space in order to retrieve desired facts from the [subject] (memory) space. Capability command MUST be set to `/memory/query`. Inovked capability MUST have `args` object conforming to a following schema

```ts
type Query = {
  select: Selector
  since?: number
}

type Selector {
  [of: URI]: {
    [the: string]: {
      [cause: string]: {
        is?: {}
      }
    }
  }
}
```

The `since` field MAY be specified to limit query to select only facts that have being asserted after that number of transactions. It defaults to `0` if omitted.

The `of`, `the` and `cause` of the `select` CAN be set to select facts that match those values. Each of these components CAN be `"_"` value to match any value.

#### Example

Selects fact for the `uuid:5d59a2ff-5aa7-419d-8b3a-547063a2fd23` corresponding to the `application/json` media type.

```ts
{
  "iss": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "sub": "did:key:z6MkffDZCkCTWreg8868fG1FGFogcJj5X6PY93pPcWDn9bob",
  "cmd": "/memory/transact",
  "args": {
    "select": {
      "uuid:5d59a2ff-5aa7-419d-8b3a-547063a2fd23": {
        "application/json": {
        }
      }
    }
  },
  "nonce": {"/": {"bytes": "TWFueSBopvcs"}},
  "meta": {},
  "exp": 1697409438,
  "prf": [
    {"/": "zdpuAzx4sBrBCabrZZqXgvK3NDzh7Mf5mKbG11aBkkMCdLtCp"}
  ]
}
```

Selects all facts that have state associated with `application/commit+json` media type.

```ts
{
  "iss": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "sub": "did:key:z6MkffDZCkCTWreg8868fG1FGFogcJj5X6PY93pPcWDn9bob",
  "cmd": "/memory/transact",
  "args": {
    "select": {
      "_": {
        "application/commit+json": {
        }
      }
    }
  },
  "nonce": {"/": {"bytes": "TWFueSBopvcs"}},
  "meta": {},
  "exp": 1697409438,
  "prf": [
    {"/": "zdpuAzx4sBrBCabrZZqXgvK3NDzh7Mf5mKbG11aBkkMCdLtCp"}
  ]
}
```

#### `/memory/subscribe`

[Common Tools]: https://common.tools/
[Irakli Gozalishvili]: https://github.com/gozala
[did:key]:https://w3c-ccg.github.io/did-key-spec/
[UCAN]:https://github.com/ucan-wg/spec
[capability]:https://github.com/ucan-wg/spec?tab=readme-ov-file#capability
[delegation]:https://github.com/ucan-wg/delegation
[invocation]:https://github.com/ucan-wg/invocation
[JSON]:https://www.json.org/json-en.html
[media type]:https://www.iana.org/assignments/media-types/media-types.xhtml
[subject]:#subject
[fact]:#fact
[command]:https://github.com/ucan-wg/spec?tab=readme-ov-file#command
[CAS]:https://en.wikipedia.org/wiki/Compare-and-swap
[merkle-reference]:https://github.com/Gozala/merkle-reference/blob/main/docs/spec.md
