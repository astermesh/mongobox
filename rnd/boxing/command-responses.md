# MongoDB Command Response Formats

Exact response schemas for every command MongoBox must implement, sourced from official MongoDB documentation and the MongoDB server specification repository.

**Sources:**

- [MongoDB find command](https://www.mongodb.com/docs/manual/reference/command/find/)
- [MongoDB insert command](https://www.mongodb.com/docs/manual/reference/command/insert/)
- [MongoDB update command](https://www.mongodb.com/docs/manual/reference/command/update/)
- [MongoDB delete command](https://www.mongodb.com/docs/manual/reference/command/delete/)
- [MongoDB getMore command](https://www.mongodb.com/docs/manual/reference/command/getmore/)
- [MongoDB aggregate command](https://www.mongodb.com/docs/manual/reference/command/aggregate/)
- [MongoDB findAndModify command](https://www.mongodb.com/docs/manual/reference/command/findandmodify/)
- [MongoDB listDatabases command](https://www.mongodb.com/docs/manual/reference/command/listDatabases/)
- [MongoDB listCollections command](https://www.mongodb.com/docs/manual/reference/command/listcollections/)
- [MongoDB createIndexes command](https://www.mongodb.com/docs/manual/reference/command/createindexes/)
- [MongoDB dropIndexes command](https://www.mongodb.com/docs/manual/reference/command/dropindexes/)
- [MongoDB driver spec: find/getMore/killCursors](https://specifications.readthedocs.io/en/latest/find_getmore_killcursors_commands/find_getmore_killcursors_commands/)
- [MongoDB error codes (server source)](https://github.com/mongodb/mongo/blob/master/src/mongo/base/error_codes.yml)

## 1. find

### Response

```js
{
  cursor: {
    id: NumberLong(0),        // int64 — 0 means cursor exhausted, non-zero means more batches
    ns: "db.collection",      // string — namespace
    firstBatch: [             // array — initial batch of result documents
      { _id: ..., ... }
    ]
  },
  ok: 1                       // 1 success, 0 failure
}
```

### Batch sizing behavior

- Default initial batch: lesser of **101 documents** or **16 MiB**
- Subsequent batches (via getMore): max **16 MiB**
- `batchSize: 0` establishes cursor but returns empty firstBatch
- `singleBatch: true` returns results and closes cursor (id: 0) in one response
- Setting batchSize to 1 or a negative value: server returns that many docs then closes cursor

### Key fields

| Field | Type | Description |
|-------|------|-------------|
| `cursor.id` | int64 | Cursor ID. 0 = exhausted (no more batches) |
| `cursor.ns` | string | Namespace (`database.collection`) |
| `cursor.firstBatch` | array | First batch of documents |
| `ok` | number | 1 = success, 0 = failure |

## 2. insert

### Response

```js
{
  ok: 1,                             // number — 1 success, 0 failure
  n: 3,                              // number — count of documents inserted
  writeErrors: [                     // array — per-document errors (optional, only on error)
    {
      index: 0,                      // number — position in the documents array
      code: 11000,                   // number — error code
      errmsg: "E11000 duplicate key error ...",
      keyPattern: { _id: 1 },        // object — index key pattern that caused error
      keyValue: { _id: "abc" }       // object — duplicate key value
    }
  ],
  writeConcernError: {               // object — write concern error (optional)
    code: 64,
    codeName: "WriteConcernTimeout",
    errmsg: "waiting for replication timed out",
    errInfo: { wtimeout: true }
  }
}
```

### Key fields

| Field | Type | Description |
|-------|------|-------------|
| `ok` | number | 1 success, 0 failure |
| `n` | number | Number of documents successfully inserted |
| `writeErrors` | array | Per-document errors; each has `index`, `code`, `errmsg` |
| `writeErrors[].index` | number | Index of failed doc in the input array |
| `writeErrors[].code` | number | MongoDB error code |
| `writeErrors[].errmsg` | string | Human-readable error message |
| `writeErrors[].keyPattern` | object | Index pattern causing duplicate key (for code 11000) |
| `writeErrors[].keyValue` | object | Actual duplicate key value (for code 11000) |
| `writeConcernError` | object | Write concern failure details |

### Notes

- `ok: 1` even when writeErrors exist (partial success with ordered: false)
- With `ordered: true` (default), execution stops at first error; remaining docs not inserted
- With `ordered: false`, all docs attempted; all errors collected in writeErrors
- `n` reflects only successfully inserted documents

## 3. update

### Response

```js
{
  ok: 1,
  n: 5,                               // number — documents matched
  nModified: 3,                        // number — documents actually modified
  upserted: [                          // array — present only when upserts occurred
    {
      index: 0,                        // number — position in updates array
      _id: ObjectId("507f1f77...")     // value — _id of newly inserted document
    }
  ],
  writeErrors: [
    {
      index: 0,
      code: 11000,
      errmsg: "E11000 duplicate key error ...",
      op: { q: {...}, u: {...}, ... }  // object — the failing update statement
    }
  ],
  writeConcernError: { ... }           // same format as insert
}
```

### Key fields

| Field | Type | Description |
|-------|------|-------------|
| `ok` | number | 1 success, 0 failure |
| `n` | number | Number of documents matched by all update statements combined |
| `nModified` | number | Number of documents actually modified (excludes matches where values unchanged) |
| `upserted` | array | Documents created by upsert; each has `index` and `_id` |
| `upserted[].index` | number | Index of the update statement in the request array |
| `upserted[].\_id` | any | `_id` of the upserted document |
| `writeErrors` | array | Per-statement errors |

### Notes

- `n` counts matched docs, `nModified` counts actually changed docs
- If an update matches a doc but values are identical, `n` increments but `nModified` does not
- `upserted` array only present when at least one upsert occurred
- Each `upserted` entry links back to the update statement via `index`

## 4. delete

### Response

```js
{
  ok: 1,
  n: 5,                               // number — documents deleted
  writeErrors: [
    {
      index: 0,                        // number — position in deletes array
      code: 2,
      errmsg: "...",
    }
  ],
  writeConcernError: { ... }           // same format as insert
}
```

### Key fields

| Field | Type | Description |
|-------|------|-------------|
| `ok` | number | 1 success, 0 failure |
| `n` | number | Total number of documents deleted |
| `writeErrors` | array | Per-statement errors |
| `writeConcernError` | object | Write concern failure |

## 5. getMore

### Response

```js
{
  cursor: {
    id: NumberLong(12345),             // int64 — 0 when exhausted
    ns: "db.collection",
    nextBatch: [                       // array — note: nextBatch, NOT firstBatch
      { _id: ..., ... }
    ]
  },
  ok: 1
}
```

### Key fields

| Field | Type | Description |
|-------|------|-------------|
| `cursor.id` | int64 | Cursor ID; 0 = cursor exhausted, no more data |
| `cursor.ns` | string | Namespace |
| `cursor.nextBatch` | array | Next batch of documents (not `firstBatch`) |
| `ok` | number | 1 success, 0 failure |

### Cursor exhaustion

- When `cursor.id` is `0`, the cursor is exhausted; client must not send another getMore
- For tailable cursors, an empty `nextBatch` does NOT mean exhaustion (cursor stays open)
- Driver must update its local cursor id and ns from every getMore response

### Request shape (for reference)

```js
{
  getMore: NumberLong(12345),          // int64 — cursor id from find/aggregate
  collection: "collectionName",        // string — collection name (not full ns)
  batchSize: 100                       // optional — max docs to return
}
```

## 6. aggregate

### Response (cursor-based, modern)

```js
{
  cursor: {
    id: NumberLong(0),
    ns: "db.collection",
    firstBatch: [                      // same cursor format as find
      { ... }
    ]
  },
  ok: 1
}
```

### Notes

- The `cursor` option is mandatory (unless using `explain`)
- `cursor: {}` uses default batch size; `cursor: { batchSize: N }` controls first batch
- Default batch size: same as find (101 docs or 16 MiB)
- `cursor: { batchSize: 0 }` returns empty firstBatch (cursor established, no docs)
- Subsequent batches retrieved via getMore (same as find)
- `$out` and `$merge` stages always return cursor.id: 0 with empty firstBatch (results go to collection)

### With explain

When `explain: true`, returns an explain document instead of cursor:

```js
{
  stages: [ ... ],
  serverInfo: { ... },
  ok: 1
}
```

## 7. findAndModify

### Update response (document found)

```js
{
  lastErrorObject: {
    updatedExisting: true,             // boolean — existing doc was updated
    n: 1                               // number — matched count (0 or 1)
  },
  value: { _id: 1, ... },             // document — before or after update (depends on `new` option)
  ok: 1
}
```

### Update response (no document found)

```js
{
  lastErrorObject: {
    updatedExisting: false,
    n: 0
  },
  value: null,                         // null when no document matched
  ok: 1
}
```

### Upsert response (new document created)

```js
{
  lastErrorObject: {
    updatedExisting: false,
    upserted: ObjectId("507f1f77..."), // ObjectId — _id of new document
    n: 1
  },
  value: { _id: ObjectId("507f1f77..."), ... },
  ok: 1
}
```

### Remove response (document found)

```js
{
  lastErrorObject: {
    n: 1                               // note: no updatedExisting for remove
  },
  value: { _id: 1, ... },             // the removed document
  ok: 1
}
```

### Remove response (no document found)

```js
{
  value: null,
  ok: 1
}
```

### Key fields

| Field | Type | Description |
|-------|------|-------------|
| `lastErrorObject.updatedExisting` | boolean | Only present for update/upsert operations; true if existing doc was modified |
| `lastErrorObject.upserted` | ObjectId | Only present when upsert created a new document |
| `lastErrorObject.n` | number | 0 or 1 — number of documents matched |
| `value` | document/null | The document (before or after modification); null if no match |
| `ok` | number | 1 success, 0 failure |

### Behavioral notes

- `new: false` (default): `value` is the document BEFORE modification
- `new: true`: `value` is the document AFTER modification
- For remove: `updatedExisting` field is absent from `lastErrorObject`
- For remove with no match: `lastErrorObject` may be absent entirely; response is `{ value: null, ok: 1 }`

## 8. listDatabases

### Response

```js
{
  databases: [
    {
      name: "admin",                   // string
      sizeOnDisk: 20480,               // number — bytes
      empty: false                     // boolean
    },
    {
      name: "myapp",
      sizeOnDisk: 1048576,
      empty: false
    }
  ],
  totalSize: 1069056,                  // number — bytes, sum of all sizeOnDisk
  totalSizeMb: 1.019,                  // number — totalSize in MiB (not MB)
  ok: 1
}
```

### Key fields

| Field | Type | Description |
|-------|------|-------------|
| `databases` | array | Array of database info objects |
| `databases[].name` | string | Database name |
| `databases[].sizeOnDisk` | number | Size in bytes |
| `databases[].empty` | boolean | True if database has no collections |
| `totalSize` | number | Sum of all sizeOnDisk values (bytes) |
| `totalSizeMb` | number | Total size in MiB |
| `ok` | number | 1 success, 0 failure |

### Notes for MongoBox

- MongoBox is in-memory, so `sizeOnDisk` can report estimated memory usage or 0
- `empty` should reflect whether the database has any collections
- Response is NOT cursor-based (direct array)

## 9. listCollections

### Response

```js
{
  cursor: {
    id: NumberLong(0),
    ns: "mydb.$cmd.listCollections",   // note: special namespace format
    firstBatch: [
      {
        name: "users",                 // string — collection name
        type: "collection",            // string — "collection" or "view"
        options: {},                    // object — collection options (capped, size, etc.)
        info: {
          readOnly: false,             // boolean
          uuid: UUID("...")            // UUID — collection uuid
        },
        idIndex: {                     // object — spec of the _id index
          v: 2,
          key: { _id: 1 },
          name: "_id_"
        }
      }
    ]
  },
  ok: 1
}
```

### Key fields per collection document

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Collection or view name |
| `type` | string | `"collection"` or `"view"` |
| `options` | object | Creation options (empty object for defaults) |
| `info.readOnly` | boolean | Whether collection is read-only |
| `info.uuid` | UUID | Collection UUID |
| `idIndex` | object | `_id` index specification (absent for views) |

### Notes

- Uses cursor-based response (same pagination via getMore as find)
- Namespace format for cursor.ns is `"<db>.$cmd.listCollections"`
- Views have `type: "view"` and no `idIndex` field

## 10. createIndexes

### Response

```js
{
  createdCollectionAutomatically: false, // boolean — true if collection was created
  numIndexesBefore: 1,                   // number — index count before command
  numIndexesAfter: 2,                    // number — index count after command
  ok: 1
}
```

### When index already exists

```js
{
  numIndexesBefore: 2,
  numIndexesAfter: 2,
  note: "all indexes already exist",     // string — present when no new indexes created
  ok: 1
}
```

### Key fields

| Field | Type | Description |
|-------|------|-------------|
| `createdCollectionAutomatically` | boolean | True if collection didn't exist and was created |
| `numIndexesBefore` | number | Index count before operation |
| `numIndexesAfter` | number | Index count after operation |
| `note` | string | Present when indexes already exist |
| `ok` | number | 1 success, 0 failure |

### Error response (conflicting index)

```js
{
  ok: 0,
  errmsg: "Index with name: myIndex already exists with different options",
  code: 85,
  codeName: "IndexOptionsConflict"
}
```

## 11. dropIndexes

### Drop specific index response

```js
{
  nIndexesWas: 3,                        // number — index count before dropping
  ok: 1
}
```

### Drop all indexes (except _id) response

```js
{
  nIndexesWas: 3,
  msg: "non-_id indexes dropped for collection",
  ok: 1
}
```

### Error: index not found

```js
{
  ok: 0,
  errmsg: "index not found with name [badName]",
  code: 27,
  codeName: "IndexNotFound"
}
```

### Key fields

| Field | Type | Description |
|-------|------|-------------|
| `nIndexesWas` | number | Number of indexes before the drop |
| `msg` | string | Present when dropping all non-_id indexes |
| `ok` | number | 1 success, 0 failure |

## 12. killCursors

### Response

```js
{
  cursorsKilled: [ NumberLong(12345) ],    // array — successfully killed
  cursorsNotFound: [ NumberLong(67890) ],  // array — already dead / not found
  cursorsAlive: [],                        // array — could not be killed (ignore)
  cursorsUnknown: [],                      // array — unknown cursors
  ok: 1
}
```

## 13. Error format (general command failure)

All commands use the same error envelope when the command itself fails:

```js
{
  ok: 0,
  errmsg: "human-readable error message",
  code: 59,                                // number — MongoDB error code
  codeName: "CommandNotFound"              // string — symbolic name
}
```

## 14. writeConcernError format

Appears inside write command responses (insert/update/delete) alongside `ok: 1`:

```js
{
  ok: 1,
  n: 3,
  writeConcernError: {
    code: 64,
    codeName: "WriteConcernTimeout",
    errmsg: "waiting for replication timed out",
    errInfo: {
      wtimeout: true,
      writeConcern: {
        w: 2,
        wtimeout: 5000,
        provenance: "clientSupplied"
      }
    }
  }
}
```

### Notes for MongoBox

- MongoBox is single-node in-memory; write concern errors should never occur
- For parity, MongoBox should accept write concern options and silently succeed (`w: 1` behavior)
- If a client sends an impossible write concern (e.g., `w: 2`), MongoBox may need to decide: error or ignore

## 15. Key error codes

Error codes from [error_codes.yml](https://github.com/mongodb/mongo/blob/master/src/mongo/base/error_codes.yml) that MongoBox must implement:

| Code | Name | When used |
|------|------|-----------|
| 2 | BadValue | Invalid argument values |
| 13 | Unauthorized | Auth failure (if auth is implemented) |
| 14 | TypeMismatch | Wrong BSON type for a field |
| 26 | NamespaceNotFound | Collection or database does not exist |
| 27 | IndexNotFound | Dropping a non-existent index |
| 43 | CursorNotFound | getMore with expired/invalid cursor ID |
| 48 | NamespaceExists | Creating a collection that already exists |
| 59 | CommandNotFound | Unrecognized command name |
| 66 | ImmutableField | Trying to modify `_id` field |
| 73 | InvalidNamespace | Malformed namespace string |
| 85 | IndexOptionsConflict | Creating index with same name but different options |
| 86 | IndexKeySpecsConflict | Creating index with same key pattern but different name |
| 112 | WriteConflict | Concurrent write conflict |
| 11000 | DuplicateKey | Unique index violation |
| 13297 | DatabaseDifferCase | Database name differs only in case |

### Priority for MVP

**Must-have (P0):**

- 11000 — DuplicateKey (unique index enforcement)
- 26 — NamespaceNotFound (operating on non-existent collection)
- 59 — CommandNotFound (unknown command)
- 43 — CursorNotFound (invalid getMore)
- 2 — BadValue (input validation)
- 48 — NamespaceExists (duplicate collection creation)

**Should-have (P1):**

- 66 — ImmutableField (`_id` modification guard)
- 73 — InvalidNamespace (validation)
- 27 — IndexNotFound
- 85 — IndexOptionsConflict
- 14 — TypeMismatch

**Nice-to-have (P2):**

- 86 — IndexKeySpecsConflict
- 112 — WriteConflict (MongoBox is single-threaded, unlikely)
- 13 — Unauthorized (auth not in MVP scope)
- 13297 — DatabaseDifferCase

## Assessment: G2 gap status

**Conclusion: G2 can be fully closed with publicly available documentation.**

All response schemas are thoroughly documented in:

1. Official MongoDB manual (`docs.mongodb.com/manual/reference/command/`)
2. MongoDB driver specifications (`specifications.readthedocs.io`)
3. MongoDB server source (`github.com/mongodb/mongo`)

Every field, type, and behavioral edge case needed for MongoBox implementation is available. No proprietary or undocumented behavior needs to be reverse-engineered.

---

[← Back](README.md)
