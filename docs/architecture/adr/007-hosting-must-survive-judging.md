# ADR 007 — Hosting must survive until Dec 15 (no expiring/sleeping free tiers)
Date: 2026-10-22 · Status: accepted

## Context
Judging runs Dec 1–15. Free web tiers sleep (cold starts lose judges). Some free Postgres tiers expire after ~30 days — a DB created around Nov 1 could vanish right as judging starts.

## Decision
Use the hackathon's Render credits for a non-sleeping web instance and a non-expiring Postgres plan (or Neon free tier if Render's free DB would expire). Add uptime monitoring and a nightly demo reset.

## Consequences
+ Demo reliability during judging. − Small credit spend; budget it.
