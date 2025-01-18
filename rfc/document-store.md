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
type Path = [document: DocumentID, at:...Key[]]

// Some unique identifier for the document
type DocumentID = string
// Path component in the document
type Key = string|number

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
 * Observes content from the JSON at the given path. If document
 * does not exists it will return an error. If path or document
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
  write: { [Key in keyof For['write']]: Result<{}, WriteError> }
  read: { [Key in keyof For['read']]: Result<{ data: JSON }, ReadError> }
  delete: { [Key in keyof For['delete']]: Result<{}, DeleteError> }
  watch: { [Key in keyof For['watch']]: Result<{}, WatchError> }
  unwatch: { [Key in keyof For['unwatch']]: Result<{}, UnwatchError> }
}
```

## Persistence

JSON documents will be persisted in SQLite. It will be [stored as ordinary text](https://arc.net/l/quote/vsmijstd) and all read/write operations will use [SQLites builtin JSON functionality](https://www.sqlite.org/json1.html).


