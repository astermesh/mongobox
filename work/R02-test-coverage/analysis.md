# Test Coverage Analysis

MongoBox currently has zero source code and zero tests — the project is in the research/planning phase. This document identifies all test areas that will be critical as implementation begins, organized by priority.

## Test Strategy Overview

| Layer | Framework | Approach |
|-------|-----------|----------|
| Unit tests | Vitest | Wire protocol parsing, message building, document store |
| Integration tests | Vitest + `mongodb` driver | Full round-trip: driver → TCP → MongoBox → response |
| Parity tests | Vitest + real MongoDB (via `mongodb-memory-server`) | Run same operations against both MongoBox and real MongoDB, compare results |

The **parity test suite** is the most valuable — it directly enforces the core principle that every behavioral difference is a bug.

## Priority 1: Wire Protocol Layer

Foundation everything else depends on.

### Message Framing
- TCP stream parsing with length-prefixed messages
- Partial reads (message split across multiple TCP chunks)
- Multiple messages in a single chunk
- Messages exactly at boundary conditions

### OP_MSG Parsing/Serialization
- Kind 0 (body) section — single BSON document
- Kind 1 (document sequence) sections — batch inserts, updates, deletes
- Flag bits: checksumPresent, moreToCome, exhaustAllowed
- Round-trip fidelity (parse → serialize → parse = identical)
- Unknown payload types must close connection
- Unknown required flag bits (0-15) must error

### OP_QUERY Parsing (Legacy)
- Legacy handshake path (`admin.$cmd` with `isMaster`)
- Flag bits parsing
- cstring parsing for `fullCollectionName`

### OP_REPLY Serialization
- Response to OP_QUERY with correct flags, cursorID, numberReturned

### OP_COMPRESSED
- Decompression with noop/snappy/zlib/zstd
- Compression negotiation in handshake
- Restricted commands must not be compressed

### Edge Cases
- Messages at max size (48MB)
- Empty sections
- Malformed messages (truncated, invalid BSON, wrong length)

## Priority 2: Handshake & Connection

### `hello` Command
- Returns correct standalone topology fields
- Correct capabilities (maxBsonObjectSize, maxMessageSizeBytes, maxWriteBatchSize)
- Correct wire version (minWireVersion, maxWireVersion)
- `helloOk: true` in response

### `isMaster` Legacy
- Via OP_MSG
- Via OP_QUERY (legacy drivers)
- `helloOk` negotiation flow

### `ping`
- Returns `{ok: 1}`

### Connection Handling
- Connection ID uniqueness
- Multiple concurrent connections
- Client metadata validation (512 byte limit, 128 byte app name limit)
- Compression negotiation (intersection of client/server lists)

## Priority 3: CRUD Commands

### `insert`
- Single document insert
- Batch insert (multiple documents)
- Auto-generated `_id` (ObjectId)
- Duplicate key error (code 11000)
- Kind 1 document sequences
- Ordered vs unordered batch behavior

### `find`
- All query operators ($eq, $gt, $gte, $lt, $lte, $in, $nin, $ne)
- Logical operators ($and, $or, $not, $nor)
- Element operators ($exists, $type)
- Array operators ($elemMatch, $size, $all)
- Regex operator ($regex with options)
- Projection (inclusion, exclusion, mixed rules)
- Sort (single field, compound, ascending/descending)
- Skip and limit
- Cursor batching (batchSize)
- Empty filter (return all documents)

### `update`
- Field update operators ($set, $unset, $inc, $mul, $rename, $min, $max)
- Array update operators ($push, $pull, $addToSet, $pop, $pullAll)
- Array modifiers ($each, $slice, $sort, $position)
- Upsert (insert if not found)
- Multi-update
- arrayFilters
- Replacement document (no operators)
- Return value: matchedCount, modifiedCount, upsertedId

### `delete`
- Single delete (limit: 1)
- Multi-delete (limit: 0)
- Filter matching
- Return value: deletedCount

### `getMore`
- Cursor continuation with correct batch sizes
- Cursor expiry / timeout
- Invalid cursor ID error
- Exhausted cursor behavior

### `killCursors`
- Cursor cleanup
- Double-kill (already killed cursor)

## Priority 4: Query Engine / Mingo Integration

This is where the most subtle behavioral parity bugs will live.

