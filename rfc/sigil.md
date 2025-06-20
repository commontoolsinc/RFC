# Sigil Protocol: Cross-Fact References for Memory Protocol

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [Irakli Gozalishvili], [Common Tools]

## Authors

- [Bernhard Seefeld], [Common Tools]

## Abstract

This RFC extends the [Memory Protocol] with a standardized system for creating references between facts in user-controlled spaces. The protocol defines five fundamental types of references:

1. **Inline references**: Embed data directly within facts using [IPLD data model] types.
2. **Immutable references**: Reference data by content using [merkle reference]s which are valid [IPLD Links].
3. **Mutable references**: Reference data by its memory address using `link` sigils.
4. **Blob references**: Reference binary data interpreted as `Blob` instances
5. **File references**: Reference binary data interpreted as `File` instances.

These reference types enable sophisticated data modeling including binary content references, computed data sources, and relational queries within facts, while preserving the causal integrity of the memory protocol. References are resolved by the [Schema Query Protocol], which provides the `/memory/graph/query` capability for advanced querying with automatic resolution.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## Motivation

The [Memory Protocol] provides a robust foundation for storing and querying facts in user-controlled spaces. However, applications need standardized ways to reference data across fact boundaries with different consistency guarantees:

- **Immutable references**: Reference data that should never change, with cryptographic integrity guarantees
- **Mutable references**: Reference data that can be updated over time, with automatic propagation of changes
- **Binary content**: Handle binary data and file metadata consistently within the fact model
- **Computed relationships**: Establish dynamic relationships and computed values between facts
- **Reactive systems**: Build systems that respond to changes in referenced data

This RFC addresses these needs by defining five fundamental reference types that provide different consistency and mutability guarantees, enabling applications to choose the appropriate reference semantics for their use cases.

## IPLD Data Model and DAG-JSON Convention

This protocol uses the [IPLD data model] which provides a canonical way to represent structured data that is encoding-agnostic. The IPLD data model supports the same basic types as JSON (null, boolean, integer, float, string, list, map) plus additional types for binary data and cryptographic links.

For JSON encoding, we follow the [DAG-JSON] specification which provides a standard way to represent IPLD data in JSON format. The DAG-JSON specification uses the `/` field as a special namespace for encoding data types outside the JSON data model. We adopt the standard encoding for [bytes] and [links] per the [DAG-JSON] specification and define our own custom data types under the `/` field as described in detail in this document.

This approach:
- Maintains compatibility with standard JSON parsers
- Provides a clear namespace for special types separate from user data
- Enables progressive enhancement where simple JSON becomes linkable
- Supports efficient serialization and deserialization
- Preserves the memory protocol's fact structure

## Reference Types

This protocol describes a set of reference types with different characteristics and use cases:

### Reference Resolution

References are resolved by the [Schema Query Protocol] to reduce roundtrips when following references across facts. The resolution process understands the semantics of each reference type and can efficiently traverse relationships to build comprehensive result sets.

### Inline References

Inline references embed data directly within facts' `is` field using values conforming to the [IPLD data model]. Binary data can be inlined as IPLD [bytes].

#### Bytes in JSON

Inline bytes in JSON encoding follow the [DAG-JSON] specification for representing [bytes]. Memory protocol implementations are RECOMMENDED to impose reasonable size limits for inlined binary data.

##### TypeScript Definition

```ts
type Bytes = {
  '/': {
    /* Base64 encoded binary */
    'bytes': string
  }
}
```

##### Example


```json
// Binary data using DAG-JSON bytes
{
  "/": {
    "bytes": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+P+/HgAFeAJ5gZ5fCQAAAABJRU5ErkJggg=="
  }
}
```

### Immutable References

Immutable references use [merkle reference]s that provide cryptographic integrity guarantees while remaining agnostic of content encoding. They follow [IPLD data model] and are represented as [IPLD Links] with following added constraints:

1. Immutable references MUST be [merkle reference]s.
2. Immutable reference MUST reference content of the fact's `is` field.

> ℹ️ Above constraints attempt to strike a good balance between flexibility and practicality and are optimized for expected usage.

**Key Properties**:
- Reference only the content of the fact's `is` field, not the entire fact structure or subset of the `is` field.
- Use the [merkle reference] codec for content addressing
- Provide cryptographic integrity guarantees
- Never change - the same content always has the same reference and same reference resolve to the same content.


