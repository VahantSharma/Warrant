# ADR 001 — One Next.js app (TypeScript) on Render
Date: 2026-10-22 · Status: accepted (hosting superseded by ADR 008 → Vercel Hobby + Neon free)

## Context
Solo builder, ~21 days, part-time. Judges read code and open a hosted URL. PayPal JS SDK, AG Studio (React) and the Vercel AI SDK are all first-class in the JS ecosystem. The owner works in JS/Node daily.

## Decision
Single Next.js (App Router) app; route handlers for the API; Postgres via Drizzle. Deploy to Render (web service + Postgres + cron). Render credits ($50) from the hackathon cover a non-sleeping instance.

## Consequences
+ One deploy, one language, shared zod types between UI and server.
+ Render cron covers expiry voids and nightly demo reset.
− Long-running agent loops must stay within request limits → stream via route handlers; keep runs short (agent max steps).
