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

| Package | Purpose | Size | Status |
|---------|---------|------|--------|
| `bson` | BSON encoding/decoding | ~45KB | Official MongoDB package, 6M+ weekly downloads |
| `mingo` | Query matching + aggregation + updates | ~1.1MB (tree-shakeable) | v7.2.0, 222K weekly downloads, zero deps, MIT |

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
| MVP | Wire protocol + CRUD + handshake | 3-4 weeks (revised down — Mingo handles updates) |
| Usable | + indexes + metadata + aggregation | 6-8 weeks |
| Production | + transactions + auth + advanced queries | 10-14 weeks |

## Risks

1. **Query semantics edge cases** — MongoDB has subtle behaviors (array matching, type coercion). Mingo's `useStrictMode` helps but edge cases may exist
2. **Driver compatibility** — MVP targets Node.js driver only; multi-driver support is future scope
3. **Aggregation completeness** — Mingo covers 28 pipeline stages. Missing: server-dependent stages ($changeStream, $geoNear, $search). Acceptable for MVP
4. **Scale** — In-memory JS fine for test datasets, may struggle with 100K+ documents
5. **Mingo BSON type gaps** — Mingo works with JS objects, not native BSON types (ObjectId, Decimal128, Timestamp). Need to verify type coercion at serialization boundary

---

[← Back](README.md)
