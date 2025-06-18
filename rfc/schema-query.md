# Schema Query Protocol: Graph Querying for Memory Protocol

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [Irakli Gozalishvili], [Common Tools]

## Authors

- [Bernhard Seefeld], [Common Tools]

## Abstract

This RFC extends the [Memory Protocol] with graph querying capabilities that enable complex queries across fact relationships. The schema query protocol introduces the `/memory/graph/query` capability, which provides a different querying mechanism from the basic `/memory/query` capability, including reference resolution, graph traversal, and schema-aware operations.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## Motivation

The basic `/memory/query` capability in the [Memory Protocol] provides efficient fact retrieval but requires multiple roundtrips when working with references that point to other facts. Additionally, it returns complete facts even when only a subset of the data is needed.

The schema query protocol reduces roundtrips by providing a single query capability that can follow cross-fact references and collect all relevant facts in a single response bundle. It uses schemas to determine which subset of fact data is actually needed, improving efficiency by excluding irrelevant fields.

## Core Concepts

### Reference Following

Schema queries can follow cross-fact references during query execution, collecting related facts to build comprehensive result sets in a single roundtrip.

### Bundled Responses

Rather than requiring multiple queries to resolve cross-fact references, schema queries return bundled responses containing all relevant facts.

### Schema-Based Filtering

Schema queries use schemas to determine which portions of facts are actually needed. A fact may contain extensive data, but the schema specifies only the subset required for the query, reducing response size and improving performance.

## Capabilities

### `/memory/graph/query`

*[Detailed specification to be completed]*

This capability provides:

- Automatic reference resolution
- Graph traversal operations
- Schema-aware filtering
- Advanced aggregation functions

## Integration with Cross-Fact References

Schema queries are designed to work seamlessly with cross-fact references:

- **References** are resolved to their target values during query execution
- **Complex reference networks** can be traversed and queried as graph structures

## Examples

*[Examples to be provided in future versions]*

## Implementation Requirements

*[Implementation requirements to be specified]*

## Future Work

This specification is currently in draft status. Future versions will include:

- Complete capability specification for `/memory/graph/query`
- Detailed query language syntax
- Performance optimization guidelines
- Security considerations for cross-space graph queries
- Integration examples with reference resolution

[Common Tools]: https://common.tools/
[Irakli Gozalishvili]: https://github.com/gozala
[Memory Protocol]: ./memory.md

