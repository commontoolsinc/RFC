# Cohesion Protocol

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [Irakli Gozalishvili], [Common Tools]

## Authors

- [Irakli Gozalishvili], [Common Tools]

## Abstract

This document describes the **Cohesion Protocol**, which enables entities in a space to define rules for interpreting facts. The protocol defines two types of rules:

1. **Deductive Rules**: Act as fact pre-processors. They run on assertion, can query (memory) space and return derived facts that are included in the same transaction.
2. **Reactive Rules**: Act as fact post-processors. They are scheduled to run asynchronously, can query (memory) space, and perform system-provided effects. They return derived facts that are asserted in a follow-up transaction while maintaining consistency guarantees.

The protocol also defines an **HTTP Gateway** mechanism built on top of this protocol that can be utilized for exposing an HTTP interface for user spaces, while maintaining appropriate authorization boundaries.

> Enabling the HTTP Gateway requires (out of band) authorization of the principal operating the gateway server (e.g., `did:web:common.tools`) by delegating `/memory/transact` capability to it.
>
> This delegation is then used by the gateway service to assert facts containing HTTP request in the value (`is` field), allowing the space to interpret request through deployed rules, forming an interface with the outside world while retaining full control over which parts of the space are accessible and how.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Core Concepts

### Rules

Rules provide the computational substrate of the space.

#### Rule Definition

Rules are entities with an active assertion where

1. Type (`the` field) is the `"application/javascript"`.
2. Value (`is` field) is the ES module source code.

Rules CAN be added to the space by asserting a facts using [memory protocol].

### Rule Binding

Rule binding is a fact that maps entity (`of` field) and type (`the` field) to a desired export of the [rule definition] value. It is similar to a setter concept in the programming languages, when matching fact is asserted or retracted mapped rule gets executed.

Rule MUST be considered bound if space has corresponding **rule binding** - an active fact where:

1. Entity (`of` field) is same as of asserted fact.
1. Type (`the` field) is the same as of asserted fact with a `/` prefix.
2. Value (`is` field) is a URI for a rule entity in the space with an optional `#` fragment.
  - If there is a `#` fragment it denotes export name of the rule module.
  - If `#` fragment is omitted the `default` export of ther rule module is implied.

### Rule Execution

Rule MUST be executed when fact matching a binding is asserted or retracted. Valid implementation of the protocol MUST do perform following steps when facts are transected into memory

1. Resolve **rule binding** by selecting a fact where
  - Entity (`of` field) is matches asserted fact entity.
  - Type (`the` field) is matches asserted fact entity with `/` prefix.
  - Value (`is` field) is URI.

2. Resolve **rule** by selecting a fact where
  - Entity (`of` field) matches resolved rule binding value (`is` field) without `#` framgent.
  - Type (`the` field) matches `application/javascript`.
  - Value (`the` field) matches ES module source code.

3. Inovke **rule** by performing following steps
  - Evaluating ES module in a sandboxed JS runtime.
  - If rule binding value (`is` field) has `#` fragment call module export, with a matching name of the fragment, passing asserted/retracted fact.
  - If rule binding value (`is` field) has no `#` fragment, call `default` module export passing asserted/retracted fact.
  - Run returned generator folling rule execution steps until it returns or throws an exception.


### Rule Execution Steps

#### Deductive Rule Execution Steps

Deductive rules MUST be run in a **synchronously** inside a transaction call. If rule yields `{ memory/query: args }` command runtime MUST perform `/memory/query` invocation defined by [memory protocol] with the `args`. Execution MUST be resumed with a result of the invocation.

If rule yields command other than `/memory/query` runtime MUST resume execution with an exception, allowing rule to `catch` and continue execution.

If rule throws an exception transaction MUST fail with thrown error.

If rule returns a value it MUST be interpreted as derived facts. All derived facts MUST be included in the transaction. Derived assertion with same entity and attribute as fact that caused execution SHOULD NOT cause recursive rule call, instead it should assert fact directly.



#### Reactive Rule Execution Steps

