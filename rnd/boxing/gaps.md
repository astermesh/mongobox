# Research Gaps — Closure Matrix

Audit and validation of research gaps identified during R01-boxing.

## Closure Matrix

| Gap | Status | Confidence | Evidence Quality | Blocking? |
|-----|--------|------------|------------------|-----------|
| G1: Mingo Deep-Dive | **Closed** | High | npm, GitHub, source tree | No |
| G2: Command Response Formats | **Closed** | High | Official MongoDB docs | No |
| G3: Test Strategy | **Reclassified** → task | N/A | Design decision | No* |
| G4: Document Store Design | **Closed** | Medium-High | Constrained by Mingo | No |
| G5: Missing ADR | **Reclassified** → task | N/A | Deliverable, not research | No* |
| G6: Multi-Driver Compat | **Closed (scoped)** | Medium | Wire protocol spec | No |
| G7: mongo-wire Evaluation | **Closed** | High | npm registry, GitHub | No |

\* Reclassified gaps become tasks in the first implementation story, not research blockers.

## G1: Mingo Library Deep-Dive

**Status**: Closed — comprehensive evaluation complete

**Evidence sources**: npm registry (v7.2.0), [GitHub kofrasa/mingo](https://github.com/kofrasa/mingo), source tree analysis

### Findings

**Package health**: v7.2.0 (Jan 2025), ~222K weekly downloads, zero dependencies, MIT license, actively maintained with monthly-quarterly releases, 1 open issue.

**Query operators** — comprehensive:
- Comparison: `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`
- Logical: `$and`, `$or`, `$nor`, `$not`
- Element: `$exists`, `$type`
- Array: `$all`, `$elemMatch`, `$size`
- Evaluation: `$expr`, `$jsonSchema`, `$mod`, `$regex`, `$where`
- Bitwise: `$bitsAllClear`, `$bitsAllSet`, `$bitsAnyClear`, `$bitsAnySet`
- **Missing**: Geospatial (`$geoWithin`, `$near`, etc.), full-text (`$text`) — server-dependent features, acceptable gaps

**Update operators** — ALL supported (corrects feasibility estimate of "2-3 weeks"):
`$set`, `$unset`, `$inc`, `$mul`, `$min`, `$max`, `$rename`, `$currentDate`, `$push`, `$pop`, `$pull`, `$pullAll`, `$addToSet`, `$bit`, positional `.$`

API: `update(obj, modifier)` for single object, `updateOne(docs, condition, modifier)` / `updateMany(...)` returning `{ matchedCount, modifiedCount }`.

**Aggregation stages** — 28 stages:
`$addFields`, `$bucket`, `$bucketAuto`, `$count`, `$densify`, `$documents`, `$facet`, `$fill`, `$graphLookup`, `$group`, `$limit`, `$lookup`, `$match`, `$merge`, `$out`, `$project`, `$redact`, `$replaceRoot`, `$replaceWith`, `$sample`, `$set`, `$setWindowFields`, `$skip`, `$sort`, `$sortByCount`, `$unionWith`, `$unset`, `$unwind`

**Missing stages**: `$changeStream`, `$currentOp`, `$listSessions`, `$collStats`, `$indexStats`, `$search`, `$geoNear` — all server-dependent, acceptable gaps.

**Expression operators** — 100+ across arithmetic (17), string (19), date (21), conditional (3), array (20+), type (10), set (7), object (4), boolean (3), comparison (7), accumulators (25+), window (10+).

**Edge-case handling**:
- `useStrictMode: true` (default) enforces MongoDB behavioral parity
- Array matching semantics match MongoDB (element-level + whole-array comparison)
- Dot notation fully supported (`a.b.c`, `arr.0.name`)
- Positional `.$` supported in projections and updates (v7.0.2+)
- Special variables: `$$ROOT`, `$$CURRENT`, `$$DESCEND`, `$$PRUNE`, `$$KEEP`, `$$REMOVE`, `$$NOW`

**What Mingo does NOT provide** (MongoBox must implement):
- Wire protocol, BSON serialization, indexes, persistence, transactions, cursor management, server commands

**sift.js comparison**: Query-only (no aggregation, no updates). Higher downloads (~4.35M/week) because it's a Mongoose dependency. Not suitable as Mingo replacement for MongoBox.

### Impact on feasibility estimates

The feasibility doc estimated "Update operators: Medium-High, 2-3 weeks." This is **wrong** — Mingo handles all update operators. The correct estimate for "Query engine (via Mingo)" should be **1 week integration only**, not 1-2 weeks.

### Remaining validation experiments

| Experiment | Priority | When |
|-----------|----------|------|
| Run Mingo update operators against MongoDB parity test cases | Medium | During S01 implementation |
| Benchmark Mingo query performance (1K-100K docs) | Low | Post-MVP |
| Verify `$elemMatch` projection behavior matches MongoDB | Medium | During S01 implementation |

These are implementation-time validations, not planning blockers.

## G2: MongoDB Command Response Formats

**Status**: Closed

**Evidence sources**: Official MongoDB documentation, MongoDB driver specifications, server source code

Complete response schemas documented in [command-responses.md](command-responses.md) covering: find, insert, update, delete, getMore, aggregate, findAndModify, listDatabases, listCollections, createIndexes, dropIndexes, killCursors. Error codes catalogued with MVP priority tiers (P0/P1/P2). Write concern error format documented.

## G3: Test Strategy

**Status**: Reclassified — this is a design task, not a research gap

**Rationale**: Test strategy is an engineering decision to be made as part of the first implementation story. It requires choosing tools and defining patterns, not discovering unknown information. The "behavioral parity" principle from AGENTS.md already defines the acceptance criteria standard.

**Recommended approach** (to be formalized during implementation):
- Framework: vitest (standard for modern TS projects)
- Parity approach: same test suite runs against MongoBox and mongodb-memory-server
- MVP scope: Node.js driver only
- Boundary: unit tests for wire protocol parsing, integration tests for command behavior

**Action**: Create as T01 task under the first implementation story.

## G4: Document Store and Cursor Design

**Status**: Closed — decisions are now constrained by G1 findings

**Rationale**: With Mingo's API surface known, most storage decisions are determined:

| Question | Answer | Rationale |
|----------|--------|-----------|
| Storage format | JS objects | Mingo operates on JS objects, not BSON buffers. Deserialize on receive, serialize on respond |
| Collection structure | Array | Mingo `find(collection, ...)` expects an array of documents |
| ObjectId generation | `bson` package `ObjectId` | Official implementation, matches wire format |
| Namespace management | `Map<string, Map<string, Document[]>>` | db → collection → docs |
| Cursor | Iterator over Mingo `Query.find()` result | Mingo returns a Cursor with `.next()`, `.limit()`, `.skip()`, `.sort()` |
| Index structures | Defer to post-MVP | Mingo does full scans; indexes are optimization, not correctness |
| Memory limits | None for MVP | In-memory test DB; configurable limits are future scope |

**Remaining design work** (not blocking planning):

| Topic | Priority | When |
|-------|----------|------|
| Unique index enforcement for `_id` | High | S01 story design |
| Cursor timeout/cleanup strategy | Medium | S01 story design |
| B-tree library selection for secondary indexes | Low | Post-MVP |

## G5: Missing ADR

**Status**: Reclassified — this is a deliverable task, not a research gap

**Rationale**: The research supporting the ADR is complete (summary.md, feasibility.md). Writing `docs/adr/01-pure-js-engine.md` is a task to be done before or during the first story.

**Action**: Create as task — write ADR from existing research artifacts.

## G6: Multi-Driver Compatibility

**Status**: Closed (scoped to MVP)

**Evidence sources**: Wire protocol spec (wire-protocol.md), MongoDB driver specifications

**Findings**:
- MVP targets **Node.js driver only** (the official `mongodb` npm package)
- Mongoose compatibility is automatic (Mongoose uses the Node.js driver)
- The wire protocol already handles both OP_MSG (modern) and OP_QUERY (legacy handshake)
- `maxWireVersion: 21` in handshake response is sufficient for current Node.js driver
- Multi-driver compatibility (Python, Java, Go) is a future concern, documented as a risk in feasibility.md

**Rationale for closure**: The remaining questions ("which drivers use OP_QUERY?") are implementation-time concerns that don't affect architecture or planning. Wire protocol implementation is driver-agnostic by design.

## G7: mongo-wire npm Evaluation

**Status**: Closed — build decision confirmed

**Evidence sources**: npm registry, GitHub

**Findings**:
- `mongo-wire-protocol` (the actual package name): v1.0.0, last published **2016**, 9 GitHub stars, 0 forks, unmaintained
- No modern standalone MongoDB wire protocol parser exists on npm
- The official `bson` package (6M+ weekly downloads) handles BSON serialization — this is the only needed dependency
- Wire protocol parser is ~500 lines of code per feasibility analysis

**Decision**: Build wire protocol parser from scratch. The protocol is simple (length-prefixed OP_MSG), well-documented, and no viable library exists.

## Summary

All 7 gaps are resolved:
- **4 closed** (G1, G2, G6, G7) — research complete with evidence
- **1 closed by constraint** (G4) — Mingo API constrains design decisions
- **2 reclassified as tasks** (G3, G5) — not research questions, just deliverables

**No remaining research blockers for planning.**

---

[← Back](README.md)
