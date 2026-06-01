# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-06-01 — validateRequired as semantic root in AgentDescriptorValidator

**Priority:** low  
**Status:** active

Move validation logic directly into `validateRequired` and make `validateOptional` a guard-then-call wrapper (`if (value == null) return; validateRequired(...)`). Eliminates delegation indirection through the private `validateField`, makes `validateRequired` the semantic root of all string validation, and removes the need for the comment added in eidos#24. Low cost — internal to `AgentDescriptorValidator`, no callers affected.

**Context:** Surfaced during code review of `issue-024-review-followups` (eidos#24). The reviewer flagged `validateRequired`'s delegation to `validateField` as indirection that only needs a comment to explain — suggesting the structural fix would remove the need for the comment entirely.

**Promoted to:**
