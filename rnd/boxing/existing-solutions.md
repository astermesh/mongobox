# Existing MongoDB Emulators and Compatibility Layers

Survey of existing solutions for MongoDB emulation, in-memory operation, and wire protocol compatibility.

## Summary Table

| Solution | Protocol Level | In-Memory | Language | Active | Approach |
|----------|---------------|-----------|----------|--------|----------|
| FerretDB | Wire protocol | Via PostgreSQL | Go | Yes | SQL translation |
| mongodb-memory-server | Wire protocol (real mongod) | ephemeralForTest | TS/C++ | Yes | Real process |
| TingoDB | API only | File-based | JS | Dead | API shim |
| NeDB (@seald-io) | API only | Yes | JS | Maintained fork | Embedded DB |
| mongo-mock | API only | Yes | JS | Dead | Mock |
| mongo-wire (npm) | Wire parser only | N/A | JS | Limited | Parser library |
| Mingo | Query engine only | Yes | JS | Yes | Query/aggregation |

## Detailed Analysis

### FerretDB

Go-based proxy that translates MongoDB wire protocol to SQL.

**Architecture**: Acts as MongoDB protocol-compatible frontend → translates commands to SQL → executes against PostgreSQL backend.

**Protocol support**: OP_MSG, targets MongoDB 5.0+ client compatibility. Implements SCRAM-SHA-256 auth.

**FerretDB 2.0 (March 2025)**: Major architectural shift — dropped the SQLite backend entirely, now exclusively uses PostgreSQL with the Microsoft DocumentDB extension. This makes FerretDB even less embeddable than before (requires a full PostgreSQL+DocumentDB stack).

**Supported features**:
- CRUD operations (find, insert, update, delete)
- Basic aggregation ($match, $group, $sort, $limit, $skip, $project, $count, $unwind)
- Indexes (basic types)
- Sessions and basic transactions
- All basic BSON data types

**Limitations**:
- No change streams
- No sharding
- Limited aggregation operators
- No GridFS
- Missing some query operators ($where, some geospatial)
- SQL translation overhead
- Requires PostgreSQL+DocumentDB (cannot run standalone)

**For MongoBox**: Valuable as reference for MongoDB command semantics, but cannot be embedded in Node.js/WASM (Go binary, requires external PostgreSQL).

**License**: Apache 2.0

### mongodb-memory-server

Downloads and runs actual mongod in-memory.

**How it works**:
1. First use: downloads mongod binary (~100-300MB)
2. Starts mongod as child process with `--storageEngine ephemeralForTest`
3. Provides connection URI
4. Tears down on close

**Pros**: Full MongoDB compatibility — it IS real MongoDB.

**Cons**:
- ~300MB binary download
- Heavy: each instance is a real mongod process
- Slow startup (1-5 seconds per instance)
- Platform-dependent (Linux/Mac/Windows binaries)
- Not embeddable — process wrapper only

**Maintenance**: Active, popular (~700K npm downloads/week)

**For MongoBox**: Opposite of our goal. Useful as compatibility test target ("does MongoBox match real MongoDB?").

### TingoDB

Embedded JavaScript database with MongoDB-compatible API.

**Approach**: API-compatible replacement for `mongodb` npm driver. Swap import, code works against TingoDB.

**Limitations**:
- API-level only, no wire protocol
- Abandoned (last commit ~2017-2018)
- Limited compatibility (MongoDB 2.x era)
- No aggregation framework
- File-based storage only

**For MongoBox**: Dead project, no value.

### NeDB (@seald-io/nedb)

Embedded persistent/in-memory database for Node.js with MongoDB-like API.

**Features**:
- In-memory or file-based persistence
- CRUD operations
- Single-field indexing (unique, sparse)
- Basic query operators ($gt, $lt, $in, $exists, etc.)
- Sort, projection, limit/skip

**Limitations**:
- No wire protocol
- No aggregation framework
- No transactions
- Limited query operators vs MongoDB
- Original abandoned (2016); fork `@seald-io/nedb` maintained

**For MongoBox**: Could serve as simple in-memory store, but Mingo is better for query compatibility.

### Mingo

JavaScript implementation of MongoDB query language.

**Features**:
- Query matching (comprehensive operator support)
- Aggregation pipeline (many stages and expressions)
- Update operators
- Pure JavaScript, works in Node.js and browser
- Actively maintained

**For MongoBox**: **Key building block.** Can serve as the query engine, avoiding the need to reimplement MongoDB's complex query semantics.

### Wire Protocol Libraries

**mongo-wire (npm)**: Parser/serializer for MongoDB wire protocol. Handles OP_MSG, OP_QUERY (legacy), uses official `bson` package.

**MongoDB Node.js driver source** (`mongodb` npm): Contains full wire protocol implementation in TypeScript (`src/cmap/` directory). Most production-tested implementation.

**mongoproxy / mongoshake**: Go-based proxy tools that parse and forward MongoDB wire protocol traffic. Reference for building protocol-level interceptors.

### Proxy / Testing Tools

**MongoBridge**: Built into MongoDB test suite (C++). TCP proxy for delays, drops, partition simulation. Not usable standalone.

**mongoreplay**: Official tool for capturing/replaying MongoDB traffic from pcap files. Proves protocol is inspectable at network level.

## Key Gap

There is **no existing lightweight, in-memory, wire-protocol-compatible MongoDB emulator in JavaScript/TypeScript**. The space between "API-level mock" (NeDB, Mingo) and "full MongoDB process" (mongodb-memory-server) is exactly what MongoBox would fill.

---

[← Back](README.md)
