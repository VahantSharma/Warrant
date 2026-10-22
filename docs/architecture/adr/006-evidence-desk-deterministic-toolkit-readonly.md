# ADR 006 — Evidence Desk: deterministic decisions; PayPal Agent Toolkit read-only
Date: 2026-10-22 · Status: accepted (pending spike-05)

## Decision
Dispute triage (WITHIN_MANDATE / MERCHANT_FAULT / NEEDS_REVIEW) is computed from data: signatures, chain integrity, cart vs mandate, fulfillment vs cart. The Evidence Desk agent uses PayPal Agent Toolkit tools restricted to reads (list/get disputes, order/transaction lookups) to gather context and write the narrative. Provide-evidence and accept-claim are called by our own typed client after the deterministic decision (human click, or auto-mode policy).

## Consequences
+ Uses PayPal's own agent tooling visibly (judges notice) without violating ADR 003.
− Two code paths to PayPal (toolkit + own client); acceptable and explained in README.
