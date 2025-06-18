# Sigil Protocol: Cross-Fact References for Memory Protocol

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [Irakli Gozalishvili], [Common Tools]

## Authors

- [Bernhard Seefeld], [Common Tools]

## Abstract

This RFC extends the [Memory Protocol] with a standardized system for describing references across facts boundaries in user-controlled spaces. The sigil protocol adopts the [DAG-JSON] convention, using the `/` field to encode custom data types.

Sigils provide a unified mechanism for expressing mutable redirects, transparent cursors, binary content references, computed data sources, and relational queries within facts, enabling sophisticated data modeling while preserving the causal integrity of the memory protocol. Sigils are resolved by the [Schema Query Protocol], which provides the `/memory/graph/query` capability for advanced querying with automatic sigil resolution.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## Motivation

The [Memory Protocol] provides a robust foundation for storing and querying facts in user-controlled spaces.
Without a standardized cross-fact reference mechanism, applications must implement ad-hoc solutions for:

- Referencing data stored in other facts within the same or different resources
- Creating mutable redirects that can be updated without breaking references
- Establishing computed relationships between facts
- Handling binary data and file metadata consistently within facts
- Building reactive systems that respond to changes in referenced facts

The sigil protocol addresses these needs by providing a type-safe, extensible system for encoding various kinds of references directly within the memory protocol's JSON fact data model. Sigils are resolved through the [Schema Query Protocol], enabling applications to work with computed values and relationships without implementing complex resolution logic.

## Sigil Foundation

### DAG-JSON Convention

Sigils adopt the [DAG-JSON] linking convention, where the `/` field contains metadata about the reference type and target fact. This approach:

- Maintains compatibility with standard JSON parsers
- Provides a clear namespace for link metadata separate from user data
- Enables progressive enhancement where simple JSON facts become linkable
- Supports efficient serialization and deserialization
- Preserves the memory protocol's fact structure

### Basic Sigil Structure

All sigils follow this general pattern within a fact's `is` field:

```json
{
  "/": {
    "<sigil-name>@<version>": {
      // sigil-specific fields
    }
  }
}
```

The sigil type identifier consists of two components:
- **Sigil name**: Identifies the kind of reference (e.g., `query`, `cursor`, `blob`)
- **Version**: Follows semver convention with omitted components (e.g., `@1`, `@1.2`, `@2.0.1`)

This naming convention allows implementations to support multiple versions of the same sigil type and provides a clear upgrade path as the protocol evolves. Version components default to `0` when omitted (e.g., `@1` equals `@1.0.0`).

## Sigil Resolution

Sigils are resolved by the [Schema Query Protocol] to reduce roundtrips when following references across facts.



## Core Sigil Types

### Embed Sigil (`embed@1`)

The embed sigil provides references to a JSON value held by another fact at a given path. If path is omitted it references whole JSON value - `is` field of the target fact. When an embed sigil is overwritten, the sigil itself is replaced, not the target fact.

#### Fields

- `the` (required): Media type of the target fact
- `of` (required): Resource URI of the target fact
- `at` (optional): Array of strings/numbers for navigating into the target fact's `is` field
- `from` (optional): DID of the space containing the target fact. Defaults to current space

#### TypeScript Definition

```typescript
type EmbedSigil = {
  "embed@1": {
    the: MIME
    of: URI
    at?: JSONPath
    from?: SpaceDID
  }
}
```

#### Resolution Behavior

Embed sigils resolve to the current value at the specified location within the target fact's `is` field. When the target fact changes, all references automatically reflect the new value.

**Write Behavior**: When an embed sigil is assigned a new value, the sigil itself is overwritten and replaced with the new value. The target fact remains unchanged.

#### Example

```json
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "name": "Alice Smith",
    "manager": {
      "/": {
        "embed@1": {
          "the": "application/json",
          "of": "user:bob",
          "at": ["name"]
        }
      }
    }
  },
  "cause": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5"
}
```

### Mount Sigil (`mount@1`)

The mount sigil provides mutable references to a JSON value held by another fact at a given path, acting like a permanent redirect. Unlike embeds, mounts are followed first before writing - when you write to a mount, the write operation is redirected to the underlying target fact, making mounts transparent to mutations. If `at` is omitted it references the whole JSON value - `is` field of the target fact. If `the` and `of` are omitted, they default to the containing fact's `the` and `of` values.

#### Fields