##### TypeScript Definition

```ts
type Reference = {
  /* Multibase Base32 encoded CIDv1 with 0x07 merkle-reference codec */
  "/": string
}
```

##### Example

```json
{
  "/": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5"
}
```
### Mutable References

Mutable references use `link` sigils to reference facts that can be updated over time. These references automatically reflect changes to the addressed data.

**Key Properties**:
- Reference facts by address in memory space using coordinates (`accept`, `source`, `space`, `path`)
- Automatically reflect changes when the addressed data is updated
- Support path navigation within the target fact's `is` field
- Provide configurable write behavior through the `overwrite` field

#### Link Sigil (`link@1`)

The `link` sigil provides mutable references to JSON values held by other facts. If `path` is omitted, it references the whole JSON value - the `is` field of the addressed fact. If `accept` is omitted, it defaults to the type of the fact the link is embedded in. If `source` is omitted, it defaults to the entity the link is embedded in. If both are omitted, it creates a self-reference. The `overwrite` field controls write behavior.

> ℹ️ Therefore `link` with omitted `accept`, `source` and `path` represents a self-reference.

##### Fields

- `source` (optional): Resource URI of the target fact. Defaults to linker's id
- `accept` (optional): Media type preferences for the target fact, following HTTP Accept header semantics. Defaults to linker's type
- `path` (optional): Array of strings/numbers for navigating into the target fact's `is` field.
- `space` (optional): DID of the space containing the target fact. Defaults to current space
- `schema` (optional): JSON Schema for validation
- `overwrite` (optional): Controls write behavior - `"this"` (default) or `"redirect"`

#### TypeScript Definition

```typescript
type LinkSigil = {
  "link@1": {
    // defaults to the entity containing this link
    source?: URI
    // HTTP Accept header format, defaults to the type containing this link
    accept?: string
    // defaults to the space containing this link
    space?: SpaceDID
    // defaults to the empty path [] representing `is` of the addressed fact
    path?: Array<string|number>
    // defaults to "this"
    overwrite?: "this" | "redirect"
    // defaults to any if omitted
    schema?: JSONSchema
  }
}
```

#### Resolution Behavior

Link sigils resolve to the current value at the specified location within the target fact's `is` field. When the target fact changes, all references automatically reflect the new value.

##### Example with Default Values

When `accept`, `source`, or `space` are omitted, they implicitly inherit the `type`, entity, and space of the record this link is contained in. If all are omitted, it creates a self-reference:

```json
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "name": "Alice Smith",
    "displayName": "ali",
    "nickname": {
      "/": {
        "link@1": {
          "path": ["displayName"]
        }
      }
    }
  },
  "cause": "da6lce9tv6fnr7o7wwxez9kjv7qogwmcxjacdrod7y6jer8fn6a7ugmt7"
}
```

In this example, the link creates a self-reference to the same fact (`user:alice` with `application/json`) at the `displayName` path, effectively creating an alias within the same fact.

#### Write Behavior

Setting a property in the referenced data structure keeps the fact containing the link unchanged and forwards changes to the addressed memory space.

##### Example of setting property inside reference

```json
// We start with following set of facts
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "contact": {
      "github": "@alice"
    }
  }
}
{
  "the": "application/json",
  "of": "profile:alice",
  "is": {
    "contact": {
      "/": {
        "link@1": {
          "source": "user:alice",
          "path": ["contact"]
        }
      }
    }
  }
}

// After profile.contact.email = "alice@web.mail", we end up with the first
// fact updated and the second fact remaining unchanged
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "contact": {
      "email": "alice@web.mail",
      "github": "@alice"
    }
  }
}
{
  "the": "application/json",
  "of": "profile:alice",
  "is": {
    "contact": {
      "/": {
        "link@1": {
          "source": "user:alice",
          "path": ["contact"]
        }
      }
    }
  }
}
```

The behavior of setting a property that is a link sigil itself depends on the `overwrite` setting:

- When `overwrite` is `"this"` (default): The property is replaced.
- When `overwrite` is `"redirect"`: The property addressed by the link is updated.


##### Link Write Behavior Examples

The `overwrite` field controls how property assignment behaves:

###### Default Behavior (`overwrite: "this"`)

When assigning a value to a property that is a link with an `overwrite: "this"` (default) setting, the property is **replaced with the value**:

