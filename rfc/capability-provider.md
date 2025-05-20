# Capability Provider Protocol

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [Irakli Gozalishvili], [Common Tools]

## Authors

- [Irakli Gozalishvili], [Common Tools]

## Abstract


This document describes the **Capability Provider Protocol**, which enables entities in the space (and spaces themselves) to be provisioned with capabilities that external systems can invoke. At its core, any entity can be provisioned with a capability by asserting a fact like `{ the: "http/get", of: "entity:5d59a2ff", is: "route:8b3a-547063a2fd23#handle" }`.
The protocol bridges the Common Fabric with the broader web ecosystem by allowing spaces to self-host programmable interfaces. This approach empowers space owners to customize how their spaces interface with the outside world.

This RFC defines the `http/` capability family that allows spaces to interface with the web ecosystem, while establishing patterns that can extend to other capability families in the future. By designating providers for `http/` capabilities, entities can participate in the web as first-class citizens with their own URLs, serving content and processing requests.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Core Concepts

### Capability Providers

A capability provider is an entity that has been assigned executable generator functions, enabling it to receive requests, yield capability invocations, and respond to clients. These providers operate within a supervisor hierarchy where entities' providers can invoke space capabilities, and space providers can invoke system capabilities.

Entities with assigned `http/` capability provider(s) can:

1. Respond to HTTP requests with custom logic
2. Expose programmable interfaces for interaction
3. Access and manipulate data by yielding capability invocations
4. Bridge between Common Fabric and external services

### Entities and Capabilities

In this protocol:

1. **Entities** are objects inside a space identified by stable URIs. Since a space is identified by [did:key], it is also an entity contained by itself.

2. **Capabilities** are provided by asserting facts:
   ```
   { the: "http/get", of: "inbox:", is: "route:5d59a2ff:#receive" }
   ```
   - `the`: Specifies the capability (e.g., `http/get`, `http/post`, `http/*`)
   - `of`: The entity providing the capability
   - `is`: Reference to the provider with optional export function

3. **HTTP Resolution**:
   - Entity within space: `/did:key:space/note:5d59a2ff/...`
   - Space capabilities: `/did:key:space/...` (acts as fallback route)
   - Space can handle not-found paths: `/did:key:space/not-found/path`

4. **Provider Implementation**:
   ```
   { the: "application/javascript", of: "provider:some-id", is: <code> }
   ```

5. **Supervisor Hierarchy**: Providers operate in a layered system where:
   - Entity providers yield capabilities that are handled by the space provider
   - Space providers yield capabilities that are handled by the system provider
   - When no appropriate provider exists, the system provider handles capabilities by default

## Implementation

### Publishing capabilities

Entity capabilities can be provisioned by asserting facts via the Memory Protocol.

```json
{
  "the": "http/get",  // or http/post, http/put, http/delete, http/* for all methods
  "of": "note:5d59a2ff",
  "is": "route:547063a2fd23#fetch"
}
```

Providers represent programs compiled into JavaScript modules also asserted through the Memory Protocol:

```json
{
  "the": "application/javascript",
  "of": "route:547063a2fd23#fetch",
  "is": "export function* fetch(request) { return new Response('Hello!', {status: 200}); }"
}
```

HTTP capabilities use a modular design:
- **Method-Specific Providers**: Separate providers for different HTTP methods (GET, POST, etc.)
- **General Handler**: `http/*` for handling all methods with one provider
- **Export Selection**: Fragment identifier (`#fetch`) specifies which generator function to call. Without a fragment, the `default` export is implied.

### URL Resolution and Request Handling

The protocol defines a clear mapping between URLs and capabilities:

1. **Entity in Space**: `https://common.tools/did:key:z6Mk.../note:5d59a2ff/...`
2. **Space Capabilities**: `https://common.tools/did:key:z6Mk.../any/path/not/matched`

When a request arrives:
1. The system identifies the entity from the URL path
2. It determines the HTTP method (GET, POST, etc.)
3. It looks for a capability provider matching that method for the entity
4. If no method-specific provider exists, it tries the `http/*` provider
5. If entity is not found or provides no matching capability it falls back to matching capability exposed by the space
6. The provider generator function is executed in a sandboxed environment
7. When the provider yields a capability invocation, the system follows a supervisor hierarchy:
   - Entity provider's yields are handled by the space provider (if it has matching capability)
   - Space provider's yields are handled by the system provider
   - If no providers are provisioned, the system provider handles invocations by default
8. The capability result is passed back to the provider by resuming the generator
9. The provider's final return value is sent back to the client as a response

### Provider Execution Environment

Providers run in a secure JavaScript sandbox that:
1. Isolates providers from each other and the underlying system
2. Enforces resource limits (CPU, memory, execution time)
3. Provides standard Web APIs (URL, Headers, etc.)
4. Blocks access to sensitive APIs and file system
5. Requires all dependencies to be bundled with the provider
6. Executes generator functions and handles yielded capability invocations
7. Resumes execution by returning capability results as generator values

### Data Access Through System Provider

Providers cannot directly access anything beyond what has been passed to them. They can, however, yield capability invocations to access built-in capabilities like those defined by the Memory Protocol:

```javascript
export function* handle(request) {
  // Query data through system provider by yielding capability invocation
  const posts = yield {
    cmd: "/memory/query",
    args: {
      select: {
        "note:5d59a2ff": {
          "application/json": {}
        }
      }
    }
  };

  // Could also delegate to another generator function
  // return yield* renderPosts(posts);

  return new Response(generateHTML(posts), { status: 200 });
}
```