- `the` (optional): Media type of the target fact. Defaults to containing fact's `the` value
- `of` (optional): Resource URI of the target fact. Defaults to containing fact's `of` value
- `at` (optional): Path into the target fact's `is` field
- `from` (optional): Target space DID
- `schema` (optional): JSON Schema for validation

#### TypeScript Definition

```typescript
type MountSigil = {
  "mount@1": {
    the?: MIME    // defaults to containing fact's `the` value
    of?: URI      // defaults to containing fact's `of` value
    at?: JSONPath
    from?: SpaceDID
    schema?: JSONSchema7
  }
}
```

#### Resolution Behavior

- **Reads**: Return the current value from the target fact's location
- **Writes**: Follow the cursor to the target fact first, then modify the target fact directly, making the cursor transparent to mutations while maintaining causal consistency. This is like a permanent redirect for write operations.
- **Validation**: If schema is provided, validate writes against it before applying to the target fact

#### Example

```json
{
  "the": "application/json",
  "of": "dashboard:main",
  "is": {
    "currentUser": {
      "/": {
        "mount@1": {
          "the": "application/json",
          "of": "session:current",
          "at": ["user"],
          "schema": {
            "type": "object",
            "properties": {
              "name": { "type": "string" }
            }
          }
        }
      }
    }
  },
  "cause": "ca5kbd8su5emr6n6vvwdz8jiu6pnfvlbwizccqnd6x5idq7em5z6tfls6"
}
```

#### Example with Default Values

When `the` and `of` are omitted, they default to the containing fact's values:

```json
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "name": "Alice Smith",
    "nickname": {
      "/": {
        "mount@1": {
          "at": ["displayName"]
        }
      }
    }
  },
  "cause": "da6lce9tv6fnr7o7wwxez9kjv7qogwmcxjacdrod7y6jer8fn6a7ugmt7"
}
```

In this example, the mount references the same fact (`user:alice` with `application/json`) at the `displayName` path, effectively creating an alias within the same fact.

## Embed vs Mount: Write Behavior Comparison

The key difference between Embed and Mount sigils lies in their write behavior:

### Embed Sigil Write Behavior

When you write to an Embed sigil, you **replace the sigil itself** with the new value:

```json
// Initial state
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "manager": {
      "/": {
        "embed@1": {
          "the": "application/json",
          "of": "user:bob",
          "at": ["name"]
        }
      }
    }
  }
}

// After writing "Charlie" to manager field
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "manager": "Charlie"  // Embed sigil replaced with literal value
  }
}
```

The target fact (`user:bob`) remains unchanged.

### Mount Sigil Write Behavior

When you write to a Mount sigil, the write **follows the mount** to modify the target fact:

```json
// Initial state
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "manager": {
      "/": {
        "mount@1": {
          "the": "application/json",
          "of": "user:bob",
          "at": ["name"]
        }
      }
    }
  }
}

// After writing "Charlie" to manager field
// The mount remains, but user:bob fact is modified
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "manager": {
      "/": {
        "mount@1": {
          "the": "application/json",
          "of": "user:bob",
          "at": ["name"]
        }
      }
    }
  }
}

// And user:bob fact is updated:
{
  "the": "application/json",
  "of": "user:bob",
  "is": {
    "name": "Charlie"  // Target fact modified
  }
}
```

### Use Cases

- **Embed sigils**: Use when you want a mutable reference that can be replaced with direct values
- **Mount sigils**: Use when you want transparent redirection, like permanent redirects or symbolic links

### Blob Sigil (`blob@1`)

The blob sigil provides references to binary data stored as another fact with a corresponding media type information. It enables referencing binary content stored as separate facts, extending the [Binary Data Support] for the memory protocol.

#### Fields

- `the` (optional): Media type of the binary data. Defaults to `application/octet-stream`
- `of` (required): Entity identifier for the blob fact

#### TypeScript Definition

```typescript
type BlobSigil = {
  "blob@1": {
    the?: MIME // defaults to application/octet-stream
    of: URI
  }
}
```

#### Resolution Behavior

Blob sigils resolve by referencing binary data stored in another fact identified by the `of` field. The `the` field specifies the expected media type of the target fact. In environments that support it, they SHOULD resolve to native `Blob` instances.

#### Relationship to Binary Facts

Blob sigils work directly with the memory protocol's binary fact support by referencing facts that contain binary data. This approach enables efficient storage and reuse of binary content across multiple references while maintaining proper media type information.

#### Example

