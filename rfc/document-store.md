# Common store

> Attempt to capture JSON store described by seefeld

## Overview

This will be a simple JSON store for a catalog of JSON documents. Clients will have a websocket based API to read/write/update documents and watch for changes at **document level granularity**. In nutshell it's similar to filesystem but only supported file types are JSON.

## API

This is a description of the websocket based API.

```ts
/**
 * Command can describe set of operations to be performed. When submitting a
 * more then one operation all operations are performed before response is
 * sent back.
 * 
 * ⚠️ No transactional gurantees, meaning some operations may succeed and some
 * may fail. Response will contain results for each operation under the same
 * keys.
 *
 * ⚠️ Execution order is non-deterministic. If you need order send one command
 * and then send another.
 */
type Command = {
 pull?: Record<string, Push>
 push?: Record<string, Pull>
 watch?: Record<string, Watch>
 unwatch?: Record<string, Watch>
}


/**
 * Database identifier. It is RECOMMENDED to
 * use `did:key` identifiers or some other
 * format for representing public keys.
 */
type DatabaseID = string

/**
 * Document identifier. It is RECOMMENDED to
 * use unique identifiers either derived
 * like merkle reference or random like UUID.
 */
type DocumentID = string



/**
 * Updates document of the specified database to the `is` value.
 * If `cause` is specified ensures that document has not being modified. 
 */
type Push = {
  the: DocumentID
  of: DatabaseID
  
  is: JSON
  
  cause?: Checksum
}


/**
 * Document checksum. Store implementation can choose whatever
 * way it wishes to derive canonical checksum for the document.
 */ 
type Checksum = string

/**
 * Pulls content from the JSON document. If `cause` is specified
 * pull may return no data when chesum of the document matches it.
 */
type Pull = {
  the: DocumentID
  of: DatabaseID
  
  cause?: Checksum
}

/**
 * Registeres watcher for the document.
 */
type Subscribe = {
  the: DocumentID
  of: DatabaseID
}

type Status = {
  the: DocumentID
  of: DatabaseID
  cause: Checksum
}



/**
 * Describes response returned for the command.
 */
type Receipt<For extends Command> = {
  pull: { [Key in keyof For['pull']]: Result<Required<Push>|Status, PullError> }
  push: { [Key in keyof For['push']]: Result<Status, PushError> }
  watch: { [Key in keyof For['watch']]: Result<Required<Push>, WatchError> }
  unwatch: { [Key in keyof For['unwatch']]: Result<Status, UnwatchError> }
}
```

### Command

Command describes batch of operations client can request service to perform. It is important to call out that batches are not executed with any transactional gurantees, implying that some operations in the command may succeed while others fail. Furthermore, opeartion execution order is non-deterministic, therefor if order of operation matters client MUST take execute them in a separate commands.

> For example if document `A` has some pointer to the content in the document `B`, client would want to update pointer in the `A` document before deleting content `A` pointed in the document `B`. If both operations were send in a single command, race may occur and there may be a time frame in which `A` is pointing to the content in document `B` that does not exist.

#### Operations

Command MAY contain `pull`, `push`, `watch` and `unwatch` operation sets, represented by dictionaries e.g.

```js
const response = client.request({
  pull: {
    profile: {
      the: "4301a667-5388-4477-ba08-d2e6b51a62a3",
      of: "did:key:z6Mkk89bC3JrVqKie71YEcc5M1SMVxuCgNx6zLZ8SYJsxALi"
    },
    recepies: {
      the: "4301a667-5388-4477-ba08-d2e6b51a62a3",
      of: "did:key:z6Mkk89bC3JrVqKie71YEcc5M1SMVxuCgNx6zLZ8SYJsxALi",
      cause: "ba4jcb2tb7gb5llskpwvdfxckvvkmrsl4lfooktkfd4thucfgawd7ghqp"
    }
  },
  push: {
    status: {
      the: "8c642d0d-9c67-4925-9fe5-405486e7dea7",
      of: "did:key:z6Mkk89bC3JrVqKie71YEcc5M1SMVxuCgNx6zLZ8SYJsxALi",
      is: { online: Date.now() },
    },
    description: {
      the: "3c8b0faf-cfb7-4537-a040-78bb48cfb962",
      of: "did:key:z6Mkk89bC3JrVqKie71YEcc5M1SMVxuCgNx6zLZ8SYJsxALi",
      is: { "description": "Chief Witch" },
      cause: "ba4jcbmzammd3t4cwbpfdxjbo4st4k4azmaozlhtklv6rxkppuxal3ywe"
    }
  },
})
```

