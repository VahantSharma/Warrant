# Warrant — Architecture (v1)

One Next.js app (App Router) + Postgres (Neon free), deployed on Vercel Hobby — all free tiers (ADR 008). Server-side only for anything touching
PayPal, signing keys, or the LLM. See ADRs in `adr/` for the reasoning behind each choice.

## Components
```mermaid
flowchart LR
  subgraph Buyer side
    Studio[Mandate Studio /studio]
    Console[Agent Console /agent]
  end
  subgraph Merchant side
    Store[Supplier catalog /store + /api/catalog]
    Desk[Evidence Desk /desk]
  end
  Verify[/verify public/]

  Studio -->|compile| Compiler[Mandate Compiler LLM]
  Studio -->|sign| Signer[Signer jose Ed25519]
  Studio -->|JS SDK v6 approve| PP[(PayPal Sandbox)]
  Console --> Agent[Shopping Agent LLM + tools]
  Agent -->|search_catalog / propose_cart| Store
  Agent -->|proposed cart| Gate[Policy Gate]
  Gate -->|deterministic checks| Rules[Rule engine]
  Gate -->|veto-only| Judge[Intent-match Judge LLM]
  Gate -->|PASS| Capture[Capture Service]
  Capture -->|partial capture, PayPal-Request-Id| PP
  PP -->|webhooks| WH[Webhook ingest + verify]
  WH --> Ledger[(Ledger: hash-chained, signed)]
  Capture --> Ledger
  Gate --> Ledger
  Desk -->|Disputes v1| PP
  Desk --> Triage[Triage: deterministic + LLM explanation]
  Triage --> Packet[Evidence PDF]
  Packet -->|provide-evidence / accept-claim| PP
  Verify --> Ledger
  Cron[Expiry cron] -->|void remainder| PP
```

## The money state machine (mandate)
```mermaid
stateDiagram-v2
  [*] --> DRAFT: compiled
  DRAFT --> SIGNED: user signs
  SIGNED --> HOLD_PENDING: PayPal order created (AUTHORIZE)
  HOLD_PENDING --> ACTIVE: buyer approved + /authorize OK
  HOLD_PENDING --> CANCELLED: approval abandoned
  ACTIVE --> ACTIVE: partial capture (sum <= hold)
  ACTIVE --> EXHAUSTED: captured == hold (final capture)
  ACTIVE --> CLOSED: expiry/revoke -> void remainder
  EXHAUSTED --> CLOSED
  CLOSED --> [*]
```
Rules: transitions happen only in `src/lib/mandates/state.ts`; every transition writes a ledger event in the
same DB transaction as the state change; PayPal calls happen before commit with idempotency keys so a retry
after a crash is safe.

## Capture path — the critical section
1. Agent calls `propose_cart(items)`. The agent signs the cart proposal with its agent key (light KYA).
2. Gate loads mandate (`SELECT ... FOR UPDATE` on the mandate row) and computes:
   - deterministic checks: merchant allowed, each SKU/category allowed, qty ≤ max, unit price ≤ ceiling,
     cart total ≤ remaining (= hold − Σ captures − Σ in-flight), now within validity window, step-up threshold.
   - any FAIL → BLOCK. All PASS → ask Judge.
3. Judge (LLM, structured output: `{verdict: MATCH|MISMATCH|SUSPICIOUS, reason}`) can only downgrade:
   MISMATCH → ESCALATE, SUSPICIOUS → BLOCK. Judge errors/timeouts → ESCALATE (fail closed to human).
4. PASS → insert `captures` row status `PENDING` with id `cap_<ulid>` (reserves the amount inside the
   transaction), commit; call `POST /v2/payments/authorizations/{id}/capture` with
   `PayPal-Request-Id: cap_<ulid>`, `amount`, `final_capture:false`, `invoice_id: cap_<ulid>`,
   `note_to_payer` referencing the mandate; on success update row `COMPLETED` + ledger event; on failure
   mark `FAILED` (releases reservation) + ledger event.
5. A DB CHECK/trigger guarantees Σ(PENDING+COMPLETED captures) ≤ hold amount per mandate.

## Dispute path
1. `CUSTOMER.DISPUTE.CREATED` webhook (verified) or periodic `GET /v1/customer/disputes` poll (fallback for sandbox latency).
2. Resolve dispute → disputed transaction (capture ID) → `captures` row → cart → mandate → ledger slice.
3. Deterministic triage:
   - capture exists in ledger, chain intact, mandate signature valid, cart ⊆ mandate constraints, fulfillment
     record matches cart → **WITHIN_MANDATE** → provide-evidence.
   - fulfillment record ≠ cart (substitution/short-ship) → **MERCHANT_FAULT** → accept-claim (refund).
   - anything missing/invalid → **NEEDS_REVIEW** (human).
4. LLM writes the human-readable narrative and maps the buyer's claim text to the evidence (never decides).
5. Evidence PDF (pdf-lib) + `provide-evidence` multipart, or `accept-claim`. Both are ledger events.
6. In the demo, the sandbox-only settle/adjudicate call can show the final outcome.

