# Sigil Protocol: Cross-Fact References for Memory Protocol

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [Irakli Gozalishvili], [Common Tools]

## Authors

- [Bernhard Seefeld], [Common Tools]

## Abstract

This RFC extends the [Memory Protocol] with a standardized system for creating references between facts in user-controlled spaces. The protocol defines two fundamental types of references:

1. **Immutable references (by value)**: Use merkle references encoded as [DAG-JSON] links to reference content that never changes
- **Mutable references (by address)**: Use `link` sigils to reference facts that can be updated over time

These reference types enable sophisticated data modeling including binary content references, computed data sources, and relational queries within facts, while preserving the causal integrity of the memory protocol. References are resolved by the [Schema Query Protocol], which provides the `/memory/graph/query` capability for advanced querying with automatic resolution.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## Motivation

The [Memory Protocol] provides a robust foundation for storing and querying facts in user-controlled spaces. However, applications need standardized ways to reference data across fact boundaries with different consistency guarantees:

- **Immutable references**: Reference content that should never change, with cryptographic integrity guarantees
- **Mutable references**: Reference facts that can be updated over time, with automatic propagation of changes
- **Binary content**: Handle binary data and file metadata consistently within the fact model
- **Computed relationships**: Establish dynamic relationships and computed values between facts
- **Reactive systems**: Build systems that respond to changes in referenced data

This RFC addresses these needs by defining two fundamental reference types that provide different consistency and mutability guarantees, enabling applications to choose the appropriate reference semantics for their use cases.

## Reference Types

This protocol defines two fundamental types of references between facts, each with different consistency and mutability guarantees:

### Immutable References (By Value)

Immutable references use [IPLD Links] to reference the content of a fact's `is` field using content addressing. These provide cryptographic integrity guarantees and never change:

```json
{
  "/": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5"
}
```

**Key Properties**:
- Reference only the content of the fact's `is` field, not the entire fact structure
- Use the [merkle reference] codec for content addressing
- Provide cryptographic integrity guarantees
- Enable content deduplication
- Never change - the same content always has the same reference

### Mutable References (By Address)

Mutable references use `link` sigils to reference facts that can be updated over time. These references automatically reflect changes to the target fact:

```json
{
  "/": {
    "link@1": {
      "type": "application/json",
      "id": "user:bob",
      "path": ["name"]
    }
  }
}
```

**Key Properties**:

- Reference current memory by fact (`the` and `of` fields)
- Automatically reflect changes when target fact is updated
- Support path navigation within the target fact's `is` field
- Provide configurable write behavior through the `replace` field

### DAG-JSON Convention

Both reference types adopt the [DAG-JSON] linking convention, where the `/` field contains reference metadata. This approach:

- Maintains compatibility with standard JSON parsers
- Provides a clear namespace for reference metadata separate from user data
- Enables progressive enhancement where simple JSON facts become linkable
- Supports efficient serialization and deserialization
- Preserves the memory protocol's fact structure

### Reference Resolution

References are resolved by the [Schema Query Protocol] to reduce roundtrips when following references across facts.

## Reference-Based Sigil Types

### Mutable References: Link Sigil (`link@1`)

The `link` sigil provides mutable references to JSON values held by other facts. If `path` is omitted it references whole JSON value - `is` field of the target fact. If `accept` is omitted, it defaults to the type of the linker (the fact containing the `link` sigil). If `source` is omitted, it defaults to the id of the linker. Therefore, if both are omitted, it creates a self-reference. The `overwrite` field controls write behavior.

> ℹ️ Therefore `link` with omitted `accept`, `source` and `path` represents a self-reference.

