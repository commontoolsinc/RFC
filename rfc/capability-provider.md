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
3. Access and manipulate data by yielding queries or effectful commands
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

Routes are mapped to entities as follows:
- **Entity in Space**: `/did:key:space/note:5d59a2ff/...` maps to the `http+note:5d59a2ff` entity
- **Space itself**: `/did:key:space/whatever/else` maps to the `http+did:key:space` entity

The gateway asserts fact on the entity if path component after DID contains `:` character. Otherwise gateway asserts fact of the space entity.

##### Content-Type to Type Mapping

The HTTP request's Content-Type header determines type (`the` field) of the fact.
- The Content-Type is normalized to lowercase


##### Request Processing Flow

When an HTTP request arrives at the gateway to derives corresponding fact as follows:

1. Identifies target entity from the URL path (using mapping rules above) and sets it to the fact entity (`of` field).
2. It normalizes `content-type` header of the request and sets it to the fact type (`the` field).
3. It derives unique temporary URI for the HTTP request and sets it to the fact value (`is` field).

Derived fact is asserted only if space contains rule binding for the derived fact, otherwise fact is not asserted and 404 - Not Found response is returned. If rule binding is found requset is blocked until triggered rule transacts response.

### Rational

#### Generators

This generator pattern offers several advantages over regular functions:

1. **Suspension and Resumption**: Allows execution to be suspended and resumed when results are available. Generator can even be rerun in then new session by replaying results from last run.
1. **Transparent effect tracking**: All side effects are explicitly yielded as data


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
   - Resource limits (CPU, memory, timeout)
   - No direct file system or network access
   - No await or promises
   - Access limited to yielding commands
   - Clearly defined input and output boundaries
   - Different execution contexts based on rule type (synchronous for deductive, asynchronous for reactive)

2. **Permission Model**:
   - By default, the system enforces authorization for all rule triggering
   - Only space owners or delegates can bind rules
   - Deductive rules have read-only access to the space
   - Reactive rules can perform changes with consistency guarantees

3. **Code Security**:
   - Rule generator functions are validated before execution
   - All dependencies must be bundled with the rule implementation
   - Runtime errors are contained within the rule execution environment
   - Commands are explicitly yielded and controlled
   - Reactive operations are transactional and atomic


[Irakli Gozalishvili]: https://github.com/gozala
[Common Tools]: https://github.com/wnfs-wg
[memory protocol]:./memory.md
[shebang]:https://en.wikipedia.org/wiki/Shebang_(Unix)
