# R01: Boxing

Initial research for MongoBox — wire protocol, existing solutions, feasibility analysis.

## Status

Complete — all research gaps validated and closed. Ready for planning.

## Output

All research output in [rnd/boxing/](../../rnd/boxing/README.md).

Key results:

- [Wire protocol](../../rnd/boxing/wire-protocol.md) — OP_MSG, BSON, handshake, connection lifecycle
- [Existing solutions](../../rnd/boxing/existing-solutions.md) — FerretDB, mongodb-memory-server, NeDB, Mingo survey
- [Feasibility analysis](../../rnd/boxing/feasibility.md) — architecture options, effort estimates, risks
- [Command response formats](../../rnd/boxing/command-responses.md) — exact response schemas for all MongoDB commands
- [Summary](../../rnd/boxing/summary.md) — key findings and recommendation
- [Research gaps — closure matrix](../../rnd/boxing/gaps.md) — all 7 gaps validated and resolved

## Key Findings

1. **No existing solution fills the gap** between API-level mocks (NeDB, Mingo) and full MongoDB process (mongodb-memory-server)
2. **Wire protocol is simple** — only OP_MSG + OP_COMPRESSED needed, ~500 lines parser
3. **FerretDB + PGlite path is impractical** — Go binary can't run in WASM
4. **Pure JS engine with Mingo** is the recommended approach — fast startup, small footprint, full hook integration
5. **Mingo coverage is broader than expected** — query, update, and aggregation operators are all handled; original effort estimates for update operators were too high
6. **No wire protocol library worth reusing** — `mongo-wire-protocol` abandoned since 2016; build from scratch
7. **MVP in 4-6 weeks**: wire protocol + CRUD + handshake (may be faster given Mingo covers updates)

## Revised Effort Estimate

Original feasibility doc overestimated "Update operators: Medium-High, 2-3 weeks." Mingo handles all 15 update operators natively. Revised component estimate:

| Component | Original | Revised | Rationale |
|-----------|----------|---------|-----------|
| Wire protocol | 1-2 weeks | 1-2 weeks | No change |
| Command routing + handshake | 1 week | 1 week | No change |
| Document store (in-memory) | 1-2 weeks | 1 week | Design constrained by Mingo API |
| Query engine (via Mingo) | 1-2 weeks | 1 week | Integration only |
| Update operators | 2-3 weeks | **0** (included in Mingo) | Mingo handles all update operators |
| Basic indexes | 2-3 weeks | Defer to post-MVP | Mingo does full scans; indexes are optimization |
| **MVP total** | **4-6 weeks** | **3-4 weeks** | |

## Recommendation

Build pure JS engine: `TCP → wire protocol parser → command router → Mingo query engine → in-memory store`

## Pre-Planning Tasks

Before starting S01 story:

1. Write ADR `docs/adr/01-pure-js-engine.md` (from existing research)
2. Define test strategy as T01 task within S01

---

[← Back to Work](../README.md)
