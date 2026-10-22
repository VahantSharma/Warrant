# PayPal sandbox facts — VERIFIED vs ASSUMED

Rule: nothing moves from ASSUMED to VERIFIED without a spike script in `scripts/spikes/` and the
date + debug ID recorded here. If a demo-critical assumption fails, escalate to the owner the same day.

## VERIFIED (from docs / forum, still re-verify in our own app)
| Fact | Source | Our own spike |
|---|---|---|
| Orders v2 `intent: AUTHORIZE` places a hold valid 29 days; 3-day honor period; reauthorize days 4–29 | PayPal docs (auth-capture, delayed capture) | TODO spike-01 |
| Multiple partial captures up to the authorized total; `final_capture: false` keeps the auth open | PayPal docs; hackathon forum post (Oct 2026) says it works in sandbox | TODO spike-01 |
| **Sandbox authorizations don't expire after 29 days** (good: holds survive judging) | docs.paypal.ai delayed-capture | TODO |
| Merchants can create disputes in sandbox; sandbox-only helper ops exist (e.g. require-evidence, settle/adjudicate) | Customer Disputes v1 reference | TODO spike-02 |
| Provide-evidence works when status is `WAITING_FOR_SELLER_RESPONSE`; multipart request; only if the `provide-evidence` HATEOAS link is present | Disputes docs | TODO spike-02 |
| Accept-claim closes the dispute in the buyer's favor and refunds from merchant | Disputes docs | TODO spike-02 |
| Vaulting PayPal wallet may be refused for hackathon apps (`NOT_ENABLED_TO_VAULT_PAYMENT_SOURCE`) → **we do not depend on vaulting** | Hackathon forum | n/a |
| Sandbox access tokens expire (hours) → server fetches/refreshes via client credentials; never paste tokens | AI-Toolkit README | built into client |
| Remote sandbox MCP: `https://mcp.sandbox.paypal.com` (`/sse` or `/http`), client-credential auth supported | PayPal MCP docs | optional |

## ASSUMED — must be spiked
1. **spike-01 hold & captures:** create AUTHORIZE order → approve as sandbox buyer → authorize → capture $40 (`final_capture:false`) → capture $46.40 → void remainder. Record: any cap on number of partial captures; error code when capture > remaining; idempotent replay behaviour with same `PayPal-Request-Id`; whether `invoice_id` must be unique per capture (expect yes → DUPLICATE_INVOICE_ID); **whether PayPal permits capturing more than the authorized amount** (docs may allow over-capture up to a percentage/fixed limit) — if yes, the DB constraint is the hard cap and the "PayPal never reserved more" wording must change.
2. **spike-02 disputes:** how to open a dispute on *our* capture in sandbox (options: API create in sandbox; sandbox buyer Resolution Center at sandbox.paypal.com; referred disputes). Measure **latency** until it appears in `GET /v1/customer/disputes` and until `CUSTOMER.DISPUTE.CREATED` webhook fires. Then: provide-evidence (multipart with PDF), accept-claim on a second dispute, sandbox settle/adjudicate. **This is demo-critical. If latency is hours, pre-create disputes before recording and say so honestly in README.**
3. **spike-03 JS SDK v6:** load v6 core script with a server-generated browser-safe client token; render PayPal button; approve an order whose server-side intent is AUTHORIZE; confirm server can then call `/authorize`. Record exact init steps here (v6 differs from v5 — read docs via Context7/docs.paypal.ai, don't trust memory).
4. **spike-04 webhooks:** subscribe to `CHECKOUT.ORDER.APPROVED`, `PAYMENT.AUTHORIZATION.CREATED`, `PAYMENT.AUTHORIZATION.VOIDED`, `PAYMENT.CAPTURE.COMPLETED`, `PAYMENT.CAPTURE.REFUNDED`, `CUSTOMER.DISPUTE.CREATED`, `CUSTOMER.DISPUTE.UPDATED`, `CUSTOMER.DISPUTE.RESOLVED`. Verify signatures via `/v1/notifications/verify-webhook-signature`. Note: events fired from the dashboard **simulator** are not verifiable — test with real sandbox actions.
5. **spike-05 Agent Toolkit:** which dispute/order/transaction tools exist in `@paypal/agent-toolkit` for the AI SDK; confirm we can enable a **read-only subset** (list/get disputes, get order, list transactions) and keep money-moving tools disabled.

## Findings log (append; newest first)
<!-- YYYY-MM-DD · spike-NN · result · debug_id(s) · implication -->
