# ADR 008 — Zero-cost stack: Gemini + Groq free tiers, Vercel Hobby, Neon free
Date: 2026-10-25 · Status: accepted · Supersedes: hosting parts of ADR 001 and ADR 007; Claude-API assumption in early docs

## Context
Owner decision (Oct 25): no money spent on APIs or hosting. Judges need a hosted demo that works free of charge until Dec 15
and have no API keys of their own (manager's forum answer). Free LLM tiers have rate limits; free hosts have sleep/expiry traps.

Facts checked Oct 25, 2026 (re-verify in the consoles; quotas change):
- **Google Gemini API free tier:** no card; current Flash/Flash-Lite models free (e.g. `gemini-3.8-flash`, `gemini-3.6-flash`,
  `gemini-3.5-flash-lite` on the pricing page). Google no longer publishes a central per-model table; community estimates
  ~10 RPM / ~1,500 RPD for Flash — read the real numbers at aistudio.google.com/rate-limit. Daily reset at midnight PT.
  **Free-tier data may be used to improve Google's products** → only fictional/sandbox data goes in (it does).
- **Groq free tier:** e.g. `openai/gpt-oss-120b`, `openai/gpt-oss-20b`, `qwen/qwen3.8-27b` at ~30 RPM, 1K RPD, **8K TPM**
  (small token budget per minute → use Groq for short prompts: the judge and narrative, not big catalog contexts).
- **Vercel Hobby:** free; function max duration 300 s; **cron jobs at most once per day, ±59 min precision**; runtime logs kept 1 h;
  can't connect to Git repos owned by an org (use the owner's personal repo); non-commercial use (a hackathon demo qualifies).
- **Neon free:** permanent plan, no card; 100 CU-hours/project/month; 1 GB storage per project; scale-to-zero after 5 min (brief resume).
- **Render:** free web services sleep (cold starts lose judges); credits route likely needs a card → not used.

## Decision
- AI: Vercel AI SDK with `@ai-sdk/google` (primary) and `@ai-sdk/groq` (fallback) behind `src/lib/ai/` — provider registry,
  ordered fallback on 429/5xx/timeouts, self-imposed daily call caps stored in Postgres, response cache for the seeded demo path,
  `fixture` provider for tests. Model IDs from env.
- Hosting: Vercel Hobby (Next.js native). Database: Neon free (pooled URL at runtime, direct URL for migrations).
- Cron limits: expiry handled **lazily** (any read of a mandate past `validUntil` triggers the void path) + a daily cron sweep +
  an "End mandate now" button. Dispute sync: webhook + on-demand "Sync with PayPal" + sync on Desk page load. Nightly demo reset = daily cron.
- Rate limiting: Postgres-based counters (no Redis).
- Everything else free: PayPal sandbox, GitHub, AG Grid 45-day trial (start Nov 1), UptimeRobot free, YouTube, OBS for recording.

## Consequences
+ $0 to build and to keep alive through judging. No card anywhere.
+ Provider-agnostic AI layer is itself good engineering (and protects against one provider's outage on judging day).
− Free models are less capable than frontier paid models → mitigated by design: deterministic checks do the heavy lifting
  (ADR 003), structured outputs validated with zod + one repair retry, small focused prompts, and an eval suite (`pnpm eval`)
  that gates prompt changes.
− Quota exhaustion is possible during judging → invariant 9: fail closed + honest labelled fallbacks; caps set below quotas.
− Daily-only cron → lazy expiry; documented.
− Losing the (credits-only) Render sponsor prize — irrelevant; we target AG Grid.
