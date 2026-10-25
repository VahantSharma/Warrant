# ADR 007 — Hosting must survive until Dec 15 (no expiring/sleeping free tiers)
Date: 2026-10-22 · Status: accepted (implementation superseded by ADR 008)

## Context
Judging runs Dec 1–15. Free web tiers sleep (cold starts lose judges). Some free Postgres tiers expire after ~30 days — a DB created around Nov 1 could vanish right as judging starts.

## Decision
Host on Vercel Hobby (serverless: no sleeping instance to wake) and Neon free Postgres (permanent plan, scale-to-zero resumes quickly). Add uptime monitoring (UptimeRobot free, which also keeps things warm) and a nightly demo reset (daily Vercel cron). See ADR 008.

## Consequences
+ Demo reliability during judging at $0. − Daily-only crons (lazy expiry instead); Neon CU-hours must stay under 100/month (monitor in the Neon console).