Keys in of the operation sets are arbitrary and are used for mapping operation results in the response.

```js
assert.deepEqual(response, {
  ok: {
    pull: {
      profile: {
        ok: {
          the: "4301a667-5388-4477-ba08-d2e6b51a62a3",
          of: "did:key:z6Mkk89bC3JrVqKie71YEcc5M1SMVxuCgNx6zLZ8SYJsxALi",
          is: { value: { name: "Alice" } },
          cause: "ba4jcbgvygxoujlhz5a44cfyxtnrb6a6zxby2mdjhbthee7jvkcijdmlg"
        }
      },
      author: {
        error: {
          name: 'Not Found',
          message: 'Document "4301a667-5388-4477-ba08-d2e6b51a62a3" not found in "did:key:z6Mkk89bC3JrVqKie71YEcc5M1SMVxuCgNx6zLZ8SYJsxALi"'
        }
      }
    },
    push: {
      status: {
        ok: {
          the: "8c642d0d-9c67-4925-9fe5-405486e7dea7",
          of: "did:key:z6Mkk89bC3JrVqKie71YEcc5M1SMVxuCgNx6zLZ8SYJsxALi",
          cause: "ba4jcb3cxb2lvygdel7ljg4akgvppv6lrblhznsclcruz2m7ya75p4cvn"
        }
      },
      description: {
        error: {
          name: "PushError",
          the: "3c8b0faf-cfb7-4537-a040-78bb48cfb962",
          of: "did:key:z6Mkk89bC3JrVqKie71YEcc5M1SMVxuCgNx6zLZ8SYJsxALi",
          is: { value: { online: 1737492200331 } },
          
          expected: "ba4jcbmzammd3t4cwbpfdxjbo4st4k4azmaozlhtklv6rxkppuxal3ywe"
          actual: "ba4jcbpionvcxjrzkl6lkwp5vtaanzwdnh5ydgrnylhd3cmjuemxr2owh",
          message: "Target invariant missmatch, please rebase your changes"
        }
      }
    }
  }
})
```


### Pull

Pulls latest document from the store. If `cause` checksum is provided and state of the document in the store matches it response will not include the `is` field. If `cause` is omitted or it is different from the stored document result will include `is` with the latest document state.

Attempt to pull document will fail if either pulled document or database does not exist.

### Push

Pushes changed document into a store into a given database. If document does not exist new document will be created. If database does not exist new database will be created.

If `cause` is specified it is used to enforce the invariant, meaning update will fail if the target checksum does not match specified `cause`.

If `cause` is omitted document will be overwritten or new document will be created.


> ℹ️ To ensure that new document is create merkle-reference of empty object `{}` `ba4jcbckylwzunx663kx5t7lgkjmgglyhz4fet3pge7dl7wmg677jlyfm` could be specified.


### Watch

Watch operation can be used in order to subscribe to document changes. When subscribed store will return `Required<Push>` for the current document state and will send subsequent `Required<Push>` objects every time document is updated.

Duration of watch is session bound, meaning that after client disconnects all the watches will be dropped and it's up to client to set them up again.

> ⚠️ Only one unique document watch per session is allowed, meaning client can watch several documents but issuing watch to the same document multiple times will be considered noop. That is to say it's up to cient to deal with multiple customers it may have locally for the same document.

Watches are not persisted across sessions, meaning once cliend disconnects all the watches are dropped.

#### unwatch

Stops watching a document for changes. Once document is unwatched no future push notifications will be send by the store for that document.

## Persistence

JSON documents will be persisted in SQLite. It will be [stored as ordinary text](https://arc.net/l/quote/vsmijstd) and all read/write operations will use [SQLites builtin JSON functionality](https://www.sqlite.org/json1.html).


