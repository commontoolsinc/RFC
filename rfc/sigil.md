# Sigil Protocol: Cross-Fact References for Memory Protocol

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [Irakli Gozalishvili], [Common Tools]

## Authors

- [Irakli Gozalishvili], [Common Tools]

## Abstract

This RFC extends the [Memory Protocol] with a standardized system for describing references across facts boundaries in user-controlled spaces. The sigil protocol adopts the [DAG-JSON] convention, using the `/` field to encode custom data types.

Sigils provide a unified mechanism for expressing mutable redirects, transparent aliases, binary content references, computed data sources, and relational queries within facts, enabling sophisticated data modeling while preserving the causal integrity of the memory protocol.

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

The sigil protocol addresses these needs by providing a type-safe, extensible system for encoding various kinds of references directly within the memory protocol's JSON fact data model.

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
    "<sigil-type>": {
      // sigil-specific fields
    }
  }
}
```

The `<sigil-type>` identifies the specific kind of reference, determining both the structure of the contained metadata and the resolution behavior.

## Core Sigil Types

### Link Sigil (`link-v1`)

The link sigil provides basic cross-fact references with optional path navigation within the target fact's data.

#### Structure

```json
{
  "/": {
    "link-v1": {
      "the": "<media-type>",
      "of": "<resource-uri>",
      "path": ["field", "nested", "0"],
      "space": "<space-did>"
    }
  }
}
```

#### Fields

- `the` (required): Media type of the target fact
- `of` (required): Resource URI of the target fact
- `path` (optional): Array of strings/numbers for navigating into the target fact's `is` field
- `space` (optional): DID of the space containing the target fact. Defaults to current space

#### Resolution Behavior

Link sigils resolve to the current value at the specified location within the target fact's `is` field. When the target fact changes, all references automatically reflect the new value.

#### Example

```json
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "name": "Alice Smith",
    "manager": {
      "/": {
        "link-v1": {
          "the": "application/json",
          "of": "user:bob",
          "path": ["name"]
        }
      }
    }
  },
  "cause": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5"
}
```

### Alias Sigil (`alias-v1`)

Aliases create transparent references that behave like queries for reads but allow writes to pass through to the underlying target fact.

#### Structure

```json
{
  "/": {
    "alias-v1": {
      "the": "<media-type>",
      "of": "<resource-uri>",
      "path": ["field", "nested"],
      "space": "<space-did>",
      "schema": { /* JSON Schema */ }
    }
  }
}
```

#### Fields

- `the` (required): Media type of the target fact
- `of` (required): Resource URI of the target fact
- `path` (optional): Path into the target fact's `is` field
- `space` (optional): Target space DID
- `schema` (optional): JSON Schema for validation

#### Resolution Behavior

- **Reads**: Return the current value from the target fact's location
- **Writes**: Modify the target fact directly, making the alias transparent to mutations while maintaining causal consistency
- **Validation**: If schema is provided, validate writes against it before applying

#### Example

```json
{
  "the": "application/json",
  "of": "dashboard:main",
  "is": {
    "currentUser": {
      "/": {
        "alias-v1": {
          "the": "application/json",
          "of": "session:current",
          "path": ["user"],
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

### Blob Sigil (`blob-v1`)

Blob sigils reference binary data stored as facts with media type information, extending the [Binary Data Support] for the memory protocol.

#### Structure

```json
{
  "/": {
    "blob-v1": {
      "type": "image/png",
      "content": { "/": { "bytes": "iVBORw0KGgo..." } }
    }
  }
}
```

#### Fields

- `type` (optional): Media type of the binary data. Defaults to `application/octet-stream`
- `content` (required): Binary data in [DAG-JSON bytes] format

#### Resolution Behavior

Blob sigils resolve to the binary content with associated media type metadata. In environments that support it, they MAY resolve to native blob objects.

#### Relationship to Binary Facts

Blob sigils complement the memory protocol's binary fact support by providing inline binary references, while binary facts store large binary content as separate facts with proper media types.

### File Sigil (`file-v1`)

File sigils extend blob sigils with filesystem metadata, useful for representing files within facts.

#### Structure

```json
{
  "/": {
    "file-v1": {
      "type": "image/png",
      "name": "profile-photo.png",
      "lastModified": 1697409438000,
      "content": { "/": { "bytes": "iVBORw0KGgo..." } }
    }
  }
}
```

#### Fields

- `type` (optional): Media type. Defaults to `application/octet-stream`
- `name` (required): Filename
- `lastModified` (optional): Unix timestamp of last modification
- `content` (required): Binary data in [DAG-JSON bytes] format

### Charm Sigil (`charm-v1`)

Charms reference computational recipes that describe how data is derived from other facts, enabling computed facts within the memory protocol.

#### Structure

```json
{
  "/": {
    "charm-v1": {
      "spell": "<spell-uri>",
      "args": { /* spell arguments */ },
      "sources": [
        {"the": "application/json", "of": "sales:*"},
        {"the": "application/json", "of": "config:rates"}
      ],
      "space": "<space-did>"
    }
  }
}
```

#### Fields

- `spell` (required): URI identifying the computational recipe
- `args` (optional): Arguments passed to the spell
- `sources` (optional): Array of fact selectors indicating source facts used in computation
- `space` (optional): Space containing the source facts

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
        "charm-v1": {
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

### Backlink Sigil (`backlink-v1`)

Backlinks create dynamic collections of facts that reference a specific target, enabling reverse relationship queries.

#### Structure

```json
{
  "/": {
    "backlink-v1": {
      "target": {
        "the": "<target-media-type>",
        "of": "<target-resource-uri>"
      },
      "path": ["field", "reference"],
      "conditions": { /* additional filters */ },
      "space": "<space-did>"
    }
  }
}
```

#### Fields

- `target` (required): Fact selector for the entity being referenced
- `path` (optional): Path within referencing facts to check
- `conditions` (optional): Additional filtering conditions
- `space` (optional): Space to search in

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
        "backlink-v1": {
          "target": {
            "the": "application/json",
            "of": "user:alice"
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

### Spread Sigil (`spread-v1`)

Spread sigils simulate JavaScript-like spread operators for objects and arrays, enabling composition of facts.

#### Structure

```json
{
  "/": {
    "spread-v1": {
      "source": {
        "the": "<source-media-type>",
        "of": "<source-resource-uri>"
      },
      "path": ["field"],
      "additions": { /* additional fields */ },
      "space": "<space-did>"
    }
  }
}
```

#### Fields

- `source` (required): Fact selector for the source data
- `path` (optional): Path to the source data within the fact's `is` field
- `additions` (optional): Additional fields to merge
- `space` (optional): Source space DID

#### Resolution Behavior

For objects: Merges source fact data with additions, with additions taking precedence.
For arrays: Concatenates source fact data with additional items.

#### Example

```json
{
  "the": "application/json",
  "of": "config:production",
  "is": {
    "settings": {
      "/": {
        "spread-v1": {
          "source": {
            "the": "application/json",
            "of": "config:base"
          },
          "additions": {
            "debug": false,
            "apiUrl": "https://api.production.com"
          }
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
    "alias-v1": {
      "the": "application/json",
      "of": {
        "/": {
          "link-v1": {
            "the": "application/json",
            "of": "config:current",
            "path": ["primaryDatabase"]
          }
        }
      },
      "path": ["connectionString"]
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
        "charm-v1": {
          "spell": "dateRange",
          "args": {
            "period": {
              "/": {
                "link-v1": {
                  "the": "application/json",
                  "of": "settings:dashboard",
                  "path": ["selectedPeriod"]
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

Sigils work with the memory protocol's query system:

1. **Sigil expansion**: Queries MAY expand sigils during resolution
2. **Selective resolution**: Queries MAY choose to return sigils unexpanded for efficiency
3. **Dependency tracking**: Query results can include sigil dependency information

## Implementation Requirements

### Sigil Resolution

Implementations MUST:

1. **Parse sigils**: Recognize the `/` field pattern within fact `is` fields
2. **Type validation**: Validate sigil structure according to type-specific schemas
3. **Fact resolution**: Follow sigil references to retrieve target fact data
4. **Error handling**: Provide clear errors for broken references or invalid sigils
5. **Caching**: Implement efficient caching strategies for resolved sigil values

### Write Transparency for Aliases

For alias sigils, implementations MUST:

1. **Detect writes**: Identify when alias targets are being modified
2. **Pass through**: Route write operations to the underlying target fact
3. **Validate**: Apply schema validation if specified
4. **Maintain causality**: Ensure writes maintain proper fact causal chains
5. **Transaction integrity**: Process alias writes within proper memory protocol transactions

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

Sigil types include version suffixes (e.g., `link-v1`) to enable backward-compatible evolution:

1. **New versions**: Can add fields or change behavior while maintaining old version support
2. **Deprecation**: Old versions should be supported during transition periods
3. **Migration tools**: Implementations SHOULD provide utilities for upgrading sigil versions

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
        "link-v1": {
          "the": "application/json",
          "of": "user:current",
          "path": ["name"]
        }
      }
    },
    "logo": {
      "/": {
        "file-v1": {
          "name": "company-logo.png",
          "type": "image/png",
          "content": { "/": { "bytes": "iVBORw0KGgo..." } }
        }
      }
    },
    "metrics": {
      "/": {
        "charm-v1": {
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
        "spread-v1": {
          "source": {
            "the": "application/json",
            "of": "config:default"
          },
          "additions": {
            "customizations": true
          }
        }
      }
    },
    "relatedDocuments": {
      "/": {
        "backlink-v1": {
          "target": {
            "the": "application/json",
            "of": "document:project-status"
          },
          "path": ["references"]
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
                "link-v1": {
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
                "blob-v1": {
                  "type": "image/jpeg",
                  "content": { "/": { "bytes": "/9j/4AAQSkZJRgABAQEASABI..." } }
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
[DAG-JSON]: https://ipld.io/specs/codecs/dag-json/spec/
[DAG-JSON bytes]: https://ipld.io/specs/codecs/dag-json/spec/#bytes
