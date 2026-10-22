# ADR 002 — The PayPal authorization hold IS the mandate spending cap
Date: 2026-10-22 · Status: accepted (pending spike-01)

## Context
Most "agent wallet" demos enforce budgets in app code. A bug or prompt injection can bypass app code. PayPal's Orders v2 supports `intent: AUTHORIZE` with multiple partial captures up to the authorized amount.

## Decision
On mandate signing, create a PayPal order (intent AUTHORIZE) for the mandate budget; the buyer approves with JS SDK v6; we authorize. All agent purchases are partial captures (`final_capture: false`) against that authorization. Remainder is voided at expiry/revocation.

## Consequences
+ Overspend is impossible at the payment-rail level, not just in our code. Strong innovation + tech story.
+ Uses more of PayPal's lifecycle (authorize, capture, void, reauthorize).
− Single merchant (payee) per mandate. Multi-merchant would need PayPal Commerce Platform — documented limitation.
− Honor period (3 days) and 29-day validity in production; sandbox auths don't expire. Production design would reauthorize; we document it.