```json
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "name": "Alice Smith",
    "avatar": {
      "/": {
        "blob@1": {
          "the": "image/png",
          "of": "attachment:avatar-alice-2024"
        }
      }
    }
  },
  "cause": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5"
}
```

### File Sigil (`file@1`)

The file sigil provides references to binary data with filesystem metadata including filename and modification time. It extends the blob sigil concept with additional file-specific properties, useful for representing files within facts.

#### Fields

- `the` (optional): Media type. Defaults to `application/octet-stream`
- `of` (optional): Entity identifier for the file/blob fact
- `name` (required): Filename

#### TypeScript Definition

```typescript
type FileSigil = {
  "file@1": {
    the?: MIME    // defaults to application/octet-stream
    of?: URI      // Entity identifier for the file / blob
    name: string  // File name
  }
}
```

#### Resolution Behavior

File sigils resolve by referencing binary data stored in another fact (when `of` is specified) with filesystem metadata. The `name` field provides the filename property. In environments that support it, they SHOULD resolve to native `File` instances that extend `Blob` with additional file metadata.

#### Example

```json
{
  "the": "application/json",
  "of": "document:report-2024",
  "is": {
    "title": "Annual Report",
    "attachment": {
      "/": {
        "file@1": {
          "the": "application/pdf",
          "of": "blob:report-pdf-2024",
          "name": "annual-report-2024.pdf"
        }
      }
    }
  },
  "cause": "ca5kbd8su5emr6n6vvwdz8jiu6pnfvlbwizccqnd6x5idq7em5z6tfls6"
}
```

### Charm Sigil (`charm@1`)

The charm sigil provides references to computational recipes (spells) that describe how data is derived from other facts. It enables computed values within facts by executing spells with specified arguments and source fact dependencies.

#### Fields

- `spell` (required): URI identifying the computational recipe
- `args` (optional): Arguments passed to the spell
- `sources` (optional): Array of fact selectors indicating source facts used in computation
- `from` (optional): Space containing the source facts

#### TypeScript Definition

```typescript
type CharmSigil = {
  "charm@1": {
    spell: URI
    args?: Record<string, unknown>
    sources?: FactCoordinate[]
    from?: SpaceDID
  }
}

type FactCoordinate = {
  the: MIME
  of: URI
  from?: SpaceDID
}
```

#### Resolution Behavior

Charms resolve by executing the referenced spell with the provided arguments and source facts. The spell defines how to compute the result, potentially incorporating data from multiple facts.

#### Example

```json
{
  "the": "application/json",
  "of": "report:monthly",
  "is": {
    "totalSales": {
      "/": {
        "charm@1": {
          "spell": "sum",
          "args": {
            "field": "amount"
          },
          "sources": [
            {"the": "application/json", "of": "sales:*"}
          ]
        }
      }
    }
  },
  "cause": "da6lce9tv6fnr7o7wwxez9kjv7qogwmcxjacdrod7y6jer8fn6a7ugmt7"
}
```

### Backlink Sigil (`backlink@1`)

The backlink sigil provides dynamic collections of facts that reference a specific target fact. It enables reverse relationship queries by finding all facts that contain references to the target at a specified path, with optional filtering conditions.

#### Fields

- `target` (required): Fact selector for the entity being referenced
- `at` (optional): Path within referencing facts to check
- `conditions` (optional): Additional filtering conditions
- `from` (optional): Space to search in

#### TypeScript Definition

