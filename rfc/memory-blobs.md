# Binary Data Support for Memory Protocol

## Editors

- [Irakli Gozalishvili], [Common Tools]

## Authors

- [Irakli Gozalishvili], [Common Tools]


## Abstract

This RFC extends the [memory protocol] with a support of binary data like images, audio, video, and other non-JSON media types by adopting [DAG-JSON] format. It extends supported media types to cover wide range of data formats by utilizing binary data in values.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## Extension to Memory Protocol

This RFC extends the Memory Protocol by defining how to handle binary data in the existing fact structure. It builds upon the core concepts of the [memory protocol] while providing specific guidance for binary content.

### Binary Data Representation

Binary data MUST be represented using the [DAG-JSON bytes] format, which encodes binary data in a JSON-compatible format.

The binary representation follows this structure:

```js
{ "/": { "bytes": "aGVsbG8gd29ybGQ=" } }
```

This format encodes the binary data as a base64 string within a structured object that conforms to the [DAG-JSON] specification. This approach maintains compatibility with JSON-based systems while enabling the representation of arbitrary binary data.

### Blob

Binary blobs in the [memory protocol] can be modeled like a fact utilizing on of the well known [media type]s for binary data.

#### `of` - Blob Identifier

Blobs MAY be identified using arbitrary URI that conforms to the URI specification.

#### `the` - Blob Media Type

The `the` field MUST specify the appropriate [media type] for the binary content (e.g., "image/png", "audio/mp3", "application/pdf").

The default media type for the unknown binary blob SHOULD be `application/octet-stream`.

#### `is` - Blob Content

The `is` field MUST be in [DAG-JSON bytes] format. Assertions for facts with binary content types MUST fail transaction.

#### `cause` - Cusal Reference

The `cause` field follows the same rules as in the core [Memory Protocol], ensuring proper causal chain maintenance for binary content.

### Example Assertions with Binary Content

#### Image Assertion
```json
{
  "the": "image/png",
  "of": "photo:5d59a2ff-5aa7-419d-8b3a-547063a2fd23",
  "is": { "/": { "bytes": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+P+/HgAFeAJ5gZ5fCQAAAABJRU5ErkJggg==" } },
  "cause": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5"
}
```

#### Audio Assertion
```json
{
  "the": "audio/mpeg",
  "of": "track:9b2c8e5a",
  "is": { "/": { "bytes": "SUQzBAAAAAAAI1RTU0UAAAAPAAADTGF2ZjU4Ljc2LjEwMAAAAAAAAAAAAAAA/+M4wAAAAAAAAAAAAEluZm8AAAAP..." } },
  "cause": "ca5kbd8su5emr6n6vvwdz8jiu6pnfvlbwizccqnd6x5idq7em5z6tfls6"
}
```

#### Generic Binary Data Assertion (Octet Stream)
```json
{
  "the": "application/octet-stream",
  "of": "attachment:a7e22e3c",
  "is": { "/": { "bytes": "AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8gISIjJCUmJygpKissLS4vMDEyMzQ1Njc4OTo7PD0+P0BBQkNERUZH..." } },
  "cause": "da6lce9tv6fnr7o7wwxez9kjv7qogwmcxjacdrod7y6jer8fn6a7ugmt7"
}
```

### Binary Data embedded in JSON Facts

There are two approaches to embedding binary data in JSON facts

#### Inline Embedding

Binary data can be inlined in JSON facts using the [DAG-JSON bytes] representation:

```json
{
  "the": "application/json",
  "of": "profile:123",
  "is": {
    "name": "John Doe",
    "avatar": { "/": { "bytes": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+P+/HgAFeAJ5gZ5fCQAAAABJRU5ErkJggg==" } }
  },
  "cause": "da6lce9tv6fnr7o7wwxez9kjv7qogwmcxjacdrod7y6jer8fn6a7ugmt7"
}
```

While this approach is simple, it has limitations:
- Implementations SHOULD enforce reasonable size limits for inline binary data
- Large binary data embedded directly can impact performance and transaction size
- This approach makes it harder to reference the same binary content from multiple facts

