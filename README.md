# Warrant — signed proof of intent for every AI-agent payment on PayPal

> **Live demo:** TODO hosted URL · **Video (2:59):** TODO YouTube link · **License:** MIT

**In one sentence:** Warrant lets a person give an AI agent a signed, budget-capped mandate, enforces
that cap on PayPal's own rails (an authorization hold), and resolves disputes automatically from the
signed record — so merchants stop paying for agent mistakes by default.

## PayPal + AI at a glance (for Stage One reviewers)

| PayPal capability | Where it's used | Code |
|---|---|---|
| JS SDK v6 — buyer approval of the mandate hold | Mandate Studio | TODO `src/app/studio/...` |
| Orders v2 `intent: AUTHORIZE` — hold = mandate cap | Mandate Studio | TODO `src/lib/paypal/orders.ts` |
| Payments v2 partial captures (`final_capture: false`), void, reauthorize | Policy Gate | TODO `src/lib/paypal/payments.ts` |
| Webhooks + signature verification | Ledger / Evidence Desk | TODO `src/app/api/webhooks/paypal/route.ts` |
| Disputes v1 — show, provide-evidence, accept-claim | Evidence Desk | TODO `src/lib/evidence/...` |
| Refunds | Evidence Desk | TODO |
| PayPal Agent Toolkit | Evidence Desk agent tools | TODO |

| AI component | Why it is necessary (not decorative) |
|---|---|
| Mandate compiler (Claude) | Turns "restock oat milk and 12-oz cups, under $200, approved suppliers, this week" into a structured, signable mandate |
| Intent-match judge (Claude) | Semantic check rules can't do ("16-oz ≠ 12-oz cups"). **Veto-only** — it can block or escalate, never approve money |
| Shopping agent (Claude + tools) | Searches the catalog and proposes carts; has no ability to capture money |
| Evidence Desk agent (Claude + PayPal Agent Toolkit) | Triages disputes against the signed record and writes the representment narrative |

## The problem (Potential Impact)
TODO — cite PayPal Agentic Commerce Pulse Report (Apr 2026): nearly two-thirds of merchants say a standardized
liability framework for AI-initiated purchases is urgently needed; liability today defaults to merchants.
Persona: TODO.

## What it does (Design) — 3 surfaces
TODO screenshots: Mandate Studio · Agent Console · Evidence Desk.

## How it works (Technological Implementation)
TODO architecture diagram (Mermaid) + the money state machine + the invariants (LLM can veto, never authorize; no capture without ledger; hold = cap; idempotency; verified webhooks).

## What's new (Innovation)
TODO — hold-as-cap; closed dispute loop via the real Disputes API; AP2-shaped mandates linked to PayPal IDs (`custom_id`/`invoice_id`).

## Run it yourself
TODO — ≤5 commands. Sandbox test buyer credentials for judges are in the Devpost testing instructions.

## Honest limitations & what's next
TODO — single-merchant scope (multi-merchant needs PayPal Commerce Platform), server-held signing key in demo (passkeys next), not full W3C VC stack.

## Built during the hackathon
All code written Oct 2026 for the PayPal AI Hackathon. AI used in the product: Claude via the Anthropic API (mandate compiler, intent-match judge, shopping agent, Evidence Desk narrative).
