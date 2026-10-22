# ADR 003 — The LLM can veto, never authorize
Date: 2026-10-22 · Status: accepted

## Decision
Money moves only when deterministic checks pass (and a human approved, when step-up applies). The intent-match judge can downgrade PASS→ESCALATE or PASS→BLOCK, never upgrade. Judge errors/timeouts → ESCALATE. The shopping agent has no tool that moves money; the Evidence Desk's accept/provide decisions are deterministic triage outcomes, with the LLM writing explanations only.

## Consequences
+ Safe by construction; a strong sentence for judges ("the model can veto but never authorize").
+ Prompt injection can at worst cause a block or an escalation, never a payment.
− Some false escalations; acceptable — merchants in PayPal's survey prefer human-in-the-loop.
