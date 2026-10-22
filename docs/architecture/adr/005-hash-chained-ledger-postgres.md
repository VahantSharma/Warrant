# ADR 005 — Hash-chained, signed ledger in Postgres
Date: 2026-10-22 · Status: accepted

## Decision
Per-mandate append-only `ledger_events` with prev_hash/hash (SHA-256 over canonical JSON) and a system Ed25519 signature per event. Written in the same DB transaction as every state change. No blockchain, no external anchoring (optional later: daily anchor hash in a public gist).

## Consequences
+ Tamper-evident audit trail that becomes dispute evidence. Cheap and fast.
− Trust in the system key; documented. Append-only enforced by app code + a DB trigger rejecting UPDATE/DELETE.
