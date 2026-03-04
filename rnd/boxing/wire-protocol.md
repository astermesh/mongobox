# MongoDB Wire Protocol

Technical specification of MongoDB wire protocol for building a protocol-level emulator.

## Table of Contents

- [Message Structure](#message-structure)
- [Opcodes](#opcodes)
- [OP_MSG Format](#op_msg-format)
- [OP_COMPRESSED Format](#op_compressed-format)
- [OP_QUERY Format (Legacy)](#op_query-format-legacy)
- [OP_REPLY Format (Legacy)](#op_reply-format-legacy)
- [BSON Encoding](#bson-encoding)
- [Connection Lifecycle](#connection-lifecycle)
- [Handshake](#handshake)
- [Authentication](#authentication)
- [Server Discovery (SDAM)](#server-discovery-sdam)
- [Wire Version Mapping](#wire-version-mapping)
- [Compression](#compression)
- [Implementation Notes for Emulator](#implementation-notes-for-emulator)

## Message Structure

All integers in the MongoDB wire protocol use **little-endian** byte order.

Every wire protocol message starts with a standard **MsgHeader** (16 bytes, 4 fields x 4 bytes):

```
Offset  Size    Field           Description
0       int32   messageLength   Total message size in bytes, including this header
4       int32   requestID       Client-generated identifier for this message
8       int32   responseTo      requestID from the original request (0 for client requests)
12      int32   opCode          Message type identifier
```

Byte layout example (little-endian):
```
5D 00 00 00    messageLength = 93
01 00 00 00    requestID = 1
00 00 00 00    responseTo = 0
DD 07 00 00    opCode = 2013 (OP_MSG)
```

The `messageLength` includes the 4 bytes of `messageLength` itself, so the remaining payload after reading the header is `messageLength - 16` bytes.

## Opcodes

### Current (MongoDB 3.6+)

| Opcode | Value | Hex | Description |
|--------|-------|-----|-------------|
| OP_MSG | 2013 | 0x07DD | Modern unified message format for all operations |
| OP_COMPRESSED | 2012 | 0x07DC | Wraps another message with compression |

### Deprecated / Removed

| Opcode | Value | Hex | Status |
|--------|-------|-----|--------|
| OP_REPLY | 1 | 0x0001 | Response to OP_QUERY. Deprecated, removed 5.1+ |
| OP_MSG (legacy) | 1000 | 0x03E8 | Obsolete diagnostic message. Do not implement |
| OP_UPDATE | 2001 | 0x07D1 | Removed in 5.1+ |
| OP_INSERT | 2002 | 0x07D2 | Removed in 5.1+ |
| OP_QUERY | 2004 | 0x07D4 | Removed in 5.1+ for find/commands. Exception: still accepted for `hello`/`isMaster` handshake |
| OP_GET_MORE | 2005 | 0x07D5 | Deprecated, removed 5.1+ |
| OP_DELETE | 2006 | 0x07D6 | Removed in 5.1+ |
| OP_KILL_CURSORS | 2007 | 0x07D7 | Deprecated, removed 5.1+ |

**Key insight**: For a new emulator targeting modern clients, supporting **OP_MSG + OP_COMPRESSED** is sufficient. However, some drivers still use **OP_QUERY** for the initial handshake (legacy hello / isMaster), so handling OP_QUERY for that one case adds compatibility with older drivers.

## OP_MSG Format

The primary message type since MongoDB 3.6. Used for all client requests and server responses.

### Complete Byte Layout

```
Offset  Size    Field
0       int32   messageLength      // total message size
4       int32   requestID          // unique message ID
8       int32   responseTo         // correlates to request
12      int32   opCode             // = 2013 (0xDD 0x07 0x00 0x00)
16      uint32  flagBits           // bit flags
20      ...     sections[]         // one or more sections
...     uint32  [checksum]         // optional CRC-32C (if bit 0 set)
```

### Flag Bits (uint32)

Bits 0-15 are "required" flags (message content). Bits 16-31 are "optional" flags (attributes).

| Bit | Name | Req/Resp | Description |
|-----|------|----------|-------------|
| 0 | checksumPresent | Both | CRC-32C checksum appended at end of message. Covers all bytes except the checksum itself |
| 1 | moreToCome | Both | Sender will send another message without waiting for response from receiver |
| 16 | exhaustAllowed | Request only | Client is prepared to receive multiple replies (exhaust cursor support) |

All undefined/reserved bits MUST be set to zero. Receivers MUST error on unknown required bits (0-15) and MUST ignore unknown optional bits (16-31).

**moreToCome semantics:**
- On requests: indicates an unacknowledged write (`w:0`). No response expected. If the server steps down as primary, the connection is closed.
- On responses: server will send additional responses. Client MUST continue reading until a response arrives without `moreToCome` set. Client MUST NOT send further requests until the sequence completes. Each response has a unique `requestID`; follow-up `responseTo` matches the previous response's `requestID`.

**exhaustAllowed semantics:**
- Client signals readiness for multi-response streams. Server may return `moreToCome` responses only if this bit is set.
- Supported on: `getMore` (MongoDB 4.2+), `hello` (MongoDB 4.4+, via topologyVersion).

### Section Types

Each section starts with a 1-byte `payloadType` followed by the payload.

#### Kind 0 -- Body (payloadType = 0x00)

Single BSON document. Exactly ONE Kind 0 section per message (mandatory).

```
Byte    Field
uint8   payloadType = 0
BSON    document        // size determined from BSON document's leading int32
```

The BSON document contains the command itself. For requests, it includes:
- The command name as the first key (e.g., `{find: "collectionName", ...}`)
- `$db` (required): database name string
- `$readPreference` (optional): defaults to `{mode: "primary"}`

For responses, it contains:
- `ok`: 1.0 (success) or 0.0 (failure)
- Command-specific response fields
- Optional `$clusterTime`, `operationTime`

#### Kind 1 -- Document Sequence (payloadType = 0x01)

Named sequence of documents. Zero or more Kind 1 sections per message. Each identifier MUST be unique.

```
Byte    Field
uint8   payloadType = 1
int32   size            // total size of this section (includes these 4 bytes)
cstring identifier      // null-terminated field name (e.g., "documents", "updates", "deletes")
BSON[]  documents       // concatenated BSON documents (no separators)
```

The `size` field includes itself (4 bytes) + identifier length + all document bytes. Documents are concatenated without any separator; each document's size is determined by its leading int32.

**Commands that support Kind 1 sections:**

| Command | Identifier | Contents |
|---------|-----------|----------|
| insert | `documents` | Documents to insert |
| update | `updates` | Update specifications |
| delete | `deletes` | Delete specifications |

Kind 1 sections are an optimization to avoid nesting large arrays inside the Kind 0 BSON document. They are semantically equivalent to including the array in the Kind 0 body.

**Unknown payload types**: receiving an unknown payloadType MUST trigger connection closure.

### Complete OP_MSG Example

Insert `{_id: 1, name: "test"}` into `mydb.users`:

```
                          OP_MSG structure
                          ─────────────────────────────
Header:
  57 00 00 00             messageLength = 87
  01 00 00 00             requestID = 1
  00 00 00 00             responseTo = 0
  DD 07 00 00             opCode = 2013

Flags:
  00 00 00 00             flagBits = 0 (no checksum, no moreToCome)

Section (Kind 0):
  00                      payloadType = 0
  42 00 00 00             BSON document size = 66
  02 69 6E 73 65 72 74 00 key = "insert"
  06 00 00 00             string length = 6
  75 73 65 72 73 00       value = "users"
  02 24 64 62 00          key = "$db"
  05 00 00 00             string length = 5
  6D 79 64 62 00          value = "mydb"
  04 64 6F 63 75 6D 65    key = "documents" (array type 0x04)
  6E 74 73 00
  ...                     array of documents
  00                      BSON document terminator
```

### Response Format

Server response is also OP_MSG with Kind 0 body. Typical successful response:

```json
{
  "ok": 1,
  "n": 1
}
```

Typical error response:

```json
{
  "ok": 0,
  "errmsg": "ns not found",
  "code": 26,
  "codeName": "NamespaceNotFound"
}
```

## OP_COMPRESSED Format

Wraps any other opcode with compression. OpCode = 2012.

### Byte Layout

```
Offset  Size    Field
0       int32   messageLength       // total compressed message size
4       int32   requestID           // message identifier
8       int32   responseTo          // correlation ID
12      int32   opCode              // = 2012 (0xDC 0x07 0x00 0x00)
16      int32   originalOpcode      // opcode of the wrapped message (e.g., 2013 for OP_MSG)
20      int32   uncompressedSize    // decompressed payload size (excluding MsgHeader)
24      uint8   compressorId        // compression algorithm identifier
25      ...     compressedMessage   // compressed payload (original message body without its MsgHeader)
```

### Compressor IDs

| ID | Algorithm | Notes |
|----|-----------|-------|
| 0 | noop | No compression (testing only) |
| 1 | snappy | Google Snappy. Fast, moderate ratio |
| 2 | zlib | Standard deflate. Good ratio, slower |
| 3 | zstd | Zstandard. Best balance of speed and ratio |
| 4-255 | reserved | Future use |

### Decompression Process

1. Read the 16-byte MsgHeader, extract `opCode = 2012`
2. Read `originalOpcode`, `uncompressedSize`, `compressorId`
3. Read remaining bytes as `compressedMessage`
4. Decompress using the identified algorithm
5. Reconstruct the original message: create a new MsgHeader with `originalOpcode`, prepend it to the decompressed body
6. Process as the original opcode

### Compression Negotiation

- Client advertises supported compressors in the `hello`/`isMaster` handshake via the `compression` field
- Server responds with its supported compressors in the `compression` field of the response
- When compressing, client MUST use the first compressor in the client's list that is also in the server's list

### Restricted Messages

The following commands MUST NOT be compressed: `hello`, `isMaster` (legacy hello), `saslStart`, `saslContinue`, `getnonce`, `authenticate`, `createUser`, `updateUser`, `copydbSaslStart`, `copydbgetnonce`, `copydb`.

## OP_QUERY Format (Legacy)

Opcode 2004. Still needed for initial handshake with older drivers (sending `isMaster`).

### Byte Layout

```
Offset  Size    Field
0       int32   messageLength
4       int32   requestID
8       int32   responseTo
12      int32   opCode              // = 2004 (0xD4 0x07 0x00 0x00)
16      int32   flags               // query flags (bit field)
20      cstring fullCollectionName  // "dbname.collectionname" or "admin.$cmd" for commands
...     int32   numberToSkip        // documents to skip
...     int32   numberToReturn      // documents to return (-1 = one document for commands)
...     BSON    query               // query/command document
...     [BSON]  returnFieldsSelector // optional projection
```

### Query Flag Bits

| Bit | Name | Description |
|-----|------|-------------|
| 0 | Reserved | Must be 0 |
| 1 | TailableCursor | Cursor is not closed when last data is retrieved |
| 2 | SlaveOk | Allow query of replica set secondaries |
| 3 | OplogReplay | Internal replication use only |
| 4 | NoCursorTimeout | Prevent cursor timeout after 10 min inactivity |
| 5 | AwaitData | Block briefly at data end rather than returning immediately |
| 6 | Exhaust | Stream all results in multiple packages |
| 7 | Partial | Get partial results when some shards are unavailable |
| 8-31 | Reserved | Must be 0 |

### OP_QUERY for Handshake

When used for `isMaster` handshake:
```
fullCollectionName = "admin.$cmd"
numberToSkip = 0
numberToReturn = -1
query = {isMaster: 1, helloOk: true, client: {...}}
```

## OP_REPLY Format (Legacy)

Opcode 1. Server response to OP_QUERY.

### Byte Layout

```
Offset  Size    Field
0       int32   messageLength
4       int32   requestID
8       int32   responseTo          // requestID from the OP_QUERY
12      int32   opCode              // = 1 (0x01 0x00 0x00 0x00)
16      int32   responseFlags       // response flags (bit field)
20      int64   cursorID            // cursor identifier (0 if no cursor)
28      int32   startingFrom        // starting position in result set
32      int32   numberReturned      // number of documents in this response
36      BSON[]  documents           // response documents
```

### Response Flag Bits

| Bit | Name | Description |
|-----|------|-------------|
| 0 | CursorNotFound | Cursor ID was invalid (getMore on closed cursor) |
| 1 | QueryFailure | Query failed. First document contains error info |
| 2 | ShardConfigStale | Internal mongos use only |
| 3 | AwaitCapable | Server supports AwaitData query option |
| 4-31 | Reserved | Ignore |

## BSON Encoding

Binary JSON format. All values in **little-endian** byte order.

### Document Structure

```
document ::= int32 (total_size) element* 0x00
element  ::= type_byte key_cstring value
```

- `int32 total_size`: includes the 4 bytes of the size field itself and the trailing 0x00
- Elements are stored sequentially with no padding/alignment
- Terminated by a single 0x00 byte

### String Types

**string**: `int32(length) + UTF-8_bytes + 0x00`
- `length` includes the trailing null byte
- Example: "hello" => `06 00 00 00 68 65 6C 6C 6F 00`

**cstring**: `UTF-8_bytes + 0x00`
- No length prefix, null-terminated
- MUST NOT contain internal 0x00 bytes
- Used for element keys and some values (regex patterns/options)

### Complete Type Table

| Code | Hex | Type | Value Encoding |
|------|-----|------|---------------|
| 1 | 0x01 | Double | 8 bytes IEEE 754 binary64 |
| 2 | 0x02 | String | int32(length) + UTF-8 bytes + 0x00 |
| 3 | 0x03 | Document | Nested BSON document (recursive) |
| 4 | 0x04 | Array | BSON document with sequential integer string keys ("0", "1", "2"...) |
| 5 | 0x05 | Binary | int32(byte_count) + uint8(subtype) + bytes |
| 6 | 0x06 | Undefined | No value bytes. Deprecated |
| 7 | 0x07 | ObjectId | 12 bytes (4 timestamp + 5 random + 3 counter) |
| 8 | 0x08 | Boolean | uint8: 0x00 = false, 0x01 = true |
| 9 | 0x09 | DateTime | int64, UTC milliseconds since Unix epoch |
| 10 | 0x0A | Null | No value bytes |
| 11 | 0x0B | Regex | cstring(pattern) + cstring(options). Options alphabetically sorted: i,m,s,u,x |
| 12 | 0x0C | DBPointer | string + 12 bytes. Deprecated |
| 13 | 0x0D | JavaScript | string (code) |
| 14 | 0x0E | Symbol | string. Deprecated |
| 15 | 0x0F | CodeWithScope | int32(total_size) + string(code) + document(scope). Deprecated |
| 16 | 0x10 | Int32 | 4 bytes signed two's complement |
| 17 | 0x11 | Timestamp | uint64: low 32 bits = increment, high 32 bits = seconds since epoch |
| 18 | 0x12 | Int64 | 8 bytes signed two's complement |
| 19 | 0x13 | Decimal128 | 16 bytes IEEE 754 decimal128 |
| -1 | 0xFF | MinKey | No value bytes. Special comparison type |
| 127 | 0x7F | MaxKey | No value bytes. Special comparison type |

### Binary Subtypes

| Subtype | Description |
|---------|-------------|
| 0x00 | Generic binary (default) |
| 0x01 | Function |
| 0x02 | Binary (old, deprecated - has extra int32 length prefix) |
| 0x03 | UUID (old, deprecated) |
| 0x04 | UUID (RFC 4122) |
| 0x05 | MD5 |
| 0x06 | Encrypted BSON value |
| 0x07 | Compressed time series column |
| 0x08 | Sensitive data |
| 0x09 | Vector |
| 0x80-0xFF | User-defined |

### BSON Encoding Example

Document: `{_id: 1, name: "test"}`

```
Bytes (hex):
1E 00 00 00          document size = 30
10                   type = Int32 (0x10)
5F 69 64 00          key = "_id\0"
01 00 00 00          value = 1
02                   type = String (0x02)
6E 61 6D 65 00       key = "name\0"
05 00 00 00          string length = 5 (includes null terminator)
74 65 73 74 00       value = "test\0"
00                   document terminator
```

### Stream Parsing

BSON documents and wire protocol messages are both length-prefixed, making TCP stream parsing straightforward:

```
1. Read 4 bytes → messageLength (little-endian int32)
2. Read (messageLength - 4) more bytes → complete message
3. Parse header: requestID, responseTo, opCode
4. Based on opCode, parse body
5. For OP_MSG: read flagBits, then read sections until bytes exhausted
   (account for 4-byte checksum if checksumPresent flag is set)
6. For each section: read payloadType byte, then payload
```

## Connection Lifecycle

### 1. TCP Connect

Default port **27017**. Plain TCP socket. TLS optional (negotiated at transport level, not in wire protocol).

The server listens for connections and processes each connection independently. Each connection typically handles one request-response at a time (no pipelining), though `moreToCome` and `exhaustAllowed` flags can modify this.

### 2. Handshake

First message on every new connection MUST be a handshake. Two paths:

**Modern path (MongoDB 4.4+, server API version requested, or loadBalanced):**
Client sends `hello` command via OP_MSG.

**Legacy path (for maximum compatibility):**
Client sends `isMaster` (legacy hello) via OP_QUERY to `admin.$cmd`, including `helloOk: true`. If server responds with `helloOk: true`, client switches to `hello` via OP_MSG for subsequent monitoring.

### Handshake Request

```json
{
  "hello": 1,
  "$db": "admin",
  "helloOk": true,
  "client": {
    "application": { "name": "myapp" },
    "driver": { "name": "nodejs", "version": "6.0.0" },
    "os": {
      "type": "Linux",
      "name": "linux",
      "architecture": "x64",
      "version": "5.15.0"
    },
    "platform": "Node.js v20.0.0"
  },
  "compression": ["snappy", "zstd", "zlib"],
  "saslSupportedMechs": "admin.myuser",
  "speculativeAuthenticate": {
    "saslStart": 1,
    "mechanism": "SCRAM-SHA-256",
    "payload": "<base64 client-first-message>",
    "db": "admin",
    "options": { "skipEmptyExchange": true }
  }
}
```

**Client metadata constraints**: The entire `client` document MUST NOT exceed 512 bytes (including BSON overhead). `application.name` MUST NOT exceed 128 bytes.

### Handshake Response (Standalone Server)

```json
{
  "helloOk": true,
  "isWritablePrimary": true,
  "topologyVersion": {
    "processId": {"$oid": "..."},
    "counter": {"$numberLong": "0"}
  },
  "maxBsonObjectSize": 16777216,
  "maxMessageSizeBytes": 48000000,
  "maxWriteBatchSize": 100000,
  "localTime": {"$date": "2024-01-01T00:00:00.000Z"},
  "logicalSessionTimeoutMinutes": 30,
  "connectionId": 1,
  "minWireVersion": 0,
  "maxWireVersion": 21,
  "readOnly": false,
  "ok": 1
}
```

### Key Response Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| isWritablePrimary | bool | | true for primary, mongos, standalone |
| maxBsonObjectSize | int32 | 16777216 | Max BSON document size (16 MB) |
| maxMessageSizeBytes | int32 | 48000000 | Max wire protocol message size (~48 MB) |
| maxWriteBatchSize | int32 | 100000 | Max write operations per batch |
| minWireVersion | int32 | 0 | Minimum supported wire version |
| maxWireVersion | int32 | | Maximum supported wire version |
| logicalSessionTimeoutMinutes | int32 | 30 | Session timeout |
| connectionId | int32 | | Unique connection ID |
| readOnly | bool | false | Server is read-only |
| compression | array | | Supported compressor names |
| saslSupportedMechs | array | | Auth mechanisms for requested user |
| helloOk | bool | | Server supports `hello` command |
| ok | double | | 1.0 for success |

**Additional fields for replica sets**: `hosts`, `setName`, `setVersion`, `secondary`, `primary`, `me`, `arbiterOnly`, `passive`, `hidden`, `tags`, `electionId`, `lastWrite`.

**Additional fields for mongos**: `msg: "isdbgrid"`.

### 3. Normal Operations

All operations use OP_MSG with a Kind 0 body containing the command document. The `$db` field specifies the target database.

**Common commands:**

```json
// find
{"find": "users", "filter": {"age": {"$gt": 21}}, "$db": "mydb"}

// insert
{"insert": "users", "documents": [{"_id": 1, "name": "Alice"}], "$db": "mydb"}

// update
{"update": "users", "updates": [{"q": {"_id": 1}, "u": {"$set": {"name": "Bob"}}}], "$db": "mydb"}

// delete
{"delete": "users", "deletes": [{"q": {"_id": 1}, "limit": 1}], "$db": "mydb"}

// aggregate
{"aggregate": "users", "pipeline": [{"$match": {"age": {"$gt": 21}}}], "cursor": {}, "$db": "mydb"}

// getMore (cursor continuation)
{"getMore": {"$numberLong": "12345"}, "collection": "users", "$db": "mydb"}

// killCursors
{"killCursors": "users", "cursors": [{"$numberLong": "12345"}], "$db": "mydb"}

// listDatabases
{"listDatabases": 1, "$db": "admin"}

// listCollections
{"listCollections": 1, "$db": "mydb"}

// createIndexes
{"createIndexes": "users", "indexes": [{"key": {"name": 1}, "name": "name_1"}], "$db": "mydb"}

// ping
{"ping": 1, "$db": "admin"}

// buildInfo
{"buildInfo": 1, "$db": "admin"}

// serverStatus
{"serverStatus": 1, "$db": "admin"}
```

## Authentication

### SCRAM-SHA-256 (Default since MongoDB 4.0)

SCRAM (Salted Challenge Response Authentication Mechanism) per RFC 7677 / RFC 5802. Uses SHA-256 hashing. Four-message exchange mapped to two MongoDB command round-trips.

#### Round Trip 1: saslStart

**Client sends:**
```json
{
  "saslStart": 1,
  "mechanism": "SCRAM-SHA-256",
  "payload": "BinData(0, '<base64 of client-first-message>')",
  "autoAuthorize": 1,
  "$db": "admin",
  "options": { "skipEmptyExchange": true }
}
```

**Client-first-message format** (plaintext before base64):
```
n,,n=<username>,r=<client-nonce>
```
- `n,,` is the GS2 header (no channel binding, no authzid)
- `n=<username>` is the SASL-prepped username
- `r=<client-nonce>` is a random base64 string

**Server responds:**
```json
{
  "conversationId": 1,
  "done": false,
  "payload": "BinData(0, '<base64 of server-first-message>')",
  "ok": 1
}
```

**Server-first-message format**:
```
r=<client-nonce><server-nonce>,s=<base64-salt>,i=<iteration-count>
```
- `r=` combined nonce (client nonce + server nonce appended)
- `s=` base64-encoded salt
- `i=` iteration count (default 15000 for SHA-256)

#### Round Trip 2: saslContinue

**Client computes:**
1. `SaltedPassword = PBKDF2(HMAC-SHA-256, password, salt, iterations, 32)`
2. `ClientKey = HMAC-SHA-256(SaltedPassword, "Client Key")`
3. `StoredKey = SHA-256(ClientKey)`
4. `AuthMessage = client-first-message-bare + "," + server-first-message + "," + client-final-message-without-proof`
5. `ClientSignature = HMAC-SHA-256(StoredKey, AuthMessage)`
6. `ClientProof = ClientKey XOR ClientSignature`
7. `ServerKey = HMAC-SHA-256(SaltedPassword, "Server Key")`
8. `ServerSignature = HMAC-SHA-256(ServerKey, AuthMessage)`

**Client sends:**
```json
{
  "saslContinue": 1,
  "conversationId": 1,
  "payload": "BinData(0, '<base64 of client-final-message>')",
  "$db": "admin"
}
```

**Client-final-message format**:
```
c=biws,r=<combined-nonce>,p=<base64-client-proof>
```
- `c=biws` is base64 of "n,," (channel binding data for no binding)
- `p=` is the base64-encoded client proof

**Server responds:**
```json
{
  "conversationId": 1,
  "done": true,
  "payload": "BinData(0, '<base64 of server-final-message>')",
  "ok": 1
}
```

**Server-final-message format**:
```
v=<base64-server-signature>
```

The client MUST verify the server signature matches its computed `ServerSignature`. This provides mutual authentication.

#### Speculative Authentication

Since MongoDB 4.4, the client can include `speculativeAuthenticate` in the `hello` handshake to merge the first auth round-trip with the handshake. The server's handshake response will contain `speculativeAuthenticate` with the `saslStart` response, saving one round-trip.

#### SCRAM-SHA-1

Same flow as SCRAM-SHA-256 but uses SHA-1 hashing. Default iteration count is 10000. SCRAM-SHA-256 is preferred when the server supports it.

#### Mechanism Negotiation

When no mechanism is specified by the client:
1. Client includes `saslSupportedMechs: "<db>.<username>"` in the `hello` command
2. Server returns `saslSupportedMechs: ["SCRAM-SHA-256", "SCRAM-SHA-1"]`
3. Client selects SCRAM-SHA-256 if available, otherwise falls back to SCRAM-SHA-1

## Server Discovery (SDAM)

### Topology Types

| Type | Description |
|------|-------------|
| Unknown | Initial state, type not yet determined |
| Single | Direct connection to one server |
| ReplicaSetNoPrimary | Replica set without elected primary |
| ReplicaSetWithPrimary | Replica set with active primary |
| Sharded | Multiple mongos instances |
| LoadBalanced | Connection through load balancer |

### Server Types

Standalone, Mongos, RSPrimary, RSSecondary, RSArbiter, RSOther, RSGhost, LoadBalancer, Unknown.

"Data-bearing" types: Standalone, Mongos, RSPrimary, RSSecondary, LoadBalancer.

### Monitoring

Drivers periodically send `hello` / `isMaster` to all known servers:
- `minHeartbeatFrequencyMS` = 500ms (non-configurable minimum)
- Default heartbeat interval: 10 seconds
- Drivers measure round-trip time (RTT) for server selection

### For Emulator

Respond as **Standalone** with `isWritablePrimary: true`. This is the simplest topology and avoids replica set discovery complexity. Do not include `setName`, `hosts`, `secondary`, or other replica-set-specific fields.

## Wire Version Mapping

Wire versions negotiated via `minWireVersion` / `maxWireVersion` in the `hello` response.

| Wire Version | MongoDB Version | Key Features |
|-------------|----------------|--------------|
| 5 | 3.4 | Write concern on all commands, collation |
| 6 | 3.6 | OP_MSG support, change streams, retryable writes, logical sessions |
| 7 | 4.0 | Replica set transactions, SCRAM-SHA-256 |
| 8 | 4.2 | Sharded transactions |
| 9 | 4.4 | Streaming SDAM, speculative auth, resumable change streams |
| 13 | 5.0 | $out/$merge on secondaries |
| 17 | 6.0 | Partial indexes, sharded time series |
| 21 | 7.0 | Search index management, slot-based execution |
| 25 | 8.0 | Range encryption, OIDC auth, bulkWrite command |

**For emulator**: Use `maxWireVersion: 21` (7.0 compatible) or `maxWireVersion: 17` (6.0 compatible) depending on what feature set you want to emulate.

## Compression

### Negotiation Flow

1. Client includes `compression: ["snappy", "zstd", "zlib"]` in handshake
2. Server responds with `compression: ["zstd"]` (intersection of supported)
3. Subsequent messages can be wrapped in OP_COMPRESSED
4. Client MUST use first compressor from client's list that appears in server's list

### For Emulator

Return `compression: []` (empty array) or omit the field entirely in the handshake response. This tells the client that no compression is supported, and all subsequent messages will use plain OP_MSG. This is the simplest approach.

To support compression later:
- **zlib** is the easiest to implement (built-in to Node.js via `zlib` module)
- **snappy** requires `snappy` npm package
- **zstd** requires `@napi-rs/zstd` or similar

## Implementation Notes for Emulator

### Minimum Viable Wire Protocol Server

To accept connections from a standard MongoDB driver, the server must:

1. **Listen on TCP** (default port 27017)
2. **Parse incoming messages**: read 4-byte length prefix, then remaining bytes
3. **Handle OP_MSG** (opcode 2013): parse flagBits, sections, extract BSON command
4. **Optionally handle OP_QUERY** (opcode 2004): for legacy `isMaster` handshake only
5. **Respond to `hello`/`isMaster`**: return standalone topology info with capabilities
6. **Respond to `ping`**: return `{ok: 1}`
7. **Route commands**: extract first key from BSON document to determine command type

### Message Framing (pseudo-code)

```
buffer = Buffer.alloc(0)

on('data', chunk):
  buffer = concat(buffer, chunk)
  while buffer.length >= 4:
    messageLength = buffer.readInt32LE(0)
    if buffer.length < messageLength:
      break  // wait for more data
    message = buffer.slice(0, messageLength)
    buffer = buffer.slice(messageLength)
    processMessage(message)
```

### Building OP_MSG Response

```
function buildResponse(requestId, responseDoc):
  bsonBody = serialize(responseDoc)
  // Section: Kind 0 + BSON
  section = Buffer.concat([Buffer.from([0x00]), bsonBody])
  // Flags
  flags = Buffer.alloc(4, 0)
  // Header
  totalLength = 16 + 4 + section.length  // header + flags + section
  header = Buffer.alloc(16)
  header.writeInt32LE(totalLength, 0)     // messageLength
  header.writeInt32LE(nextId(), 4)        // requestID
  header.writeInt32LE(requestId, 8)       // responseTo
  header.writeInt32LE(2013, 12)           // opCode = OP_MSG
  return concat(header, flags, section)
```

### Complexity Assessment

| Component | Complexity | Effort |
|-----------|-----------|--------|
| TCP server + message framing | Low | ~1 day |
| OP_MSG parsing/serialization | Low | ~2-3 days |
| BSON (via library) | None | Use `bson` npm package |
| Handshake (hello/isMaster) | Low | ~1 day |
| OP_QUERY for legacy handshake | Low | ~0.5 day |
| SCRAM-SHA-256 auth | Medium | ~2-3 days |
| Compression support | Low | ~1 day |
| Command routing | Low | ~1 day |

**Total wire protocol layer**: ~1-2 weeks. The protocol itself is straightforward — the complexity lies in the query engine, not the wire format.

### Topics for Further Research

The following wire-level topics are not covered here but will be needed during implementation:

- **Cursor lifecycle**: batch sizes, cursor expiry, `getMore` response format (`cursor.id`, `cursor.nextBatch`), `noCursorTimeout`
- **Session/transaction fields**: `lsid` (logical session ID), `txnNumber`, `startTransaction`, `autocommit` as they appear in OP_MSG command documents
- **Error response structures**: `writeErrors` array, `writeConcernError` object, bulk write error format
- **Write/read concern fields**: how `writeConcern` and `readConcern` appear in command documents

### Sources

- [MongoDB Wire Protocol (Official Docs)](https://www.mongodb.com/docs/manual/reference/mongodb-wire-protocol/)
- [OP_MSG Specification (GitHub)](https://github.com/mongodb/specifications/blob/master/source/message/OP_MSG.md)
- [OP_COMPRESSED Specification (GitHub)](https://github.com/mongodb/specifications/blob/master/source/compression/OP_COMPRESSED.md)
- [MongoDB Authentication Specification (GitHub)](https://github.com/mongodb/specifications/blob/master/source/auth/auth.md)
- [BSON Specification](https://bsonspec.org/spec.html)
- [MongoDB Handshake Specification](https://specifications.readthedocs.io/en/latest/mongodb-handshake/handshake/)
- [SDAM Specification](https://github.com/mongodb/specifications/blob/master/source/server-discovery-and-monitoring/server-discovery-and-monitoring.md)
- [Wire Version Feature List](https://specifications.readthedocs.io/en/latest/wireversion-featurelist/wireversion-featurelist/)

---

[← Back](README.md)