Reactive rules MUST be executed **asynchronously** outside a transaction call. If rule yields `{ memory/query: args }` command runtime MUST perform `/memory/query` invocation defined by [memory protocol] with the provided `args`. Execution MUST be resumed with a result of the invocation.

Unlike deductive rules, reactive rules MAY yield other, potentially effectful, commands provided by the system. Runtime MUST perform commands using corresponding [UCAN Invocation] with `cmd` corresponding to a key of the yielded object, and value as `args`. Runtime MAY choose which commands reacitve rule are given access to.

> ⚠️ It is RECOMMENDED that runtime does not provide commands able to transact changes in the space as those would bypass consistency guarantees.

Reactive rules sterm mus be run with an **optimistic concurrency**:

1. Runtime MUST record each facts queried during execution
1. Returned value MUST be interpreted as derived facts.
1. If any of the queried fact has changed rule execution MUST be retried with an exponential beckoff.
1. If max retry limit is reached runtum SHOULD block transactions for facts queried by rule to gurantee consistency.
1. Derived facts MUST be transacted.

This ensures **causal consistency** guarantee - changes will be applied correctly once all dependencies are stable.


**Deductive Rule Example**:
```ts
{
  "iss": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "sub": "did:key:z6MkffDZCkCTWreg8868fG1FGFogcJj5X6PY93pPcWDn9bob",
  "cmd": "/memory/transact",
  "args": {
    "changes": {
      "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK": {
        // 1. Rule defintion
        "rule:5d59a2ff": {
          "application/javascript": {
            "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
              "is": `export default function* ({ the, of, is }) {
                // Query data
                const data = yield { 'memory/select': { ... } };
                // Process data
                const summaryData = computeSummary(is);

                // Return changes object (no / prefix in the result)
                return [{ the: "inbox/summary", of, is: summaryData }]
              }`
            }
          }
        },

        // 2. Rule binding
        "note:zh7Mf5mK": {
          "inbox/recive": {
            "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
              "is": "rule:5d59a2ff"
            }
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

**Reactive Rule Example**:

```ts
{
<<<<<<< HEAD
  "the": "application/javascript",
  "of": "route:547063a2fd23",
  "is": "export function* fetch(request) { return new Response('Hello!', {status: 200}); }"
=======
  "iss": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "sub": "did:key:z6MkffDZCkCTWreg8868fG1FGFogcJj5X6PY93pPcWDn9bob",
  "cmd": "/memory/transact",
  "args": {
    "changes": {
      "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK": {
        // 1. Rule definition
        "rule:7e31a2ff": {
          "application/javascript": {
            "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
              "is": `export function* summarize({ the, of, is }) {
                const summary = yield { 'llm/prompt': { prompt: is } };
                const entity = yield { 'data/refer': { the, summary, of } };
                return [{ the, of: entity, is: summary }]
              }`
            }
          }
        },

        // 2. Rule binding
        "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK": {
          "/idea/capture!": {
            "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
              "is": "rule:7e31a2ff"
            }
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
>>>>>>> 42ce53e (overhaul the protocol)
}
```

### HTTP Gateway

The HTTP Gateway is a service that maps HTTP requests to corresponding rules and triggers them using authorization delegated from the space. This allows spaces to bind rules to desired HTTP endpoints, forming a critical bridge between the Common Fabric and outside systems, enabling entities to participate in the web ecosystem.

The gateway maps HTTP requests to rule triggering on entities with the `http+` prefix.

Through the HTTP Gateway, entities can:

1. Receive and process external HTTP requests with custom logic
2. Apply rules to HTTP inputs to pre-process or transform them
3. Access and manipulate data by yielding queries or perform effects
4. Respond with HTTP responses
5. Bridge between Common Fabric and external services

To enable the HTTP Gateway, a space needs to delegate to `did:web:common.tools` the capability to invoke `/memory/transact` on `http+` prefixed entities. This delegation enables the gateway service to route HTTP requests to the appropriate rules, empowering space owners to customize how their spaces interface with the outside world while retaining control over which parts are accessible without authorization.

The HTTP Gateway exposes rules through entities prefixed with `http+` as shown in the example below:

```js
{
  "iss": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "sub": "did:key:z6MkffDZCkCTWreg8868fG1FGFogcJj5X6PY93pPcWDn9bob",
  "cmd": "/memory/transact",
  "args": {
    "changes": {
      "http+did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK": {
        "/text/plain": {
          "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
            "is": "hello:8b3a"
          }
        },
        "http+note:5d59a2ff": {
          "/text/markdown": {
            "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
              "is": "md:pmc5w4hcp6d:#append"
            }
          }
        },
        "hello:8b3a": {
          "application/javascript": {
            "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
              "is": `export default function* ({ the, of, is: request }) {
                  return [{ the, of, is: new Response('Hello!', {status: 200}) }]
                }`
              }
            }
          }
        },
        "md:pmc5w4hcp6d": {
          "application/javascript": {
            "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
              "is": "export function append*(request) { ... }"
            }
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

##### URL to Entity Mapping

Routes are mapped to entities as follows:
- **Entity in Space**: `/did:key:space/note:5d59a2ff/...` maps to the `http+note:5d59a2ff` entity
- **Space itself**: `/did:key:space/whatever/else` maps to the `http+did:key:space` entity

The gateway uses a simple rule: if the component after the DID has a `:` character, it is considered a route to that specific component; otherwise, it's a route to the space itself.

##### Content-Type to Capability Mapping

The HTTP request's Content-Type header determines which capability is invoked:
- The Content-Type is normalized to lowercase
- A `/` prefix is added to form the capability name
// For example: a request with `Content-Type: application/json` maps to a capability with `the: "/application/json"`

##### Request Processing Flow

When an HTTP request arrives at the gateway:
1. The gateway identifies the target entity from the URL path using the mapping rules above
2. It determines the normalized Content-Type of the request
3. It constructs a rule-triggering fact with `/` prefixed Content-Type and looks for a matching rule binding (without the `/` prefix) for the `http+`-prefixed entity
4. If no media-type-specific rule binding exists, it tries the general fallback rule on the `http+`-prefixed entity
5. If the entity is not found or has no matching rule binding, it falls back to matching rule bindings exposed by the `http+`-prefixed space
6. The rule implementation's generator function is executed in a sandboxed environment with the HTTP request as input
7. When the rule yields a query or reactive command, the execution hierarchy processes it
8. The result is passed back to the rule by resuming the generator
9. For deductive rules, the returned changes are applied within the same transaction, and any HTTP response data is sent back to the client
10. For reactive rules, the rule is scheduled for asynchronous execution, and when it completes, its return value (Changes) is transacted with consistency guarantees, and any response data is sent back to the client. The HTTP connection might be maintained, or a webhook/callback mechanism might be used for long-running reactive rules.
### Execution Environment

All rules run in a secure JavaScript sandbox that:
1. Isolates code from each other and the underlying system
2. Enforces resource limits (CPU, memory, execution time)
3. Provides standard Web APIs (URL, Headers, etc.)
4. Blocks access to sensitive APIs and file system
5. Requires all dependencies to be bundled with the code
6. Executes generator functions and handles yielded effect requests
7. Resumes execution by returning results as generator values

### Data Access Through Effect Requests

Rules cannot directly access anything beyond what has been passed to them. They can, however, yield effect requests to access built-in effects like those defined by the Memory Protocol:

```javascript
// Example: Deductive rule handler
export function* handle(args) {
  // Query data through system by yielding query command
  const posts = yield {
    memory/query: {
      "note:5d59a2ff": {
        "application/json": {}
      }
    }
  };

  // Process the data
  const processedData = processData(args.input, posts);

  // Return changes that typically assert a fact on the same entity
  // with matching `the` field (without the `/` prefix)
  return {
    changes: {
      [args.of]: {
        // Note: no `/` prefix here - matches the triggering fact's `the` field
        "inbox/summary": {
          processedData
        }
      }
    }
  };
}

// Example: Reactive rule handler
export function* handleEffect(args) {
  // Query data through system by yielding query command
  const posts = yield {
    memory/query: {
      "note:5d59a2ff": {
        "application/json": {}
      }
    }
  };

  // Perform effects (only allowed in reactive rules)
  const externalData = yield {
    'external/service!': { input: args.input }
  };

  // Process the data and prepare changes
  const processedData = processData(args.input, posts, externalData);
  const entityId = `processed:${generateId()}`;

  // Return Changes to be transacted with consistency guarantees
  return {
    changes: {
      [entityId]: {
        "application/json": {
          processedData
        }
      }
    }
  };
}

// Example: HTTP gateway deductive rule handler
export function* receive(request) {
  // Parse request body
  const inputData = await request.json();

  // Query data through system by yielding query command
  const posts = yield {
    memory/query: {
      "note:5d59a2ff": {
        "application/json": {}
      }
    }
  };

  // Process data and return HTTP response
  return new Response(generateHTML(posts, inputData), { status: 200 });
}

// Example: HTTP gateway reactive rule handler
export function* receiveAndProcess(request) {
  // Parse request body
  const inputData = await request.json();

  // Query data through system by yielding query command
  const posts = yield {
    memory/query: {
      "note:5d59a2ff": {
        "application/json": {}
      }
    }
  };

  // Process data
  const processedData = processData(inputData, posts);
  const entityId = `submission:${generateId()}`;

  // Return Changes to be transacted and HTTP response data
  return {
    changes: {
      [entityId]: {
        "application/json": {
          processedData
        }
      }
    },
    response: {
      status: 200,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ status: "success", id: entityId })
    }
  };
}
```

This generator pattern offers several cohesive advantages:

1. **Suspension and Resumption**: Allows execution to be suspended at query or reactive command points and resumed when results are available
2. **Delegation**: Makes delegation straightforward with `return yield* { ... }` pattern
3. **Proper permission enforcement**: Through explicit commands and authorization checks
4. **Consistency guarantees**: Reactive rules provide transactional consistency for changes
5. **Clear separation**: Between rule code and commands/effects
6. **Transparent effect tracking**: All side effects are explicitly yielded as data
7. **Debugging and monitoring**: All commands and effects can be logged and inspected

The distinction between deductive and reactive rules provides additional cohesive benefits:
1. **Safety guarantees**: Deductive rules cannot perform effects, ensuring transaction safety
2. **Concurrency control**: Reactive rules have appropriate transactional semantics
3. **Clear intent**: The rule type clearly communicates its purpose and behavior
4. **Appropriate guarantees**: Each rule type provides the guarantees it needs
5. **Execution model clarity**: Synchronous blocking execution for deductive rules, asynchronous scheduled execution for reactive rules
6. **Reliability**: Reactive rules can be retried by the system to ensure successful execution

### Rule Execution Lifecycle

The execution lifecycle of rules differs significantly based on their type:

#### Database Analogies

The Cohesion Protocol's rule system has strong parallels to relational database concepts:

- **Deductive Rules** are analogous to **stored procedures** in relational databases:
  - Execute synchronously within the current transaction
  - Can only read data (not modify external state)
  - Return results that are processed immediately
  - Block the transaction until they complete
  - Fail the entire transaction if they encounter errors

- **Reactive Rules** are similar to **database triggers**:
  - Execute asynchronously after the triggering transaction
  - Can perform side effects and external operations
  - Run in a separate transaction context
  - May be retried multiple times to ensure consistency
  - Support eventual consistency with optimistic concurrency

These analogies can help developers understand how and when to use each type of rule.

#### Rule Execution Lifecycle

The execution lifecycle differs significantly between deductive and reactive rules:

**Deductive Rule Lifecycle**:
1. **Triggering**: A fact with `/` prefix in the `the` field is asserted that matches a rule binding with the same name (without the prefix)
2. **Synchronous Execution**: Executes immediately within the same transaction
3. **Blocking Behavior**: The transaction is blocked until completion
4. **Transaction Scope**: Execution is contained within a single transaction
5. **Result Handling**: Changes are applied within the same transaction
6. **Immediate Consistency**: Changes are immediately visible
7. **No Retries**: Once committed, execution is final
8. **Effect Restrictions**: Attempts to perform effects cause transaction failure

**Reactive Rule Lifecycle**:
1. **Triggering**: A fact is asserted that matches a rule binding with a `!` suffix
2. **Scheduling**: Rule is scheduled for asynchronous execution after triggering transaction completes
3. **Non-blocking**: Triggering transaction completes independently without waiting
4. **Separate Context**: Rule executes in its own transaction context
5. **Effect Support**: Can perform effects by yielding effectful commands with `!` suffix
6. **Optimistic Execution**: Executes with a snapshot of the state, tracking all read facts
7. **Consistency Check**: Before committing, verifies all read facts are unchanged
8. **Conflict Resolution**: If read facts changed, re-executes with updated state
9. **Retry Logic**: Uses exponential backoff for transient failures
10. **Eventual Consistency**: Changes are eventually applied correctly once dependencies stabilize

These differences provide important guarantees:
- Deductive rules: Immediate, consistent changes within the transaction
- Reactive rules: Eventual consistency with retry semantics for reliability

### Alternative Implementation Formats

The protocol supports multiple programming languages through built-in system rules:

```json
{
  "the": "application/typescript",
  "of": "agent:greet",
  "is": "export function* receive(request: Request): Generator<any, Response> {\n  return new Response('Hello, World!', { status: 200 });\n}"
}
```

This is handled by the system-provided `/application/typescript` rule, which transforms TypeScript into JavaScript:

```json
{
  "the": "application/javascript",
  "of": "agent:greet",
  "is": "export function* receive(request) {\n  return new Response('Hello, World!', { status: 200 });\n}"
}
```

This approach enables:
- Writing rules in TypeScript with type safety
- Supporting multiple exports in a single module
- Using the fragment identifier to select which function to invoke
- Future support for other languages with appropriate transformations

### System-Provided Rules

The Cohesion Protocol provides a set of built-in rules:

1. **Memory Protocol Rule Bindings**:
   - Deductive: `memory/query`: Query facts in the space
   - Reactive: `memory/transact!`: Transacts assertions / retractions

2. **HTTP Gateway Default Rule Bindings**:
   - Deductive: `*`: The fallback rule binding that handles simple HTTP requests
   - Deductive: `application/json`: Handles simple JSON HTTP requests
   - Reactive: `application/json!`: Handles JSON HTTP requests with effects
   - Deductive: `text/html`: Handles simple HTML HTTP requests
   - Reactive: `text/html!`: Handles HTML HTTP requests with effects

3. **Format Processing Rule Bindings**:
   - Deductive: `application/typescript`: Processes TypeScript and returns JavaScript
   - Reactive: `application/typescript!`: Processes TypeScript and asserts JavaScript facts

4. **External Service Integration Rule Bindings**:
   - Reactive: `llm/generate!`: Language model access (requires authorization)
   - Reactive: `storage/blob!`: Blob storage (requires authorization)
   - Other system services


// Examples of yielding effect requests:
// Examples of different rule types:

```javascript
// Example of a deductive rule
export function* processInput(input) {
  // Can only perform queries in deductive rules
  const context = yield {
    memory/query: {
      "context:latest": {
        "application/json": {}
      }
    }
  };

  // Return changes object
  return {
    changes: {
      [input.of]: {
        // Note this matches the triggering fact's `the` field (without the `/` prefix)
        "input/processed": {
          processed: combineWithContext(input, context),
          timestamp: new Date().toISOString()
        }
      }
    }
  };
}

// Example of a reactive rule using LLM
export function* generateAndStoreText(prompt) {
  // Reactive rules can perform effects (note the ! suffix in command)
    // Yield an effectful command to generate text using LLM
      const result = yield {
        'llm/generate!': {
        prompt,
        max_tokens: 100
      }
    };

  const generationId = `generation:${generateId()}`;

  // Return Changes to be transacted
  return {
    changes: {
      [generationId]: {
        "application/json": {
          text: result,
          prompt,
          timestamp: new Date().toISOString()
        }
      }
    }
  };
}

// Example of a rule implementation that delegates to another rule
export function* processData(data) {
  // For deductive rules, delegation passes through the changes
  const result = yield* processInput(data);

  // We can then add to or modify the changes as needed
  const enhancedChanges = {
    ...result.changes
  };

  // Add enhanced data to the same entity
  enhancedChanges[data.of]["data/enhanced"] = {
    enhanced: enhanceData(result.changes[data.of]["input/processed"]),
    source: data
  };

  return { changes: enhancedChanges };
}

// Example of an HTTP gateway deductive rule
export function* receiveQuery(request) {
  // Parse the user's query from the request
  const { query } = await request.json();
  const queryId = `query:${generateId()}`;

  // Query data
  const results = yield {
    memory/query: {
      [`query:${query}`]: {
        "application/json": {}
      }
    }
  };

  // Return both changes and HTTP response
  return {
    changes: {
      [queryId]: {
        "query/results": {
          query,
          results,
          timestamp: new Date().toISOString()
        }
      }
    },
    response: {
      status: 200,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(results)
    }
  };
}
```

### Future Rule Types

### Future Rule Types

The Cohesion Protocol architecture establishes patterns that can extend to various rule types:

1. **Scheduled Execution**:
   ```
   { the: "/cron/daily!", of: <entity>, is: "rule:task#handler" }
   ```

   ```javascript
   export function* handler() {
     // Yield query command for daily tasks
     const data = yield { memory/query: {...} };

     // Return Changes with deduced time-based facts
     return {
       changes: {
         "report:today": {
           "statistics/daily": {
             stats: deduceDailyStats(data),
             timestamp: new Date().toISOString()
           }
         }
       }
     };
   }
   ```

2. **Event Handling**:
   ```
   { the: "/event/subscribe!", of: <entity>, is: "rule:listener#subscribe" }
   ```

   ```javascript
   export function* subscribe(eventType) {
     // Register for event notifications
     return yield {
       cmd: "/event/register!",
       args: {
         type: eventType,
         handler: "entity:this#handleEvent"
       }
     };
   }

   export function* handleEvent(event) {
     // Process the event and return Changes
     return {
       changes: {
         [`log:${generateId()}`]: {
           "event/log": {
             type: event.type,
             timestamp: new Date().toISOString(),
             data: event.data
           }
         }
       }
     };
   }
   ```

3. **Data Processing**:

   Deductive version:
   ```
   { the: "/transform/data", of: <entity>, is: "rule:transformer#convert" }
   ```

   ```javascript
   export function* convert(data) {
     // Query existing facts using query command
     const resources = yield { memory/query: {...} };

     // Return processed data as the `is` field value
     return processData(resources, data);
   }
   ```

   Effectful version:
   ```
   { the: "/transform/data!", of: <entity>, is: "rule:transformer#convertWithEffects" }
   ```

   ```javascript
   export function* convertWithEffects(data) {
     // Query existing facts
     const resources = yield { memory/query: {...} };

     // Process data
     const processedData = processData(resources, data);

     // Return Changes to be transacted
     return {
       changes: {
         [`processed:${generateId()}`]: {
           "application/json": processedData
         }
       }
     };
   }
   ```

4. **Media Type Processing**:

   Deductive version:
   ```
   { the: "/application/markdown", of: <entity>, is: "rule:markdown#process" }
   ```

   ```javascript
   export function* process(content) {
     // Process markdown content
     const html = convertMarkdownToHtml(content);

     // Return changes asserting the HTML content
     return {
       changes: {
         [content.of]: {
           // Typically matches triggering fact's `the` field without the `/` prefix
           "application/markdown/html": html
         }
       }
     };
   }
   ```

   Effectful version (for HTTP):
   ```
   { the: "/application/markdown!", of: "http+entity:markdown", is: "rule:markdown#processAndStore" }
   ```

   ```javascript
   export function* processAndStore(request) {
     // Extract markdown content from the request
     const content = await request.text();
     const contentId = `content:${generateId()}`;

     // Process markdown content
     const html = convertMarkdownToHtml(content);

     // Return Changes and HTTP response
     return {
       changes: {
         [contentId]: {
           "text/html": html,
           "text/markdown": content
         }
       },
       response: {
         status: 200,
         headers: { "Content-Type": "text/html" },
         body: html
       }
     };
   }
   ```

The Cohesion Protocol provides flexibility to extend rule types with the `/` prefix, optionally adding an `!` suffix for effectful rules. These rules can be directly triggered through authorized UCAN invocations or exposed to HTTP through the gateway mechanism (via `http+` prefixed entities). This unified approach allows for consistent rule design while accommodating different security requirements and access patterns.

### Use Cases

The Cohesion Protocol enables powerful integrations and applications:

#### Deductive Rule Use Cases (Synchronous Processing)

1. **Virtual Properties**: Define computed properties within the same transaction
2. **Data Transformation**: Transform data into different formats or structures
3. **Content Derivation**: Generate derived content like summaries or previews
4. **Query Enhancement**: Enhance queries with additional context or parameters
5. **Input Validation**: Validate and normalize inputs before processing
6. **Content Formatting**: Format content for different presentations
7. **Computed Fields**: Generate computed fields based on other facts
8. **Default Values**: Provide default values for missing data
9. **Cross-Reference Resolution**: Resolve references between entities

#### Reactive Rule Use Cases (Asynchronous Effects)

1. **External Integration**: Interact with external systems and services
2. **Background Processing**: Perform long-running operations without blocking
3. **Event Notifications**: Send notifications when important events occur
4. **Cascading Updates**: Update related entities when source data changes
5. **Data Aggregation**: Collect and process data from multiple sources
6. **Cross-Entity Coordination**: Coordinate changes across multiple entities
7. **Scheduled Tasks**: Perform maintenance or cleanup operations
8. **Audit Logging**: Record transaction history and changes
9. **Complex Workflows**: Implement multi-step processes with dependencies

#### HTTP Gateway Use Cases (External Access)

1. **Web Applications**: Spaces can host complete web applications with both read and write operations
2. **RESTful APIs**: Create public-facing programmable APIs that process HTTP requests
3. **External Data Ingestion**: Process web forms, webhooks, and API calls from external systems
4. **Media Type Transformation**: Publicly accessible endpoints that transform content formats
5. **Static File Hosting**: Serve documents, images, and other static resources
6. **Web Hooks**: Accept notifications from external services
7. **Public Interfaces**: Create public interfaces to privately stored data
8. **Content Delivery**: Deliver content with appropriate content negotiation
9. **Form Processing**: Process and validate form submissions with appropriate feedback

## Security, Privacy, and Cohesion Considerations

1. **Cohesive Sandboxed Execution**: Rules run in isolated environments with:
   - Resource limits (CPU, memory, timeout)
   - No direct file system or network access
   - Access limited to yielding commands
   - Clearly defined input and output boundaries
   - Different execution contexts based on rule type (synchronous for deductive, asynchronous for reactive)
   - Appropriate timeouts based on rule type (shorter for deductive, longer for reactive)

2. **Cohesive Permission Model**:
   - By default, the system enforces authorization for all rule triggering
   - Only space owners can define custom rules
   - Deductive rules have read-only access to the system
   - Reactive rules can perform changes with consistency guarantees
   - When users expose rules via the HTTP Gateway, they explicitly open controlled access from external systems
   - Rules can implement custom access control logic
   - The `/memory` namespace is reserved by the system

3. **Code Security**:
   - Rule generator functions are validated before execution
   - All dependencies must be bundled with the rule implementation
   - Runtime errors are contained within the rule execution environment
   - Commands are explicitly yielded and controlled
   - Reactive operations are transactional and atomic

4. **Cohesive Privacy Protection**:
   - Space owners control what data is exposed
   - Rules can implement access controls for sensitive data
   - System rules enforce permission boundaries
   - Data access is auditable and can be revoked
   - Commands create clear audit trails for all operations
   - Consistency guarantees prevent partial updates
   - Deductive rules have predictable, transactional scope
   - Effectful rules can be monitored across their entire lifecycle


[Irakli Gozalishvili]: https://github.com/gozala
[Common Tools]: https://github.com/wnfs-wg
[memory protocol]:./memory.md
[shebang]:https://en.wikipedia.org/wiki/Shebang_(Unix)
