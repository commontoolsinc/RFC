# Cohesion Protocol

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [Irakli Gozalishvili], [Common Tools]

## Authors

- [Irakli Gozalishvili], [Common Tools]

## Abstract

This document describes the **Cohesion Protocol**, which enables entities in a space to define rules for interpreting facts. The protocol defines two types of rules:

1. **Deductive Rules**: Act as fact pre-processors. They run synchronously with transaction, can query space data, and return derived facts that are included in the same transaction.
2. **Reactive Rules**: Act as fact post-processors. They are scheduled to run asynchronously whith transaction, can query space data, and perform system-provided effects. They return derived facts commited in the follow-up transaction while maintaining consistency guarantees.

The protocol also defines an **HTTP Gateway** mechanism for exposing space over HTTP interface while maintaining appropriate authorization boundaries.

> Enabling the HTTP Gateway requires out-of-band authorization where the space owner delegates the `/memory/transact` capability on `http+` prefixed entities to the gateway service (e.g., `did:web:common.tools`).
>
> This delegation allows the gateway to assert facts on HTTP requests, enabling the space to process web requests through deployed rules. This forms a controlled interface with the outside world while giving space owners full control over which parts of their space are accessible and how they respond to requests.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Core Concepts

### Rules

Rules provide the computational substrate of the space.

#### Rule Definition

A rule is defined by an assertion where:

1. Type (`the` field) is `"application/javascript"`
2. Value (`is` field) contains JavaScript code as an ES module

Rules are added to the space by asserting facts via the [memory protocol].

### Rule Binding

A rule binding is a fact that maps an entity (`of` field) and type (`the` field) to a specific rule. It functions similarly to a setter in programming languages - when a matching fact is asserted or retracted, the bound rule is executed.

A rule is considered bound when the space contains an active fact where:

1. Entity (`of` field) matches the entity of the asserted fact
2. Type (`the` field) matches the type of the asserted fact with a `/` prefix
3. Value (`is` field) is a URI of the rule entity in the space, with an optional `#` fragment
   - If a `#` fragment is present, it specifies which export of the rule module to invoke
   - If no `#` fragment is provided, the `default` export of the rule module is used

### Rule Execution

A rule MUST be executed when a fact matching its binding is asserted or retracted. A valid implementation of the protocol MUST perform the following steps when facts are transacted into memory:

1. Resolve the **rule binding** by finding a fact where:
   - Entity (`of` field) matches the asserted fact's entity
   - Type (`the` field) matches the asserted fact's type with a `/` prefix
   - Value (`is` field) is a URI

2. Resolve bound **rule** by finding a fact where:
   - Entity (`of` field) matches the rule binding's value (`is` field) without any `#` fragment
   - Type (`the` field) is `application/javascript`
   - Value (`is` field) contains ES module source code

3. Invoke the **rule** by:
   - Evaluating the ES module in a sandboxed JS runtime
   - If the rule binding value has a `#` fragment, invoking the named export
   - If the rule binding has no `#` fragment, invoking the `default` export
   - Passing the asserted/retracted fact as an argument
   - Running the returned generator following the appropriate rule execution steps until it returns or throws


### Rule Execution Steps

#### Deductive Rule Execution Steps

Deductive rules MUST be executed **synchronously** within the triggering transaction. When a rule yields:

- `{ memory/query: args }` - The runtime MUST perform a `/memory/query` invocation as defined by [memory protocol] using the provided `args`. Execution MUST resume with the query result.

- Any other command - The runtime MUST resume execution with an exception, allowing the rule to `catch` and continue execution if desired.

If the rule throws an exception, the transaction MUST fail with the thrown error.

When a rule returns a value, it MUST be interpreted as a collection of derived facts. All these derived facts MUST be included in the current transaction. A derived fact with the same entity and attribute as the triggering fact MUST NOT cause a recursive rule call - instead, it should be asserted as a fact.



#### Reactive Rule Execution Steps

Reactive rules MUST be executed **asynchronously** concurrently with a triggering transaction. When a rule yields:

- `{ memory/query: args }` - The runtime MUST perform a `/memory/query` invocation as defined by [memory protocol] using the provided `args`. Execution MUST resume with the query result.

- Other commands - Unlike deductive rules, reactive rules MAY yield other commands, including effectful ones provided by the system. The runtime MUST perform these commands using corresponding [UCAN Invocation] where the command key becomes `cmd` and the value becomes `args`. The runtime MAY restrict which commands reactive rules can access.

> ⚠️ It is RECOMMENDED that runtimes do not provide commands to transact changes in the space, as they can violate consistency guarantees.

Reactive rules MUST be executed with **optimistic concurrency**:

1. The runtime MUST record all facts queried during execution
2. The returned value MUST be interpreted as a collection of derived facts
3. Before committing, if any queried facts have changed, rule execution MUST be retried with exponential backoff
4. If the maximum retry limit is reached, the runtime SHOULD block transactions for facts queried by the rule to guarantee consistency of derived facts
5. Derived facts MUST be transacted in a separate transaction