#### Fields

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
    source?: URI      // defaults to linker resource URI
    accept?: string   // HTTP Accept header format, defaults to linker's type
    path?: JSONPath
    space?: SpaceDID
    overwrite?: "this" | "redirect"  // defaults to "this"
    schema?: JSONSchema
  }
}
```

#### Resolution Behavior

Link sigils resolve to the current value at the specified location within the target fact's `is` field. When the target fact changes, all references automatically reflect the new value.

**Write Behavior**: When a link sigil is assigned a new value, the sigil itself is overwritten and replaced with the new value. The target fact remains unchanged.
- When `overwrite` is `"this"` (default): Writing to the link replaces the sigil itself with the new value
- When `overwrite` is `"redirect"`: Writing to the link follows the reference and modifies the target fact
- **Property assignments within the link**: Always mutate the target fact regardless of `overwrite` setting

#### Example with Default Values

When `accept` and `source` are omitted, they default to the linker's values, creating a self-reference:

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

#### Example

```json
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "name": "Alice Smith",
    "manager": {
      "/": {
        "link@1": {
          "accept": "application/json",
          "source": "user:bob",
          "path": ["name"],
          "overwrite": "redirect"
        }
      }
    }
  },
  "cause": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5"
}
```

#### Example with Default Behavior (`replace: "this"`)

```json
{
  "the": "application/json",
  "of": "dashboard:main",
  "is": {
    "currentUser": {
      "/": {
        "link@1": {
          "accept": "application/json",
          "source": "session:current",
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

## Link Write Behavior Examples

The `overwrite` field controls how property assignment behaves:

### Default Behavior (`overwrite: "this"`)

When you assign to a link with default settings, you **replace the link sigil itself**:

```json
// Initial state
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "manager": {
      "/": {
        "link@1": {
          "accept": "application/json",
          "source": "user:bob",
          "path": ["name"]
        }
      }
    }
  }
}

// After alice.manager = "Charlie"
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "manager": "Charlie"  // Link sigil replaced with literal value
  }
}
```

The target fact (`user:bob`) remains unchanged.

### Link Replacement (`overwrite: "redirect"`)

When you assign to a link with `overwrite: "redirect"`, you **replace the target fact's `is` field**:

```json
// Initial state
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "manager": {
      "/": {
        "link@1": {
          "accept": "application/json",
          "source": "user:bob",
          "path": ["name"]
        }
      }
    }
  }
}

// After alice.manager = "Charlie"
// The link remains, but user:bob fact is modified
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "manager": {
      "/": {
        "link@1": {
          "accept": "application/json",
          "source": "user:bob",
          "path": ["name"],
          "overwrite": "redirect"
        }
      }
    }
  }
}

// And user:bob fact is updated:
{
  "the": "application/json",
  "of": "user:bob",
  "is": "Charlie"  // Entire `is` field replaced
}
```

### Property Mutation (Always Affects Target)

Mutations within the linked value always affect the target fact regardless of `overwrite` setting. Here we link the entire user:bob object and then mutate a property within it:

```json
// alice.manager links the whole user:bob object
{
  "the": "application/json",
  "of": "user:alice",
  "is": {
    "name": "Alice Smith",
    "manager": {
      "/": {
        "link@1": {
          "accept": "application/json",
          "source": "user:bob"
        }
      }
    }
  }
}

// alice.manager.name = "Charlie" always mutates user:bob
{
  "the": "application/json",
  "of": "user:bob",
  "is": {
    "name": "Charlie",  // Target fact's property modified
    "department": "Engineering",
    "role": "Manager"
  }
}
```

### Use Cases

- **`overwrite: "this"`**: Use when you want a mutable reference that can be replaced with direct values
- **`overwrite: "redirect"`**: Use when you want transparent redirection, like permanent redirects or symbolic links

### Binary Content Sigils

Binary content sigils are references that need to be interpreted as `Blob` instances. They can use either immutable references for content that never changes, or mutable references (link sigils) for binary data that can be updated.

#### Blob Sigil (`blob@1`)

The blob sigil provides references to binary data that should be interpreted as a `Blob` instance. It can reference either immutable content via immutable references or mutable binary facts via link sigils.

#### Fields

- `type` (optional): Media type of the binary data. If omitted and content is referenced by immutable reference, it will be inferred from content as either `application/json` or `application/octet-stream`. If content is referenced by link sigil, content type will be the type of resolved data.
- `content` (required): Either a link sigil referencing a fact, an immutable reference to immutable content, or inline binary data using bytes representation

#### TypeScript Definition

```typescript
type BlobSigil = {
  "blob@1": {
    type?: MIME // defaults to application/octet-stream
    content: LinkSigil | Reference | Bytes
  }
}

type Reference = {
  "/": string // Merkle reference to fact `is` field content
}

type Bytes = {
  "/": {
    bytes: string // base64-encoded binary data
  }
}
```

#### Resolution Behavior

When `content` is inline bytes, the binary data is embedded directly and the content type defaults to `application/octet-stream` if no `type` field is specified.

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
  },
  "cause": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5"
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
  },
  "cause": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5"
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
  },
  "cause": "ca4jda8sw6emr7p7xxydz0klw8rpgxndykaedspd8z7kfs9go7b8vhmsu8"
}
```

#### File Sigil (`file@1`)

The file sigil is an extension of the blob sigil that provides references to binary data interpreted as `File` instances. It has the same type inference rules as blob sigils, plus an optional `name` field for filesystem metadata.

#### Fields

- `type` (optional): Media type of the binary data. If omitted and content is referenced by immutable reference, it will be inferred from content as either `application/json` or `application/octet-stream`. If content is referenced by link sigil, content type will be the type of resolved data. If content is inline bytes, defaults to `application/octet-stream`.
- `content` (optional): Either a link sigil referencing a fact, an immutable reference to immutable content, or inline binary data using bytes representation. If omitted, only metadata is provided
- `name` (optional): Filename

#### TypeScript Definition

```typescript
type FileSigil = {
  "file@1": {
    type?: MIME    // same inference rules as blob sigil
    content?: LinkSigil | Reference | Bytes // optional reference to binary data
    name?: string  // optional filename
  }
}
```

#### Resolution Behavior

When `content` is inline bytes, the content type defaults to `application/octet-stream` if no `type` field is specified.

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
          "type": "application/pdf",
          "content": {
            "/": {
              "link@1": {
                "accept": "application/pdf",
                "source": "blob:report-pdf-2024"
              }
            }
          },
          "name": "annual-report-2024.pdf"
        }
      }
    }
  },
  "cause": "ca5kbd8su5emr6n6vvwdz8jiu6pnfvlbwizccqnd6x5idq7em5z6tfls6"
}
```

### Computational Sigils

Computational sigils use mutable references to create dynamic relationships, computed values, and data transformations based on other facts.

#### Charm Sigil (`charm@1`)

The charm sigil provides references to TypeScript modules that compute values based on input data and write results to specified outputs. It enables computed values within facts by executing TypeScript code with specified imports, inputs, and outputs.

#### Fields

- `language` (required): Programming language, currently only `"typescript"` is supported
- `imports` (required): Record mapping import names to immutable references containing the module code
- `exports` (required): Object with `"."` property pointing to the entry point module via immutable reference
- `main` (optional): Name of the main export function, defaults to `"default"`
- `input` (required): Arguments passed to the main function, can be a record of named inputs or a single JSON value, supporting embed sigils for dynamic data
- `output` (required): Destination where the return value is written, can be a record of named outputs or a single destination, supporting embed sigils for dynamic targets

#### TypeScript Definition

```typescript
type CharmSigil = {
  "charm@1": {
    language: "typescript"
    imports: Record<string, Reference>
    exports: { ".": Reference }
    main?: string
    input: Record<string, EmbedSigil | JSONValue> | JSONValue
    output: Record<string, EmbedSigil | JSONValue> | JSONValue
  }
}
```

#### Resolution Behavior

Charms resolve by loading the TypeScript module from the entry point reference, importing any specified dependencies, calling the main function with the provided input, and writing the result to the specified output destination. The TypeScript code executes in a sandboxed environment with access only to the explicitly imported modules.

#### Example

```json
{
  "the": "application/json",
  "of": "report:monthly",
  "is": {
    "totalSales": {
      "/": {
        "charm@1": {
          "language": "typescript",
          "imports": {
            "lodash": { "/": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5" }
          },
          "exports": {
            ".": { "/": "ca5kbd8su5emr6n6vvwdz8jiu6pnfvlbwizccqnd6x5idq7em5z6tfls6" }
          },
          "main": "calculateTotal",
          "input": {
            "salesData": {
              "/": {
                "embed@1": {
                  "accept": "application/json",
                  "source": "sales:oeu242"
                }
              }
            }
          },
          "output": {
            "/": {
              "embed@1": {
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
- Sigils reference other facts using their `{accept, source}` coordinates

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

Implementations MUST handle link sigil writes based on the operation type and `overwrite` setting:

#### Property Assignment

For property assignment (`alice.manager = value`), behavior depends on the `overwrite` field:

**`overwrite: "this"` (default)**:
1. **Replace sigil**: Replace the entire link sigil with the new value
2. **Preserve target**: The target fact referenced by the link remains unchanged
3. **Local mutation**: The write affects only the containing fact

**`overwrite: "redirect"`**:
1. **Follow link**: Follow the link to the target fact first
2. **Replace target**: Replace the target fact's `is` field with the new value
3. **Preserve link**: The link sigil itself remains unchanged in the containing fact
4. **Validate**: Apply schema validation if specified before writing to target
5. **Maintain causality**: Ensure writes maintain proper fact causal chains for the target fact

#### Property Mutation

For property mutation (`alice.manager.name = value`), implementations MUST:

1. **Always follow link**: Follow the link to the target fact regardless of `overwrite` setting
2. **Route writes**: Redirect write operations to the underlying target fact at the specified path
3. **Preserve link**: The link sigil itself remains unchanged in the containing fact
4. **Validate**: Apply schema validation if specified before writing to target
5. **Maintain causality**: Ensure writes maintain proper fact causal chains for the target fact
6. **Transaction integrity**: Process link writes within proper memory protocol transactions

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

Sigil types include version suffixes (e.g., `link@1`) to enable backward-compatible evolution:

1. **New versions**: Can add fields or change behavior while maintaining old version support
2. **Deprecation**: Old versions should be supported during transition periods
3. **Migration tools**: Implementations SHOULD provide utilities for upgrading sigil versions

#### TypeScript Version Support

Implementations can use TypeScript's union types to support multiple versions:

```typescript
// Support for multiple versions of the same sigil
type LinkSigilAny = LinkSigil1 | LinkSigil2 // when @2 is introduced

type LinkSigil1 = {
  "link@1": {
    accept?: string
    source?: URI
    path?: JSONPath
    space?: SpaceDID
    overwrite?: "this" | "redirect"
    schema?: JSONSchema
  }
}

// Future version example
type LinkSigil2 = {
  "link@2": {
    accept?: string
    source?: URI
    path?: JSONPath
    space?: SpaceDID
    overwrite?: "this" | "redirect"
    schema?: JSONSchema
    // hypothetical new fields
    cache?: boolean
    timeout?: number
  }
}

// Version-aware sigil processing
function processLinkSigil(sigil: LinkSigilAny): unknown {
  if ("link@1" in sigil) {
    return processLink1(sigil["link@1"])
  } else if ("link@2" in sigil) {
    return processLink2(sigil["link@2"])
  }
  throw new Error("Unsupported link sigil version")
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
        "link@1": {
          "accept": "application/json",
          "source": "user:current",
          "path": ["name"]
        }
      }
    },
    "logo": {
      "/": {
        "file@1": {
          "type": "image/png",
          "content": {
            "/": "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5"
          },
          "name": "company-logo.png"
        }
      }
    },
    "metrics": {
      "/": {
        "charm@1": {
          "spell": "aggregateMetrics",
          "sources": [
            {"accept": "application/json", "source": "metrics:*"}
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
                "link@1": {
                  "accept": "application/json",
                  "source": "config:default"
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
            "accept": "application/json",
            "source": "document:project-status"
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
                "link@1": {
                  "accept": "application/json",
                  "source": "profile:alice-public"
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
                  "type": "image/jpeg",
                  "content": {
                    "/": {
                      "link@1": {
                        "accept": "image/jpeg",
                        "source": "blob:avatar-alice-profile"
                      }
                    }
                  }
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
[IPLD Links]: https://ipld.io/specs/codecs/dag-json/spec/#links
[merkle reference]: https://github.com/Gozala/merkle-reference
[Blob]: https://developer.mozilla.org/en-US/docs/Web/API/Blob
[Accept header]:developer.mozilla.org/en-us/docs/web/http/reference/headers/accept