```typescript
type BacklinkSigil = {
  "backlink@1": {
    target: FactCoordinate
    at?: JSONPath
    conditions?: Record<string, unknown>
    from?: SpaceDID
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
            "the": "application/json",
            "of": "user:alice"
          },
          "at": ["following"],
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

### Merge Sigil (`merge@1`)

The merge sigil provides composition of JSON values by merging data from multiple sources. For objects, sources are merged with later sources taking precedence over earlier ones (key ordering matters). For arrays, sources are concatenated in order. This enables flexible fact composition and data inheritance.

#### Fields

- `sources` (required): Array of values to merge. Can contain literal values, embed sigils, or any other sigils
- `from` (optional): Default space DID for any embed sigils that don't specify their own

#### TypeScript Definition

```typescript
type MergeSigil = {
  "merge@1": {
    sources: unknown[]
    from?: SpaceDID
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
                  "the": "application/json",
                  "of": "config:base"
                }
              }
            },
            {
              "/": {
                "embed@1": {
                  "the": "application/json",
                  "of": "config:environment"
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

## Advanced Features

### Nested Sigils

Sigils can be nested within other sigils to create complex reference patterns:

```json
{
  "/": {
    "mount@1": {
      "the": "application/json",
      "of": {
        "/": {
          "embed@1": {
            "the": "application/json",
            "of": "config:current",
            "at": ["primaryDatabase"]
          }
        }
      },
      "at": ["connectionString"]
    }
  }
}
```

### Sigil Composition with Facts

Multiple sigils can work together within and across facts to create sophisticated data relationships:

```json
{
  "the": "application/json",
  "of": "dashboard:sales",
  "is": {
    "currentPeriod": {
      "/": {
        "charm@1": {
          "spell": "dateRange",
          "args": {
            "period": {
              "/": {
                "embed@1": {
                  "the": "application/json",
                  "of": "settings:dashboard",
                  "at": ["selectedPeriod"]
                }
              }
            }
          }
        }
      }
    }
  },
  "cause": "ga9ofh2wy9iqt0r0zza1s2mnz0trhzpfamcgftqfaz9mhutbqq9dxjptv0"
}
```

## Integration with Memory Protocol

### Fact Structure Preservation

Sigils operate entirely within the `is` field of facts, preserving the memory protocol's core structure:

- `the` field remains the media type of the containing fact
- `of` field remains the resource URI of the containing fact
- `cause` field maintains causal consistency as defined by the memory protocol
- Sigils reference other facts using their `{the, of}` coordinates

### Transaction Support

Sigils integrate seamlessly with the memory protocol's transaction system:

1. **Atomic updates**: Changes to facts containing sigils are processed atomically
2. **Causal consistency**: Sigil resolution respects the causal ordering of facts
3. **Cross-fact dependencies**: Updates to referenced facts automatically invalidate dependent sigil resolutions

### Query Integration

Sigils work with both the basic `/memory/query` and advanced `/memory/graph/query` capabilities:

#### Basic Query Support (`/memory/query`)

Basic queries return sigils in their raw form without following references.

#### Schema Query Support (`/memory/graph/query`)

The [Schema Query Protocol] can follow sigil references and return bundled results to reduce roundtrips.

## Implementation Requirements

### Sigil Resolution

Implementations MUST:

1. **Parse sigils**: Recognize the `/` field pattern within fact `is` fields
2. **Type validation**: Validate sigil structure according to type-specific schemas
3. **Fact resolution**: Follow sigil references to retrieve target fact data
4. **Error handling**: Provide clear errors for broken references or invalid sigils
5. **Caching**: Implement efficient caching strategies for resolved sigil values

### Write Handling

Implementations MUST handle writes differently for Embed and Mount sigils:

#### Embed Sigil Writes

For embed sigils, implementations MUST:

1. **Replace sigil**: When an embed sigil is written to, replace the entire sigil with the new value
2. **Preserve target**: The target fact referenced by the embed remains unchanged
3. **Local mutation**: The write affects only the containing fact, not the referenced fact

#### Mount Sigil Writes

For mount sigils, implementations MUST:

1. **Follow mount**: When a mount sigil is written to, follow the mount to the target fact first
2. **Route writes**: Redirect write operations to the underlying target fact at the specified path
3. **Preserve mount**: The mount sigil itself remains unchanged in the containing fact
4. **Validate**: Apply schema validation if specified before writing to target
5. **Maintain causality**: Ensure writes maintain proper fact causal chains for the target fact
6. **Transaction integrity**: Process mount writes within proper memory protocol transactions

### Reactive Updates

Implementations SHOULD:

1. **Track dependencies**: Monitor changes to facts referenced by sigils
2. **Propagate updates**: Notify dependent facts when targets change
3. **Batch notifications**: Efficiently handle cascading updates across fact dependencies
4. **Prevent cycles**: Detect and handle circular references between facts

## Security Considerations

### Cross-Space Fact References

When sigils reference facts across different spaces:

1. **Authorization**: Verify read permissions for cross-space fact access
2. **Privacy**: Respect space access controls and fact visibility
3. **Integrity**: Validate that cross-space fact references are authentic

### Computational Security for Charms

For charm sigils:

1. **Sandboxing**: Execute spells in secure, isolated environments
2. **Resource limits**: Prevent denial-of-service through resource exhaustion
3. **Code verification**: Ensure spell code integrity and authenticity
4. **Fact access control**: Restrict spell access to authorized source facts

## Compatibility

This specification is designed to be compatible with:

- The core [Memory Protocol]
- The [Binary Data Support] extension
- Standard JSON parsing libraries
- [DAG-JSON] implementations

Implementations MAY choose to support subsets of sigil types based on their specific requirements. Facts containing unsupported sigils SHOULD still be storable and queryable, with sigils remaining unresolved.

## Migration and Versioning

### Version Evolution

Sigil types include version suffixes (e.g., `embed@1`) to enable backward-compatible evolution:

1. **New versions**: Can add fields or change behavior while maintaining old version support
2. **Deprecation**: Old versions should be supported during transition periods
3. **Migration tools**: Implementations SHOULD provide utilities for upgrading sigil versions

#### TypeScript Version Support

Implementations can use TypeScript's union types to support multiple versions:

```typescript
// Support for multiple versions of the same sigil
type EmbedSigilAny = EmbedSigil1 | EmbedSigil2 // when @2 is introduced

type EmbedSigil1 = {
  "embed@1": {
    the: MIME
    of: URI
    at?: JSONPath
    from?: SpaceDID
  }
}

// Future version example
type EmbedSigil2 = {
  "embed@2": {
    the: MIME
    of: URI
    at?: JSONPath
    from?: SpaceDID
    // hypothetical new fields
    cache?: boolean
    timeout?: number
  }
}

// Version-aware sigil processing
function processEmbedSigil(sigil: EmbedSigilAny): unknown {
  if ("embed@1" in sigil) {
    return processEmbed1(sigil["embed@1"])
  } else if ("embed@2" in sigil) {
    return processEmbed2(sigil["embed@2"])
  }
  throw new Error("Unsupported embed sigil version")
}
```

### Legacy Fact Support

Existing facts without sigils remain fully compatible. Sigils are additive and don't break existing fact functionality.

## Examples

### Complete Fact with Multiple Sigils

```json
{
  "the": "application/json",
  "of": "document:project-status",
  "is": {
    "title": "Project Status Report",
    "author": {
      "/": {
        "embed@1": {
          "the": "application/json",
          "of": "user:current",
          "at": ["name"]
        }
      }
    },
    "logo": {
      "/": {
        "file@1": {
          "the": "image/png",
          "of": "blob:company-logo-2024",
          "name": "company-logo.png"
        }
      }
    },
    "metrics": {
      "/": {
        "charm@1": {
          "spell": "aggregateMetrics",
          "sources": [
            {"the": "application/json", "of": "metrics:*"}
          ],
          "args": {
            "period": "last30days"
          }
        }
      }
    },
    "teamSettings": {
      "/": {
        "merge@1": {
          "sources": [
            {
              "/": {
                "embed@1": {
                  "the": "application/json",
                  "of": "config:default"
                }
              }
            },
            {
              "customizations": true
            }
          ]
        }
      }
    },
    "relatedDocuments": {
      "/": {
        "backlink@1": {
          "target": {
            "the": "application/json",
            "of": "document:project-status"
          },
          "at": ["references"]
        }
      }
    }
  },
  "cause": "ha0pgi3wz0jru1s1aa0bt3noxa1tsizqbmdhgurhbaankuvcra9eykvtw1"
}
```

### Transaction with Sigil Facts

```json
{
  "changes": {
    "user:alice": {
      "application/json": {
        "ia1qhi4xa1ksu2t2bb1cu4opzb2utjakdneikvsicc0olvwdsb0fzlwux2": {
          "is": {
            "name": "Alice Smith",
            "profile": {
              "/": {
                "embed@1": {
                  "the": "application/json",
                  "of": "profile:alice-public"
                }
              }
            }
          }
        }
      }
    },
    "profile:aliice-public": {
      "application/json": {
        "ja2rij5yb2ltv3u3cc2dv5pqac3vukeleofjlwtjdd1pmvxetc1gazwuy3": {
          "is": {
            "bio": "Software Engineer",
            "avatar": {
              "/": {
                "blob@1": {
                  "the": "image/jpeg",
                  "of": "blob:avatar-alice-profile"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

[Common Tools]: https://common.tools/
[Irakli Gozalishvili]: https://github.com/gozala
[Memory Protocol]: ./memory.md
[Binary Data Support]: ./memory-blobs.md
[Schema Query Protocol]: ./schema-query.md
[DAG-JSON]: https://ipld.io/specs/codecs/dag-json/spec/
[DAG-JSON bytes]: https://ipld.io/specs/codecs/dag-json/spec/#bytes
[Blob]: https://developer.mozilla.org/en-US/docs/Web/API/Blob
