# 09 — AI / Automation Layer


## 1. Purpose

This section defines:

- Why AI exists in DevPilot
- Where AI is allowed to operate
- What AI can access
- What AI cannot mutate
- How cost is controlled
- How determinism is protected
- How failure is isolated

AI is treated as:

A cost-bearing, unreliable, non-authoritative external dependency.

It is assistive — never authoritative.


---

## 2. Role of AI in DevPilot

AI is used to:

- Suggest structured draft content
- Surface potential edge cases
- Highlight missing considerations
- Assist regeneration during iteration

AI is NOT used to:

- Lock steps
- Complete revisions
- Mutate locked data
- Override validation rules
- Determine export eligibility
- Modify subscription state

AI output is advisory only.


---

## 3. Authority Boundary

Strict rule:

AI output is never authoritative.

All AI-generated content must:

- Be stored as Draft
- Be explicitly reviewed by user
- Be explicitly locked by user

Only user-triggered lock transitions change system state.

AI cannot auto-lock steps.


---

## 4. Architectural Placement

AI lives outside the trust boundary.

Flow:

Client → Backend → AI Provider → Backend → Client

Rules:

- Client never calls AI directly
- Backend constructs prompt
- Backend scopes context
- Backend validates response
- Backend enforces quota before invocation

AI is fully mediated.


---

## 5. AI Input Scope Rules

AI may receive:

- Current step draft input
- Previously Locked steps (read-only context)
- Structured user-provided fields

AI must NOT receive:

- Other users’ data
- Entire database contents
- Subscription secrets
- Internal quota counters
- Hidden system metadata
- Plan configuration internals

Context must be minimal and scoped.


---

## 6. AI Output Handling Rules

When AI returns output:

- Backend validates structure
- Output stored as Draft only
- No automatic lock
- No automatic revision completion
- No downstream invalidation

AI output must pass standard validation before lock.

AI suggestions are treated the same as manual drafts.


---

## 7. AI Cost Control Model

AI is cost-bearing.

Controls required:

- Per-user AI suggestion quota
- Per-user AI regeneration quota
- Per-period usage tracking
- Regeneration rate limits
- Request size limits

Quota rules:

- Quota check occurs BEFORE AI call
- Usage increment occurs AFTER successful response
- Quota exhaustion results in rejection
- No partial mutation on quota failure


---

## 8. AI Regeneration Model

Regeneration allowed only if:

- SubscriptionState = PaidActive
- Regeneration quota available
- Step is eligible (Draft or Invalidated)

Regeneration must:

- Produce Draft output
- Not mutate Locked steps
- Not modify RevisionStatus
- Not bypass validation rules


---

## 9. AI Failure Handling

AI may fail due to:

- Timeout
- Rate limiting
- Provider outage
- Malformed output

System behavior:

- Step remains Draft
- No state mutation
- No partial writes
- Error returned to user
- User may retry

AI failure must never corrupt blueprint state.


---

## 10. Determinism Protection

Export must never:

- Call AI
- Re-evaluate AI suggestions
- Depend on live AI behavior

Export must depend only on:

- Locked step_data
- Revision metadata
- Export template version

This guarantees reproducibility.


---

## 11. Data Exposure & Security Risks

AI introduces risk:

- Data leakage
- Prompt injection
- Overexposure of internal structure

Mitigations:

- Minimal context exposure
- No cross-user data
- No secret tokens in prompts
- Output sanitization
- Structured prompt construction server-side
- Validation before persistence


---

## 12. AI Abuse Prevention

Abuse scenarios:

- User spams regeneration
- User attempts large context extraction
- User uses AI as storage mechanism

Mitigations:

- Strict per-period quotas
- Per-request input size limits
- Step-scoped context only
- Rate limiting
- Storage caps enforced at step level


---

## 13. AI & Subscription Interaction

AI permissions depend on subscription state.

Free:

- Limited suggestion quota
- No regeneration

PaidActive:

- Suggestion quota
- Regeneration quota
- Higher usage limits

PaidExpired:

- Treated as Free for AI permissions

All AI permission checks enforced server-side.


---

## 14. AI Invariants

The following must ALWAYS hold:

- AI cannot mutate Locked steps.
- AI cannot complete revision.
- AI cannot invalidate downstream steps.
- AI failure cannot corrupt state.
- AI usage must be metered.
- AI output must be explicitly locked by user.
- Export must not depend on AI at runtime.

AI is assistive, never authoritative.