```json
// Before state
{
  "the": "application/json",
  "of": "comment:4737",
  "is": {
    "archived": false,
    "content": "Please add code comment"
  }
}
{
  "the": "application/json",
  "of": "note:bcd9124e",
  "is": {
    "done": {
      "/": {
        "link@1": {
          "accept": "application/json",
          "source": "comment:4737",
          "path": ["archived"]
        }
      }
    }
  }
}

// After setting `note.done = true`
// Addressed fact remains unchanged
{
  "the": "application/json",
  "of": "comment:4737",
  "is": {
    "archived": false,
    "content": "Please add code comment"
  }
}
// Fact containing link sigil is updated
{
  "the": "application/json",
  "of": "note:bcd9124e",
  "is": {
    "done": true  // Link sigil was replaced with literal value
  }
}
```

The addressed `archived` property of the comment (`comment:4737`) remains unchanged.

###### Redirect Behavior (`overwrite: "redirect"`)

When assigning a value to a property that is a link with an `overwrite: "redirect"` setting, the value in the addressed memory space is updated.

```json
// Initial state
{
  "the": "application/json",
  "of": "comment:4737",
  "is": {
    "archived": false,
    "content": "Please add code comment"
  }
}
{
  "the": "application/json",
  "of": "note:bcd9124e",
  "is": {
    "done": {
      "/": {
        "link@1": {
          "accept": "application/json",
          "source": "comment:4737",
          "path": ["archived"],
          ”overwrite”: “redirect”
        }
      }
    }
  }
}

// After note.done = true
// Addressed fact gets updated
{
  "the": "application/json",
  "of": "comment:4737",
  "is": {
    "archived": true,
    "content": "Please add code comment"
  }
}
// Fact containing link sigil remains unchanged
{
  "the": "application/json",
  "of": "note:bcd9124e",
  "is": {
    "done": {
      "/": {
        "link@1": {
          "accept": "application/json",
          "source": "comment:4737",
          "path": ["archived"]
        }
      }
    }
  }
}
```

### Blob References

Blob references are sigils that need to be interpreted as `Blob` instances. They can use inline bytes, immutable references for content, or mutable references (link sigils) for binary data that can be updated.

#### Blob Sigil (`blob@1`)

The blob sigil provides references to binary data that should be interpreted as a `Blob` instance. It can reference either immutable content via immutable references or mutable binary facts via link sigils.

#### Fields

- `type` (optional): Media type of the binary data. If omitted and content is referenced by immutable reference, it will be inferred from content as either `application/json` or `application/octet-stream` depending on the referenced content. If content is referenced via mutable reference (link sigil), content type will be the `type` of resolved data.
- `content` (required): Either a link sigil, an immutable reference or inline bytes.

#### TypeScript Definition

```typescript
type BlobSigil {
  "blob@1": {
    type?: MediaType
    content: LinkSigil | Reference | Bytes
  }
}

type MediaType = `${string}/${string}`
```

#### Resolution Behavior

Blob sigils resolve to `Blob` instances by referencing data through the `content` field. When `content` is a link sigil, the content type is determined by the type of the resolved data. When `content` is an immutable reference, the content type is inferred from the content as either `application/json` or `application/octet-stream` if no `type` field is specified. When `content` is inline bytes, the binary data is embedded directly and the content type defaults to `application/octet-stream` if no `type` field is specified. In all cases, implementations SHOULD resolve to native `Blob` instances.

#### Relationship to Binary Facts

Blob sigils work directly with the memory protocol's binary fact support by referencing facts that contain binary data. This approach enables efficient storage and reuse of binary content across multiple references while maintaining proper media type information.

#### Example with Link Reference

```json
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "name": "Alice Smith",
    "avatar": {
      "/": {
        "blob@1": {
          "type": "image/png",
          "content": {
            "/": {
              "link@1": {
                "accept": "image/png",
                "source": "blob:avatar-alice-2024"
              }
            }
          }
        }
      }
    }
  }
}
```

#### Example with Immutable Reference

```json
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "name": "Alice Smith",
    "avatar": {
      "/": {
        "blob@1": {
          "type": "image/png",
          "content": {
            "/": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5"
          }
        }
      }
    }
  }
}
```

#### Example with Inline Bytes

