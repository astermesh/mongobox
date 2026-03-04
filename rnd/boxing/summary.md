# MongoBox Research Summary

## What is MongoBox

Lightweight in-memory MongoDB emulator. Intercepts at wire protocol level (TCP), not process/syscall level.

## Key Findings

### Wire Protocol is Simple

MongoDB wire protocol (post-3.6) uses only two opcodes: **OP_MSG** and **OP_COMPRESSED**. Messages are length-prefixed BSON documents over TCP. A wire protocol parser is ~500 lines of code. The protocol itself is not the challenge.

### No Existing Solution Fits

| Solution | Problem |
|----------|---------|
| FerretDB | Go binary, ~100MB, can't embed in Node.js/WASM |
| mongodb-memory-server | Downloads real mongod (~300MB), heavy process |
| NeDB / TingoDB | API-level only, no wire protocol, dead/limited |
| Mingo | Query engine only (no storage, no protocol) — but useful as building block |

**The gap**: No lightweight, in-memory, wire-protocol-compatible MongoDB emulator exists in JavaScript/TypeScript.

### FerretDB + PGlite is Impractical

The dream: FerretDB (Mongo→SQL translator) + PGlite (Postgres-WASM) = in-memory MongoDB.

The reality: FerretDB is Go with CGO dependency, cannot compile to WASM. Reimplementing translation logic in TypeScript is possible but expensive (~3-6 months) for questionable value.

## Recommended Architecture

Pure JS engine with wire protocol frontend:

```
MongoDB Client
  → TCP (port 27017)
    → Wire Protocol Parser (OP_MSG)
      → Command Router
        → Mingo Query Engine (query matching + aggregation)
          → In-Memory Document Store
```

### Key Dependencies

| Package | Purpose | Size |
|---------|---------|------|
| `bson` | BSON encoding/decoding | ~45KB |
| `mingo` | Query matching + aggregation | ~100KB |

### Why Pure JS

- **Startup**: milliseconds (vs 1-5 seconds for FerretDB/mongod)
- **Memory**: ~10-50MB (vs 50-300MB)
- **Integration**: Full control, hooks at every layer
- **Browser/WASM potential**: Everything except TCP works in browser

## Wire Protocol Visibility

Wire protocol level gives complete visibility:
- Every query with full filter, projection, sort
- Every write with documents and write concern
- Transaction lifecycle (session ID, txn number)
- Driver handshakes, cursor management, auth

## Effort Estimate

| Milestone | Scope | Effort |
|-----------|-------|--------|
| MVP | Wire protocol + CRUD + handshake | 4-6 weeks |
| Usable | + indexes + metadata + aggregation | 8-10 weeks |
| Production | + transactions + auth + advanced queries | 12-16 weeks |

## Risks

1. **Query semantics edge cases** — MongoDB has subtle behaviors (array matching, type coercion). Mingo covers most but not all
2. **Driver compatibility** — Different drivers may use different protocol features
3. **Aggregation completeness** — Full pipeline is a multi-year effort. Target 80% of common operations
4. **Scale** — In-memory JS fine for test datasets, may struggle with 100K+ documents

---

[← Back](README.md)
