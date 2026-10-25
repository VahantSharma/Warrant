# Warrant — Product Requirements (v1, Oct 22 2026)

## One-liner
Signed proof of intent for every AI-agent payment on PayPal: the hold is the cap, the model can veto but
never authorize, and disputes resolve themselves from the signed record.

## Problem
AI agents are starting to buy for people and businesses. When a buyer later says "my agent bought that, not me",
there is no standard evidence of what the human actually authorized; liability defaults to the merchant.
PayPal's own merchant survey says nearly two-thirds want a standardized liability framework now.
PayPal has published (AP2 post, Sep 2025) the direction — mandates feeding Seller Protection and disputes —
but there's no working developer-level implementation on its APIs.

## Personas (fictional, consistent across UI, seed data, video)
- **Maya Rao — owner of "Ember & Oat Café".** Busy; wants her restock agent to buy supplies weekly without
  surprise charges. Cares about: control, simplicity, "what did it spend?".
- **Daniel — ops lead at "Bean Brew Supply"** (a wholesale supplier, the PayPal merchant). Sells to cafés;
  increasingly receives agent orders. Cares about: not eating disputes for purchases the customer authorized.
- **The restock agent** — an AI agent acting for Maya. Has its own signing key (light KYA).

## Surfaces
1. **Mandate Studio** (`/studio`, Maya)
   - Natural-language instruction → AI compiles a structured Intent Mandate (budget, currency, allowed merchant,
     categories/SKUs, max quantity per item, validity window, step-up threshold).
   - Human-readable summary + diff view of every constraint; Maya edits constraints directly (form), not by re-prompting.
   - Sign (server-held user key in v1; passkey/WebAuthn as stretch) → mandate ID + signature shown.
   - Approve the PayPal hold with JS SDK v6 → order (intent AUTHORIZE) authorized → "Hold active: $200.00".
2. **Agent Console** (`/agent`, Maya watches)
   - Live stream of the agent's reasoning steps and tool calls (search catalog, propose cart).
   - Each proposed cart shows the **Policy Gate** verdict: deterministic checks list ✓/✗, intent-match judge
     verdict + one-sentence reason, final decision PASS / ESCALATE / BLOCK.
   - PASS → partial capture against the hold; shows capture ID and remaining hold.
   - ESCALATE → step-up card ("Approve this cart?"), Maya approves/declines; decision signed into the ledger.
   - BLOCK → red card with exact violated rule (e.g. "quantity 50 > max 6", "instruction found in product text").
   - Mandate meter: captured / remaining / expires in.
3. **Supplier store** (`/store`, the merchant's agent-facing catalog)
   - Small catalog (~12 SKUs) incl. **one poisoned listing** whose description contains an injection attempt,
     and look-alike SKUs (16-oz cups vs 12-oz cups) for the semantic check.
4. **Evidence Desk** (`/desk`, Daniel)
   - Dispute inbox (from PayPal Disputes API + webhooks), each linked to capture → cart → mandate → ledger.
   - Triage panel: deterministic verdict (WITHIN_MANDATE / MERCHANT_FAULT / NEEDS_REVIEW) + AI explanation of
     how the buyer's claim text relates to the signed record.
   - Evidence packet preview (PDF: mandate summary, signatures valid ✓, cart vs mandate table, ledger excerpt
     with hashes, timeline). One click: **Submit evidence to PayPal** or **Accept claim & refund**.
     Auto-mode toggle (policy-driven) for the demo of "resolves itself"; default is human click.
   - Analytics tab (AG Studio, sponsor): agentic vs non-agentic volume, mandate utilization, dispute outcomes,
     Studio agent Q&A. (Post-Nov-2 scope.)
5. **Verify** (`/verify/[mandateId]`, public)
   - Recomputes signatures and the ledger hash chain server-side and shows ✓/✗ per link. Anyone (a judge) can check.
6. **Demo** (`/demo`)
   - Guided 6-step tour, finishable by a stranger in < 2 minutes. Two lanes: **full lane** (judge approves the hold in the
     real PayPal v6 popup with sandbox buyer credentials from the Devpost testing instructions) and **fast lane** (a
     pre-approved sandbox hold from a pool, clearly labelled). Nightly reset.

## Scenarios (each must work and be scriptable)
- **S1 Happy path:** mandate $200 → agent buys oat milk ×6 + 12-oz cups ×4 boxes → PASS → captures.
- **S2 Injection blocked:** agent reads poisoned listing → proposes 50 units → BLOCK (deterministic C4/C6) and judge flags injection.
  Because a good model may simply ignore the injection, `/demo` also has **"Simulate a hijacked agent"** (labelled) which submits the
  injected cart straight to the gate — proving money is safe even if the agent is fully compromised.
- **S3 Semantic mismatch → step-up:** merchant marks 12-oz cups out of stock → agent substitutes 16-oz cups → judge downgrades to ESCALATE → Maya declines.
- **S4 Dispute, agent within mandate:** buyer disputes capture ("didn't authorize this") → triage WITHIN_MANDATE → evidence submitted via Disputes API.
- **S5 Dispute, merchant fault:** Bean Brew shipped a substitute (fulfillment record mismatches cart) → triage MERCHANT_FAULT → accept claim → refund.
- **S6 Expiry:** mandate window ends → cron voids the remaining authorization → Maya sees "$X released".

## Functional requirements (MUST for v1)
- F1 Mandate compile, edit, sign, verify. F2 Hold creation via JS SDK v6 + authorize. F3 Agent with tools
  `search_catalog`, `get_product`, `propose_cart` only. F4 Policy Gate (deterministic + veto-only judge). F5 Partial
  capture service with DB-enforced cap and idempotency. F6 Step-up approvals. F7 Ledger (hash chain + signatures)
  and `/verify`. F8 Webhook ingest with signature verification. F9 Dispute ingest, triage, evidence PDF,
  provide-evidence, accept-claim. F10 Expiry void cron. F11 `/demo` + reset. F12 Hosted on Vercel Hobby + Neon free until Dec 15 (zero cost). F13 Graceful, labelled AI fallbacks when free quotas run out (invariant 9).

## SHOULD (after core loop gate, Nov 2)
- S-1 AG Studio analytics + Studio agent. S-2 Passkey signing of mandates. S-3 Reauthorize flow for long mandates.
- S-4 Payment-mandate-like metadata on captures (`custom_id`/`note_to_payer`). S-5 Email/receipt view with mandate link.

## Non-goals (say no)
Multi-merchant mandates (needs PayPal Commerce Platform), real Agent Ready/Store Sync, full W3C VC/DID stack,
mobile app, user accounts/auth beyond demo roles, internationalization, a second agent framework, vector search.

## Success metrics (for the video/README, measured in our sandbox runs)
- 0 captures ever exceed the hold (property test + DB constraint).
- 100% of captures have a ledger entry with a valid signature and chain link.
- Injection scenario blocked in 10/10 runs.
- Dispute → evidence submitted in < 60 s of the dispute appearing (excluding PayPal sandbox latency).
