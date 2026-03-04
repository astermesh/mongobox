# R01: Boxing

Initial research for MongoBox — wire protocol, existing solutions, feasibility analysis.

## Status

Complete — architecture defined, feasibility validated, recommendation ready.

## Output

All research output in [rnd/boxing/](../../rnd/boxing/README.md).

Key results:

- [Wire protocol](../../rnd/boxing/wire-protocol.md) — OP_MSG, BSON, handshake, connection lifecycle
- [Existing solutions](../../rnd/boxing/existing-solutions.md) — FerretDB, mongodb-memory-server, NeDB, Mingo survey
- [Feasibility analysis](../../rnd/boxing/feasibility.md) — architecture options, effort estimates, risks
- [Summary](../../rnd/boxing/summary.md) — key findings and recommendation

## Key Findings

1. **No existing solution fills the gap** between API-level mocks (NeDB, Mingo) and full MongoDB process (mongodb-memory-server)
2. **Wire protocol is simple** — only OP_MSG + OP_COMPRESSED needed, ~500 lines parser
3. **FerretDB + PGlite path is impractical** — Go binary can't run in WASM
4. **Pure JS engine with Mingo** is the recommended approach — fast startup, small footprint, full hook integration
5. **MVP in 4-6 weeks**: wire protocol + CRUD + handshake

## Recommendation

Build pure JS engine: `TCP → wire protocol parser → command router → Mingo query engine → in-memory store`

---

[← Back to Work](../README.md)
