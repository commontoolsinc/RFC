# Common store

> Attempt to capture JSON store described by seefeld

## Overview

This will be a simple JSON store for a catalog of JSON documents. Clients will have an websocket based API to read/write/update content and to watch for changes in the document. In nutshell it's similar to filesystem but only file types supported are JSON.

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
 write?: Record<string, Write>
 read?: Record<string, Read>
 delete?: Record<string, Delete>
 watch?: Record<string, Watch>
 unwatch?: Record<string, Watch>
}


// Paths requires mandator `document` component
// that uniquely identifies document. Rest of
// the components correspond the object path
// within the document
type Path = [
  document: DocumentID,
  target: ...Key[],
  element: Key | Range
]

// Some unique identifier for the document
type DocumentID = string
// Path component in the document
type Key = string|number

/**
 * Range describes array element range,
 * where `from` is inclusive and `to` is
 * exclusive of `to`
 */
type Range = [from:number, to:number]


/**
 * Writes piece of JSON at given path. For new document only
 * the `document` fragment is expected. If the target path
 * in the document or a document does not exists write will error.
 */
type Write = {
  path: Path
  data: JSON
}

/**
 * Reads content from the JSON at a given path. If document or
 * path does not exist read will return an error. 
 */
type Read = {
  path: Path
}

/**
 * Deletes content from the JSON a a given path. If path only
 * contains document fragment deletes the whole document. If
 * document or path does not exists wirr return an error.
 */
type Delete {
  path: Path
}

/**
 * Observes content from the JSON at the given path. If path or document
 * does not exists first notification will contain no `data`.
 */
type Watch = {
  path: Path
}


type WatchNotification = {
  path: Path
  // If `data` is omitted document fragment does not exists
  // in the document.
  data?: JSON
}


/**
 * Describes response returned for the command.
 */
type Receipt<For extends Command> = {
  read: { [Key in keyof For['read']]: Result<{ data: JSON }, ReadError> }
  write: { [Key in keyof For['write']]: Result<{}, WriteError> }
  delete: { [Key in keyof For['delete']]: Result<{}, DeleteError> }
  watch: { [Key in keyof For['watch']]: Result<{}, WatchError> }
  unwatch: { [Key in keyof For['unwatch']]: Result<{}, UnwatchError> }
}
```

### Command

Command describes batch of operations client can request service to perform. It is important to call out that batches are not executed with any transactional gurantees, implying that some operations from the command may succeed while others may fail. Furthermore, order of opeartion execution is non-deterministic, meaning service may execute them in any order, therefor when order of operation matters client should take care of executing such operations as separate commands. E.g. if document `A` has some pointer to content in document `B`, client would want to delete or update pointer as first command and once that succeeds only after delete document `B`. If both operations were send in a single command, race may occur and there may be a time frame where `A` is pointing to document that has being deleted.

### Operations

Command MAY contain `read`, `write`, `delete`, `watch` and `unwatch` operation sets represented by dictionaries.

```js
const response = client.request({
  read: {
    title: {
      path: ["4301a667-5388-4477-ba08-d2e6b51a62a3", "title"]
    },
    author: [
      path: ["4301a667-5388-4477-ba08-d2e6b51a62a3", "author"]
    ]
  },
  delete: {
    obsolete: {
      path: ["0f30d02f-4442-4bb1-a9db-d75077d5735f"]
    }
  },
})
```

Keys in the dictionary are arbitrary and are used for mapping corresponding results in the response

```js
assert.deepEqual(response, {
  ok: {
    read: {
      title: { ok: 'Untitled' },
      author: {
        error: {
          name: 'Not Found',
          message: 'Document 4301a667-5388-4477-ba08-d2e6b51a62a3 has no `.author` field'
        }
      }
    },
    delete: {
      obsolete: {
        error: {
          name: "Not Found",
          message: "Document 0f30d02f-4442-4bb1-a9db-d75077d5735f not found"
        }
      }
    }
  }
})
```

#### Read

Reads data at the requested path. If document or data leading path does not exists operation will fail.

#### Write

Write can be used both to create a new document and to update existing documents.

If path only containts document identifier and such document does not exists operations acts as a create. In all other cases operation updates existing document.

Write will error when update path is invalid for example following opeartion will fail

```js
client.request({
  write: {
    path: [
      "4301a667-5388-4477-ba08-d2e6b51a62a3",
      "user",
      "name"
    ],
    data: "Alice"
  }
})
```

If target document looks like

```json
{
  "user": "Bob"
}
```

Becuse `.user` is a string and not an object. Please note that writes will fail if target is a `null` or an `array` despite both being `"object"` per `typeof` in JS.

Above operation could be rewritten as follows to avoid an error

```js
client.request({
  write: {
    path: [
      "4301a667-5388-4477-ba08-d2e6b51a62a3",
      "user",
    ],
    data: { name: "Alice" }
  }
})
```

By definition operation also fails if `undefined` is incountered in the update path, implying that path is not going to be created on demand. That is to ensure that client updates are intentional as opposed to incidental.

##### Arrays

Arrays can be updated just like objects, except last path component is expected to be of number type.

Write corresponding to `target.length` will add new element to the array, while writes where last path component is `> target.length` will error as opposed to creating a sparse array.

It is also possible to use a `Range` in order to splice data into the target array. Exciding array length with an `range.to` is allowed but not with a `range.from`.

#### delete

Delete acts pretty much like `write` if `data` was `undefined`, meaning same errors would occur as with `write`.

> ℹ️ Given that `undefined` is not a valid JSON value `delete` does not actualy set `undefined` instead it deletes property in case of objects and deletes target range from the arrays.


#### watch

Watch operation can be used in order to observe changes to a target in the document. If document or path does not exist first watch notification will contain no `data` and next notification will be send when some data is set under a watched path.

Duration of watch is session bound, meaning that after client disconnects all the watches will be dropped and it's up to client to set them up again.

#### unwatch

Drops watched paths so no notification will be send for the given updates.

> ⚠️ Please noteth that unwatching parent path does not unwatch child paths e.g. if you had watches for
> 
> ```
> ["4301a667-5388-4477-ba08-d2e6b51a62a3", "user", "name"]
> ["4301a667-5388-4477-ba08-d2e6b51a62a3", "user"]
> ```
> Unwatching a second path will not drop the first path.

## Persistence

JSON documents will be persisted in SQLite. It will be [stored as ordinary text](https://arc.net/l/quote/vsmijstd) and all read/write operations will use [SQLites builtin JSON functionality](https://www.sqlite.org/json1.html).