```json
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "name": "Alice Smith",
    "avatar": {
      "/": {
        "blob@1": {
          "type": "image/png",
          "content": {
            "/": {
              "bytes": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+P+/HgAFeAJ5gZ5fCQAAAABJRU5ErkJggg=="
            }
          }
        }
      }
    }
  }
}
```

### File References

File references are sigils that extend blob references with filesystem metadata and are interpreted as `File` instances.

#### File Sigil (`file@1`)

The file sigil is an extension of the blob sigil that provides references to binary data interpreted as `File` instances. It has the same type inference rules as blob sigils, plus an optional `name` field for filesystem metadata.

#### TypeScript Definition

```typescript
type FileSigil = {
  "file@1": {
    type?: string // Media type, same inference rules as blob sigil
    content?: LinkSigil | Reference | Bytes // optional reference to binary data
    name?: string // optional filename
  }
}
```

### Computational Sigils

Computational sigils use mutable references to create dynamic relationships, computed values, and data transformations based on other facts.

#### Spell Sigil (`spell@1`)

The spell sigil defines reusable TypeScript modules that can be invoked by charm sigils. It contains the function definition including imports, exports, and main function specification.

#### Fields

- `language` (required): Programming language, currently only `"typescript"` is supported
- `imports` (required): Record mapping import names to immutable references containing the module code
- `exports` (required): Object with `"."` property pointing to the entry point module via immutable reference
- `main` (optional): Name of the main export function, defaults to `"default"`

#### TypeScript Definition

```typescript
type SpellSigil = {
  "spell@1": {
    language: "typescript"
    imports: Record<string, Reference>
    exports: { ".": Reference }
    main?: string
  }
}
```

#### Charm Sigil (`charm@1`)

The charm sigil provides function invocation by referencing a spell and specifying input data and output destinations. It enables computed values within facts by executing spells with specified inputs and outputs.

#### Fields

- `spell` (required): Reference to a spell sigil defining the function to execute
- `input` (required): Arguments passed to the main function, can be a record of named inputs or a single JSON value, supporting link sigils for dynamic data
- `output` (required): Destination where the return value is written, can be a record of named outputs or a single destination, supporting link sigils for dynamic targets

#### TypeScript Definition

```typescript
type CharmSigil = {
  "charm@1": {
    spell: Reference | LinkSigil
    input: Record<string, LinkSigil | JSONValue> | JSONValue
    output: Record<string, LinkSigil | JSONValue> | JSONValue
  }
}
```

#### Resolution Behavior

Charms resolve by loading the spell from the referenced spell sigil, importing any specified dependencies from the spell, calling the main function with the provided input, and writing the result to the specified output destination. The TypeScript code executes in a sandboxed environment with access only to the explicitly imported modules from the spell.

#### Spell Example

```json
{
  "the": "application/json",
  "of": "spell:sales-calculator",
  "is": {
    "/": {
      "spell@1": {
        "language": "typescript",
        "imports": {
          "lodash": { "/": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5" }
        },
        "exports": {
          ".": { "/": "ca5kbd8su5emr6n6vvwdz8jiu6pnfvlbwizccqnd6x5idq7em5z6tfls6" }
        },
        "main": "calculateTotal"
      }
    }
  },
  "cause": "ea7mdf0uw7gos8p8xxydz0klw8rpgxndykaedspd8z7kfs9go7b8vhmsu8"
}
```

#### Charm Example

```json
{
  "the": "application/json",
  "of": "report:monthly",
  "is": {
    "totalSales": {
      "/": {
        "charm@1": {
          "spell": { "/": "ea7mdf0uw7gos8p8xxydz0klw8rpgxndykaedspd8z7kfs9go7b8vhmsu8" },
          "input": {
            "salesData": {
              "/": {
                "link@1": {
                  "accept": "application/json",
                  "source": "sales:12-06-2023"
                }
              }
            }
          },
          "output": {
            "/": {
              "link@1": {
                "accept": "application/json",
                "source": "report:monthly",
                "path": ["totalAmount"]
              }
            }
          }
        }
      }
    }
  },
  "cause": "da6lce9tv6fnr7o7wwxez9kjv7qogwmcxjacdrod7y6jer8fn6a7ugmt7"
}
```

#### Backlink Sigil (`backlink@1`)

The backlink sigil provides dynamic collections of facts that reference a specific target fact. It enables reverse relationship queries by finding all facts that contain references to the target at a specified path, with optional filtering conditions.

#### Fields