This generator pattern offers several advantages:

1. **Suspension and Resumption**: Allows execution to be suspended at capability invocation points and resumed when results are available
2. **Delegation**: Makes delegation straightforward with `return yield* { ... }` pattern
3. **Proper permission enforcement**: Through explicit capability invocations
4. **Consistent access patterns**: Across all providers
5. **Clear separation**: Between provider code and capability invocations
6. **Transparent effect tracking**: All side effects are explicitly yielded as data
7. **Debugging and monitoring**: All capability invocations can be logged and inspected

### Alternative Provider Formats

The protocol supports multiple programming languages through a derivation mechanism:

```json
{
  "the": "application/typescript",
  "of": "agent:greet",
  "is": "export function* fetch(request: Request): Generator<any, Response> {\n  return new Response('Hello, World!', { status: 200 });\n}"
}
```

When TypeScript or other formats are used, an observer can automatically derive JavaScript code:

```json
{
  "the": "application/javascript",
  "of": "agent:greet",
  "is": "export function* fetch(request) {\n  return new Response('Hello, World!', { status: 200 });\n}"
}
```

This enables:
- Writing providers in TypeScript with type safety
- Supporting multiple exports in a single provider
- Using the fragment identifier to select which function to invoke
- Future support for other languages with appropriate transformations

### System Capabilities

Providers can access additional functionality through the system provider at `did:web:system.common.tools`:

1. **Memory Protocol Capabilities**:
   - `memory/query`: Query facts in the space
   - `memory/assert`: Add facts to the space
   - `memory/retract`: Remove facts from the space

2. **Utility Capabilities**:
   - Random value generation
   - DID resolution
   - Cross-entity data access (with permissions)

3. **External Service Integration**:
   - Language model access (`llm/generate`)
   - Blob storage
   - Other system services

// Example of using a system capability to access a language model:

```javascript
function* generateText(prompt) {
  // Yield a capability invocation to generate text using LLM
  // This will be handled by the supervisor hierarchy
  const result = yield {
    cmd: "/llm/generate",
    args: {
      prompt,
      max_tokens: 100
    }
  };

  return result;
}

// Example of delegation to another capability
function* processAndGenerateText(data) {
  // Use the yield* syntax for delegation
  // This transparently passes through all yields from the delegated generator
  return yield* generateText(`Process this data: ${JSON.stringify(data)}`);
}
```

### Future Capability Families

While this RFC focuses on HTTP capabilities, the architecture establishes patterns that can extend to other capability types:

1. **Scheduled Execution**: Cron-like capabilities for periodic tasks
   ```
   { the: "cron/daily", of: <entity>, is: "provider:task#handler" }
   ```

   ```javascript
   export function* handler() {
     // Yield capabilities for daily tasks
     yield { cmd: "/memory/query", args: {...} };
   }
   ```

2. **Event Handling**: Respond to system or entity events
   ```
   { the: "event/space-updated", of: <entity>, is: "provider:listener#onUpdate" }
   ```

   ```javascript
   export function* onUpdate(event) {
     // React to space updates by yielding capabilities
     const result = yield { cmd: "/memory/assert", args: {...} };

     // Delegate to notification handler if needed
     if (event.shouldNotify) {
       yield* sendNotifications(event, result);
     }
   }
   ```

3. **Data Processing**: Transform or analyze data
   ```
   { the: "process/transform", of: <entity>, is: "route:transformer#convert" }
   ```

   ```javascript
   export function* convert(data) {
     // Transform data by yielding capabilities
     const resources = yield { cmd: "/memory/query", args: {...} };
     // Process data
     const processedData = transform(resources, data);

     // Store the result by delegating to another capability
     yield* storeResult(processedData);

     return processedData;
   }
   ```

Future capability families may require explicit authorization through UCAN invocations, unlike the current HTTP capabilities which are directly accessible through web endpoints without additional authorization.

## Use Cases

The Capability Provider Protocol enables powerful integrations and applications:

1. **Web Applications and Sites**: Spaces can host complete web applications or static sites through generator functions
2. **RESTful APIs**: Create programmable APIs for entities and collections that yield capability invocations for data access
3. **External Data Ingestion**: Process web forms, API calls, and webhooks with clean capability flow
4. **Autonomous Agents**: Build agents that yield LLM capability invocations and operate on space data
5. **Federation**: Create applications that span multiple spaces and entities through capability yielding
6. **Integration**: Bridge Common Fabric with external systems and services through managed capability invocations
7. **Custom UIs**: Provide tailored user interfaces with explicit capability-yielding data access patterns

## Security and Privacy Considerations

1. **Sandboxed Execution**: Providers run in isolated environments with:
   - Resource limits (CPU, memory, timeout)
   - No direct file system or network access
   - Access limited to the provided APIs

2. **Permission Model**:
   - Only space owners can designate providers
   - Providers operate with space owner permissions via system provider
   - Cross-entity access requires explicit permissions

3. **Code Security**:
   - Provider generator functions are validated before execution
   - All dependencies must be bundled with the provider
   - Runtime errors are contained within the provider
   - Capability invocations are explicitly yielded and controlled

4. **Privacy Protection**:
   - Space owners control what data is exposed
   - Providers can implement access controls for sensitive data
   - System capabilities enforce permission boundaries
   - Data access is auditable and can be revoked


[Irakli Gozalishvili]: https://github.com/gozala
[Common Tools]: https://github.com/wnfs-wg