#### Recommended Approach: Blob References

The recommended approach is to create separate facts in form of a blob and reference it in JSON.

This enables:
- More efficient storage and transmission
- Separate loading of binary content when needed
- Reuse of the same binary content across multiple references
- Mutable content of the blob

Example reference to a blob in a JSON fact:

```json
{
  "the": "application/json",
  "of": "profile:123",
  "is": {
    "name": "John Doe",
    "picture": {
      "@": {
        "the": "image/png",
        "of": "attachment:a7e22e3c"
      }
    }
  },
  "cause": "da6lce9tv6fnr7o7wwxez9kjv7qogwmcxjacdrod7y6jer8fn6a7ugmt7"
}
```

### Higher-Level Implementation

The [memory protocol] consumer library SHOULD provide transparent handling of blobs. For example, a JS implementation could allow passing native `Blob` instances

```js
workspace.transact([
  assert({
    the: 'application/json',
    of: 'profile:',
    is: {
      name: 'Mr Nobody',
      picture: new Blob(uploadedImage, { type: 'image/png' })
    }
  })
])
```

Which would automatically be transaction into multiple facts:

```json
{
  "changes": {
    "profile:": {
      "application/json": {
        "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
          "name": "Mr Nobody",
          "picture": {
            // TODO: Align this what cells do here
            "@": {
              "the": "image/png",
              "of": "attachment:a7e22e3c"
            }
          }
        }
      }
    },
    "attachment:a7e22e3c": {
      "image/png": {
        "ba4jAzx4sBrBCabrZZqXgvK3NDzh7Mf5mKbG11aBkkMCdLtCp": {
          "is": { "/": { "bytes": "iV...AAAABJRU5ErkJggg==" } }
        }
      }
    }
  }
}
```

Web native `Blob` represents binary content that can be loaded into memory asynchronously making them an excellent fit for the high level API. E.g querying
above asserted fact could return a fact with `is` field bound to a `Blob` instance that could be loaded separately on-demand.

This approach provides a clean developer experience while maintaining efficient binary data handling.


### Query Support for Binary Content

When using the `/memory/query` capability:

1. Queries for facts with binary content SHOULD follow the same pattern as queries for JSON content.

1. Query responses containing binary data MUST follow the same bytes representation format as assertions.

Example query for binary content:
```json
{
  "select": {
    "the": "image/png",
    "of": "uuid:5d59a2ff-5aa7-419d-8b3a-547063a2fd23"
  }
}
```

### Transaction Format for Binary Content

The transaction format for binary content follows the standard `/memory/transact` pattern defined in the core Memory Protocol:

```json
{
  "changes": [
    {
      "uuid:5d59a2ff-5aa7-419d-8b3a-547063a2fd23": {
        "image/png": {
          "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
            "is": { "/": { "bytes": "iVBORw0KGgoAAAAN..." } }
          }
        }
      }
    }
  ]
}
```

## Compatibility

This extension is backward compatible with the core [memory protocol].

## Implementations

Implementations of the Memory Protocol that support this binary extension MUST:

1. Support at least the following common MIME types:
   - image/jpeg
   - image/png
   - image/gif
   - image/webp
   - audio/mpeg
   - audio/wav
   - video/mp4
   - application/pdf
   - application/octet-stream (for generic binary data)

2. Provide clear error messages when rejecting binary content that doesn't meet the requirements.

3. Document any size limitations or supported MIME types beyond the required minimum set.

4. Support efficient querying and retrieval of binary content.

#### Commits

Binary data in commits SHOULD be substituted with an [IPLD Link] of the binary data. This enables retaining verifible nature of the [memory protocol] without having to duplicate binary data inside the commits.

[DAG-JSON]:ipld.io/specs/codecs/dag-json/spec/
[Memory Protocol]:./memory.md
[media type]:https://www.iana.org/assignments/media-types/media-types.xhtml
[DAG-JSON bytes]:https://ipld.io/specs/codecs/dag-json/spec/#bytes