- `target` (required): Fact selector for the entity being referenced
- `path` (optional): Path within referencing facts to check
- `conditions` (optional): Additional filtering conditions
- `space` (optional): Space to search in

#### TypeScript Definition

```typescript
type BacklinkSigil = {
  "backlink@1": {
    target: FactCoordinate
    path?: JSONPath
    conditions?: Record<string, unknown>
    space?: SpaceDID
  }
}
```

#### Resolution Behavior

Backlinks resolve to an array of facts that contain references to the target fact at the specified path.

#### Example

```json
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "followers": {
      "/": {
        "backlink@1": {
          "target": {
            "accept": "application/json",
            "source": "user:alice"
          },
          "path": ["following"],
          "conditions": {
            "active": true
          }
        }
      }
    }
  },
  "cause": "ea7mdf0uw7gos8p8xxydz0klw8rpgxndykaedspd8z7kfs9go7b8vhmsu8"
}
```

#### Merge Sigil (`merge@1`)

The merge sigil provides composition of JSON values by merging data from multiple sources. For objects, sources are merged with later sources taking precedence over earlier ones (key ordering matters). For arrays, sources are concatenated in order. This enables flexible fact composition and data inheritance.

#### Fields

- `sources` (required): Array of values to merge. Can contain literal values, embed sigils, or any other sigils
- `space` (optional): Default space DID for any embed sigils that don't specify their own

#### TypeScript Definition

```typescript
type MergeSigil = {
  "merge@1": {
    sources: unknown[]
    space?: SpaceDID
  }
}
```

#### Resolution Behavior

For objects: Merges all sources in order, with later sources taking precedence over earlier ones when keys conflict.
For arrays: Concatenates all sources in order.
Sigils in the sources array are resolved first, then the resolved values are merged/concatenated with literal values.

#### Example

```json
{
  "the": "application/json",
  "of": "config:production",
  "is": {
    "settings": {
      "/": {
        "merge@1": {
          "sources": [
            {
              "/": {
                "embed@1": {
                  "accept": "application/json",
                  "source": "config:base"
                }
              }
            },
            {
              "/": {
                "embed@1": {
                  "accept": "application/json",
                  "source": "config:environment"
                }
              }
            },
            {
              "debug": false,
              "apiUrl": "https://api.production.com"
            }
          ]
        }
      }
    }
  },
  "cause": "fa8neh1vx8hpt9q9yyzer1lmx9sqhyoezlbfespezz8lgtsafp8cwiosu9"
}
```

## Integration with Memory Protocol

### Fact Structure Preservation

Sigils operate entirely within the `is` field of facts, preserving the memory protocol's core structure:

- `the` field remains the media type of the containing fact
- `of` field remains the resource URI of the containing fact
- `cause` field maintains causal consistency as defined by the memory protocol
- Sigils reference other facts but from the memory protocol's perspective are just data structures in the `is` field.

### Query Integration

Sigils work with both the basic `/memory/query` and advanced `/memory/graph/query` capabilities:

#### Basic Query Support (`/memory/query`)

Basic queries return sigils in their raw form without following references.

#### Schema Query Support (`/memory/graph/query`)

The [Schema Query Protocol] can follow sigil references and return bundled results to reduce roundtrips.

## Migration and Versioning

### Version Evolution

Sigil types include version suffixes (e.g., `link@1`) to enable backward-compatible evolution:

1. **New versions**: Can add fields or change behavior while maintaining old version support
2. **Deprecation**: Old versions should be supported during transition periods
3. **Migration tools**: Implementations SHOULD provide utilities for upgrading sigil versions


[Common Tools]: https://common.tools/
[Irakli Gozalishvili]: https://github.com/gozala
[Memory Protocol]: ./memory.md
[Binary Data Support]: ./memory-blobs.md
[Schema Query Protocol]: ./schema-query.md
[DAG-JSON]: https://ipld.io/specs/codecs/dag-json/spec/
[bytes]: https://ipld.io/specs/codecs/dag-json/spec/#bytes
[links]: https://ipld.io/specs/codecs/dag-json/spec/#links
[merkle reference]: https://github.com/Gozala/merkle-reference
[Blob]: https://developer.mozilla.org/en-US/docs/Web/API/Blob
[Accept header]:developer.mozilla.org/en-us/docs/web/http/reference/headers/accept
[IPLD data model]:https://ipld.io/glossary/#data-model