### Array Matching Semantics
- Implicit array element matching (`{tags: "a"}` matches `{tags: ["a", "b"]}`)
- $elemMatch vs implicit matching
- Nested arrays
- Array index dot notation (`"arr.0"`)

### Type Coercion
- How MongoDB compares different BSON types
- BSON comparison order (MinKey < Null < Numbers < String < ...)
- Numeric type comparison (Int32 vs Int64 vs Double vs Decimal128)

### Nested Path Queries
- Dotted path notation (`"a.b.c"`)
- Paths through arrays
- Paths through embedded documents

### Null/Undefined Handling
- `{field: null}` matches both null values and missing fields
- `{field: {$exists: false}}` vs `{field: null}`
- `{field: {$type: "null"}}` matches only explicit null

### Comparison Edge Cases
- $gt/$lt with mixed types
- MinKey/MaxKey behavior
- NaN handling
- Empty string vs missing field

## Priority 5: Document Store

### Namespace Management
- Database isolation
- Collection isolation within databases
- Creating collections implicitly on first insert
- `listDatabases`, `listCollections`
- `create`, `drop` (collection and database)

### `_id` Management
- Auto ObjectId generation when `_id` omitted
- Custom `_id` values (any BSON type)
- Uniqueness enforcement across `_id`

### Document Size Limits
- 16MB max BSON document size
- Correct error when exceeded

## Priority 6: Error Handling Parity

Per the core principle, error responses must exactly match real MongoDB.

### Error Codes
- Every error returns the exact MongoDB error code
- Error code names (codeName) must match

### Error Messages
- Format must match MongoDB's error message patterns

### Write Errors
- `writeErrors` array format for bulk operations
- Ordered: stop on first error
- Unordered: continue and collect all errors
- `writeConcernError` structure

### Command Errors
- Unknown command: correct error code and message
- Missing required fields
- Invalid field types
- Invalid field values

## Priority 7: Aggregation Pipeline

### Common Stages
- $match, $group, $sort, $project, $unwind
- $lookup (join), $limit, $skip
- $addFields, $set (aliases)
- $count, $out, $merge

### Expression Evaluation
- Arithmetic: $add, $subtract, $multiply, $divide
- Conditional: $cond, $ifNull, $switch
- String: $concat, $substr, $toLower, $toUpper
- Array: $arrayElemAt, $filter, $map, $reduce

### Cursor-Based Results
- Aggregation returns cursor (not direct array)
- batchSize handling
- getMore for large result sets

### Edge Cases
- Empty pipeline (returns all documents)
- Empty collection
- Pipeline stages that produce no output

## Priority 8: Integration / Driver Compatibility

### Node.js Driver
- Connect, full CRUD cycle, disconnect using official `mongodb` package
- Connection string parsing
- Connection pooling
- Auto-reconnect behavior

### Cross-Driver Testing (Tier 3)
- Python (pymongo)
- Java driver
- Go driver
- Each may use slightly different protocol features

## Priority 9: Authentication (Tier 3)

### SCRAM-SHA-256
- Full 4-message exchange
- Correct SaltedPassword derivation (PBKDF2)
- Mutual authentication (client verifies server)
- Speculative authentication (merged with handshake)

### SCRAM-SHA-1
- Fallback mechanism
- Correct iteration count (10000)

### Mechanism Negotiation
- `saslSupportedMechs` in hello response

## Priority 10: Transactions (Tier 3)

### Session Lifecycle
- `startTransaction`, `commitTransaction`, `abortTransaction`
- Session ID tracking (lsid)

### Isolation
- Uncommitted writes not visible to other sessions
- Read-your-own-writes within a transaction

### Rollback
- Aborted transactions discard all changes
- No partial commit

## Recommended Implementation Order

1. **Start with parity test infrastructure** — Set up the framework to run identical operations against MongoBox and real MongoDB, then diff results
2. **Wire protocol unit tests** — Message framing and OP_MSG parsing, since these are pure functions easy to unit test
3. **Handshake integration test** — First end-to-end test: can a real driver connect and complete handshake?
4. **CRUD integration tests** — Insert → find → update → delete lifecycle using real driver
5. **Query operator parity tests** — Systematic testing of each query operator against real MongoDB
6. **Error parity tests** — Verify error codes and messages match
7. **Remaining areas** — Aggregation, indexes, auth, transactions as they're implemented

---

[← Back](README.md)