## Data model (Drizzle, Postgres)
- `principals` (id, role: USER|AGENT|MERCHANT|SYSTEM, display_name, public_jwk)
- `mandates` (id `mdt_<ulid>`, user_id, agent_id, instruction_text, constraints jsonb, canonical_hash,
  user_signature (JWS), status, currency, hold_amount, valid_from, valid_until, step_up_threshold,
  paypal_order_id, paypal_authorization_id, created_at)
- `carts` (id `crt_<ulid>`, mandate_id, items jsonb, total, agent_signature, gate_decision, gate_checks jsonb,
  judge_verdict jsonb, approval jsonb nullable, created_at)
- `captures` (id `cap_<ulid>`, mandate_id, cart_id, amount, status PENDING|COMPLETED|FAILED|REFUNDED,
  paypal_capture_id, paypal_debug_id, created_at)
- `fulfillments` (id, capture_id, shipped_items jsonb) — demo merchant's shipping record (for MERCHANT_FAULT)
- `disputes` (id, paypal_dispute_id, capture_id, reason, buyer_message, status, triage, decision,
  evidence_pdf_url, outcome, timestamps)
- `ledger_events` (seq bigserial, mandate_id, type, payload jsonb, prev_hash, hash, signature, actor, created_at)
- `webhook_events` (paypal_event_id unique, type, verified bool, raw jsonb, processed_at) — dedupe
- `ai_usage` (day, provider, calls, tokens_in, tokens_out) — enforces self-imposed daily caps (ADR 008)
- `ai_cache` (key = sha256(task|model|canonical input), output jsonb, provider, model, created_at) — demo path only, always labelled
- `rate_limits` (bucket = ip|route|hour, count) — per-IP limits without Redis
- `approvals` (id, cart_id, requested_at, decided_at, decision, decided_by, signature) — step-up decisions

## Ledger
`hash_n = SHA-256( prev_hash_{n-1} || canonical_json(payload_n) )` per mandate chain (genesis prev_hash =
SHA-256(mandate_id)). `signature_n = Ed25519(system_key, hash_n)` (JWS detached). `/verify` recomputes all.
Canonical JSON = RFC 8785 (JCS) via a small vetted library.

## Security & safety
- Secrets server-only; `NEXT_PUBLIC_*` limited to PayPal client ID (and AG Grid license if required by the lib).
- All PayPal POSTs idempotent; webhooks verified and deduped.
- LLM inputs from the catalog are treated as untrusted data (wrapped/delimited); the agent has no money tools.
- Per-IP rate limit (Postgres counter) + `AI_MAX_CALLS_PER_RUN` + per-provider daily call caps set below free quotas (protect the hosted demo during judging).
- Demo roles via signed cookie, no real auth (document as limitation).

## Observability
- Structured logs (pino) with mandate/cart/capture IDs and PayPal `debug_id` on every PayPal error.
- `/api/health` checks DB + PayPal token fetch (cached) for the uptime monitor.

## AI layer (`src/lib/ai/`) — free tiers, provider-agnostic
```mermaid
flowchart LR
  Caller[compiler / agent / judge / narrative] --> AI[ai.run(task, input)]
  AI --> Budget{daily cap left?}
  Budget -->|yes| P1[Google Gemini free]
  P1 -->|429 / 5xx / timeout / invalid JSON x2| P2[Groq free]
  P2 -->|fails| FB[Honest fallback per task]
  Budget -->|no| FB
  AI --> Cache[(ai_cache: demo path only)]
  AI --> Usage[(ai_usage: calls per provider per day)]
```
- One entry point per task (`compileMandate`, `runShoppingAgent`, `judgeIntent`, `writeDisputeNarrative`) with a zod output schema.
- Fallback per task: judge → `ESCALATE` (fail closed); compiler → empty editable form + "use example"; narrative → deterministic template;
  agent → "AI busy, try in a minute" + guided demo offers the labelled cached run.
- Every AI output stored with `provider`, `model`, `latency_ms`, `cached` → shown in UI and ledger (judges see real AI).
- `AI_PROVIDER=fixture` replays recorded outputs for unit tests (zero quota use).
- Evals (`pnpm eval`): fixture-free live run of ~25 labelled cases (injection, look-alike SKUs, over-quantity, benign) against the judge
  and compiler; must pass thresholds before prompt/model changes merge. Results committed to `docs/research/EVALS.md`.

## Deployment (Vercel Hobby + Neon free)
- Vercel project from the owner's personal GitHub repo; env vars set in the Vercel dashboard (never in git).
- Runtime: Node.js for all routes touching PayPal/DB/signing (not Edge). `maxDuration` ≤ 300 s; agent runs are kept short (≤ 8 steps).
- Neon: pooled connection string for the app, direct string for `drizzle-kit migrate`. Interactive transactions must work with the
  chosen driver — proven by a concurrent-transaction test before any money path uses it.
- Crons (`vercel.json`, once per day max on Hobby): `/api/cron/sweep` (expire mandates → void remainder; reconcile PENDING captures;
  sync disputes) and `/api/cron/demo-reset`. Both require `Authorization: Bearer $CRON_SECRET`.
- Because crons are daily: **lazy expiry** on every mandate read, **"Sync with PayPal"** button + sync on Evidence Desk load, webhooks for real-time.
- Webhook URL = `https://<project>.vercel.app/api/webhooks/paypal` registered in the PayPal sandbox app.
- UptimeRobot (free) pings `/api/health` every 5 min → alerts + keeps functions/DB warm.
