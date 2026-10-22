# Spikes — prove PayPal sandbox behaviour before building on it

Each spike is a standalone script (`pnpm tsx scripts/spikes/NN-name.ts`) that talks to the real sandbox,
prints every PayPal response's `debug_id`, and ends with a PASS/FAIL summary. Record results in
`docs/research/SANDBOX_FACTS.md` (findings log, newest first). Spikes are throwaway code but must not leak secrets.

| # | Script | Proves | Gate |
|---|---|---|---|
| 01 | `01-hold-and-captures.ts` | AUTHORIZE order → buyer approval (prints approve link; owner approves as sandbox buyer) → authorize → 2 partial captures (`final_capture:false`) → void remainder. Also: capture > remaining error code, replay with same `PayPal-Request-Id`, duplicate `invoice_id` behaviour | 1 |
| 02 | `02-disputes.ts` | Open dispute on our capture (try API sandbox create; else owner opens one from the sandbox buyer's Resolution Center) → poll until visible (measure latency) → show → provide-evidence (multipart PDF) → second dispute accept-claim → sandbox settle | 1 |
| 03 | `03-js-sdk-v6.html` + route | v6 init with server client token, button renders, approves an AUTHORIZE order; server authorizes | 1 |
| 04 | `04-webhooks.ts` | Real events arrive at tunnel/Render URL; signature verification passes; dedupe by event id | 1 |
| 05 | `05-agent-toolkit.ts` | List tools in `@paypal/agent-toolkit` (AI SDK); confirm read-only subset config for disputes/orders/transactions | 2 |

Rule: if a spike FAILS on something demo-critical (02 especially), stop and tell the owner the same day with options.
