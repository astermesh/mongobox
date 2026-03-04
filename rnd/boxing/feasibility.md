# MongoBox Feasibility Analysis

Assessment of architecture options, effort estimates, and recommended approach.

## Option 1: FerretDB as Backend

### Current State (v2.0+, March 2025)
- FerretDB 2.0 dropped SQLite backend entirely
- Now exclusively requires PostgreSQL with Microsoft DocumentDB extension
- Startup time: ~1-2 seconds (Go binary + PostgreSQL)
- Memory footprint: ~50-100MB baseline (plus PostgreSQL)

### Can FerretDB Run in Node.js / WASM?
**No.** FerretDB is a Go application:
- Go → WASM output is 50-100MB+
- Go goroutine model maps poorly to WASM
- Requires external PostgreSQL+DocumentDB — cannot run standalone
- Network syscalls not available in WASM

**Verdict: Impractical.** Could run as sidecar process (like mongodb-memory-server) but defeats the "lightweight" goal.

## Option 2: FerretDB + PGlite

### The Idea
PGlite (Postgres-WASM) + FerretDB (Mongo→Postgres translator) = in-memory MongoDB in WASM.

### Reality
- FerretDB connects to PostgreSQL via TCP wire protocol
- PGlite runs in-process, not as TCP server (by default)
- Need: PGlite as wire-protocol server + FerretDB compiled to WASM
- Both halves are problematic (see above)

### Alternative: Reimplement Translation in TypeScript
- Use PGlite JavaScript API directly
- Reimplement FerretDB's Mongo→SQL translation logic in TypeScript
- FerretDB Go code as reference material
- Feasible but significant effort (~3-6 months for meaningful coverage)

**Verdict: Not a viable shortcut.** FerretDB's logic is useful reference, but the Go→TS port is too expensive for what you get.

## Option 3: Pure JS MongoDB Engine (Recommended)

### Architecture

```
TCP Server (Node.js net)
  → Wire Protocol Parser (OP_MSG + OP_COMPRESSED)
    → Command Router
      → Mingo-based Query Engine
        → In-Memory Document Store
```

### Core Components

**Wire protocol layer** (low complexity):
- TCP server with message framing
- OP_MSG parser/serializer
- BSON via `bson` npm package
- Handshake (hello/ismaster)

**Query engine** (use Mingo):
- Query matching — all operators ($eq, $gt, $in, $regex, $elemMatch, etc.)
- Aggregation pipeline ($match, $group, $sort, $project, $lookup, etc.)
- Update operators ($set, $unset, $inc, $push, $pull, etc.)
- Expression evaluation

**Document store** (medium complexity):
- Collections as maps of BSON documents
- Auto-generated ObjectId for `_id`
- Namespace-based (db.collection)
- Basic single-field indexes (B-tree or sorted array)
- Unique constraint enforcement

**Command set** (tiered):

| Tier | Commands | Effort |
|------|----------|--------|
| 1: Connection + CRUD | hello, ping, insert, find, update, delete, getMore, killCursors | ~2-4 weeks |
| 2: Indexes + Metadata | createIndexes, dropIndexes, listDatabases, listCollections, create, drop | +2-3 weeks |
| 3: Transactions + Advanced | abortTransaction, commitTransaction, distinct, findAndModify, count | +2-3 weeks |

### Key Shortcuts

- **`bson` npm** (~45KB): Official MongoDB BSON library, Node.js + browser. ~100K docs/sec serialization
- **`mingo` npm**: MongoDB query matching + aggregation in pure JS. Actively maintained. Avoids reimplementing the hardest part
- **Wire protocol parser**: ~500 lines of code. Length-prefixed messages = simple framing
- **`sift.js`**: Alternative to Mingo for query matching only (lighter)

### Why This Approach

| Factor | Pure JS Engine | FerretDB | mongodb-memory-server |
|--------|---------------|----------|----------------------|
| Startup time | ~ms | ~1-2s | ~1-5s |
| Memory footprint | ~10-50MB | ~50-100MB | ~100-300MB |
| Dependencies | bson + mingo | Go binary | mongod binary |
| Browser/WASM | Possible (minus TCP) | No | No |
| Hooks | Full control | External process | External process |

## Effort Estimates

### Component Breakdown

| Component | Complexity | Effort |
|-----------|-----------|--------|
| Wire protocol (TCP + OP_MSG + BSON) | Low | 1-2 weeks |
| Command routing + handshake | Low | 1 week |
| Document store (in-memory) | Medium | 1-2 weeks |
| Query engine (via Mingo) | Low (integration) | 1-2 weeks |
| Update operators | Medium-High | 2-3 weeks |
| Basic indexes | Medium | 2-3 weeks |
| SCRAM-SHA-256 auth | Medium | 2-3 days |
| Transactions (basic) | Medium | 1-2 weeks |

### Milestones

**MVP (Tier 1)**: Wire protocol + CRUD + handshake → **4-6 weeks**

**Usable (Tier 1+2)**: + indexes + metadata + aggregation → **8-10 weeks**

**Production (All tiers)**: + transactions + auth + advanced → **12-16 weeks**

## Risks

1. **MongoDB query semantics edge cases**: Array matching, nested paths, type coercion — many subtle behaviors. Mingo covers most but not all
2. **Driver compatibility**: Different MongoDB drivers may use slightly different protocol features. Need testing against Node.js, Python, Java, Go drivers
3. **Aggregation completeness**: Full pipeline coverage is a multi-year effort. Target 80% of common operations
4. **Performance**: In-memory JS engine will be faster for small datasets but may struggle with 100K+ documents per collection

## Recommendation

**Build pure JS engine with Mingo as query core.** Start with Tier 1 (wire protocol + CRUD), validate against real MongoDB Node.js driver. The wire protocol is the easy part — invest effort in query compatibility.

---

[← Back](README.md)
