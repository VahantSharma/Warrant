# ADR 004 — AP2-shaped mandates, signed as JWS (Ed25519), not a full W3C VC stack
Date: 2026-10-22 · Status: accepted

## Context
AP2 expresses Intent/Cart/Payment mandates as W3C Verifiable Credentials. A full VC/DID stack is weeks of work and invisible in a 3-minute video.

## Decision
Model mandates on AP2's Intent and Cart mandates (field names and semantics), canonicalize with RFC 8785 (JCS), and sign with Ed25519 via `jose` (JWS). User key server-held in v1 (stretch: WebAuthn passkey over the canonical hash). Agent has its own key and signs cart proposals.

## Consequences
+ Verifiable, legible, quick. `/verify` page demonstrates it.
− Must never claim "AP2-compliant"; we say "AP2-shaped". Documented as limitation + next step.