This approach ensures **causal consistency** - changes will be applied correctly once all dependencies are stable.


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
}
```

### HTTP Gateway

The HTTP Gateway is a service that maps HTTP requests to rules bound to special `http+`-prefixed entities. It acts using authorization delegated by the space owner, creating a bridge between the Common Fabric and external web systems.

This gateway enables spaces to:

1. Expose HTTP interfaces with custom request handling
2. Process and transform web inputs using rules
3. Access and manipulate space data through standard query mechanisms
4. Return HTTP responses to external clients
5. Bridge between the Common Fabric and external web services

To enable the HTTP Gateway, a space owner must delegate to the gateway service (typically `did:web:common.tools`) the capability to invoke `/memory/transact` on `http+`-prefixed entities. This delegation allows the gateway to assert facts representing HTTP requests, while maintaining proper authorization boundaries.

Space owners retain full control over which parts of their space are exposed and how requests are processed. The HTTP Gateway operates on specially designated entities with the `http+` prefix, as shown in the example below:

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
            "is": "rule:8b3a#echo"
          }
        },
      "http+note:5d59a2ff": {
        "/text/plain": {
          "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
            "is": "rule:8b3a#summary"
          }
        }
      },
      "rule:8b3a": {
        "application/javascript": {
          "ba4jca7rv4dlr5n5uuvcz7iht5omeukavhzbbpmc5w4hcp6dl4y5sfkp5": {
            "is": `
              export echo* ({ the, of, is: request }) {
                // select request body text
                const text = yield {
                  'memory/select': {[request]: {'body/text': {}} }
                }

                return [{
                  the,
                  of,
                  is: new Response('>>' + text, {status: 200})
                }]
              }

              export summary* ({ the, of }) {
                cost entity = of.slice('http+'.length)
                const source = yield { 'memory/select': { [entity]: { [the]: {} }} }
                // read value of the fact
                const [[{is}]] = Object.entrise(source?.[entity]?.[the] ?? {})
                const summary = yield {
                  'llm/prompt': { prompt: 'summarize ' + is }
                }

                return [{
                  the,
                  of,
                  is: new Response(summary, { status: 200 })
                }]
              }`
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

The gateway maps URLs to entities following these rules:

- **Entity-specific Routes**: `/did:key:space/note:5d59a2ff/path/to` maps to the `http+note:5d59a2ff` entity
- **Space-level Routes**: `/did:key:space/whatever/else` maps to the `http+did:key:space` entity

The gateway determines the target entity by checking if the path component after the DID contains a `:` character. If it does, the gateway asserts a fact on that specific entity. Otherwise, it asserts a fact on the space entity itself.

##### Content-Type to Type Mapping

The HTTP request's Content-Type header determines the type (`the` field) of the asserted fact:
- The Content-Type is normalized to lowercase
- This normalized value becomes the fact's type

##### Request Processing Flow

When an HTTP request arrives at the gateway, it processes the request as follows:

1. It identifies the target entity from the URL path using the mapping rules above and sets this as the fact's entity (`of` field)
2. It normalizes the Content-Type header of the request and sets this as the fact's type (`the` field)
3. It creates a unique temporary URI to represent the HTTP request and sets this as the fact's value (`is` field)

The gateway only asserts this derived fact if the space contains a rule binding that matches it. If no matching rule binding is found, the fact is not asserted and a 404 - Not Found response is returned. If a matching rule binding is found, the request is blocked until the triggered rule processes the request and provides a response.

### Rationale

#### Generator Pattern

The generator pattern used by rules offers several advantages over regular functions:

1. **Suspension and Resumption**: Allows execution to be suspended at yield points and resumed when results are available. Generators can even be rerun in a new session by replaying the results from the previous run.

2. **Transparent Effect Tracking**: All effects are explicitly yielded as data, making them visible, auditable, and controllable.


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
  - Run in a separate transaction context

These analogies can help developers understand how and when to use each type of rule.


## Security and Privacy Considerations

1. **Sandboxed Execution**: Rules run in isolated environments with:
   - Resource limits (CPU, memory, execution time)
   - No direct file system or network access
   - No `await` or promises (only generator-based concurrency)
   - Access limited to explicitly yielded commands
   - Clearly defined input and output boundaries
   - Different execution contexts for deductive rules (synchronous) and reactive rules (asynchronous)

2. **Permission Model**:
   - The system enforces authorization for all rule triggering by default
   - Only space owners or explicitly delegated entities can bind rules
   - Deductive rules are limited to read-only access to the space
   - Reactive rules can perform effects while maintaining consistency guarantees
   - HTTP Gateway access is controlled through explicit delegation

3. **Code Security**:
   - Rule generator functions are validated before execution
   - All dependencies must be bundled with the rule implementation
   - Runtime errors are contained within the rule execution environment
   - All effects are explicitly yielded as commands and can be monitored
   - Reactive operations are transactional and provide atomic guarantees
   - Consistency checking prevents operations based on stale data


[Irakli Gozalishvili]: https://github.com/gozala
[Common Tools]: https://github.com/wnfs-wg
[memory protocol]:./memory.md
[shebang]:https://en.wikipedia.org/wiki/Shebang_(Unix)
