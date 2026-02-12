# 10 — Operations, Resilience & Production Safety


## 1. Purpose

This section defines how DevPilot behaves in real production conditions.

It covers:

- Authentication & authorization
- Rate limiting
- Failure isolation
- Atomic mutation guarantees
- Logging & observability
- Data recovery
- Deployment discipline
- Security model

This is operational philosophy, not tooling specification.


---

## 2. Authentication Strategy

### Model

- Backend-issued secure session tokens
- HttpOnly, secure cookies (or equivalent)
- Short-lived access tokens
- Server-side session validation

### Requirements

- Authentication required for all state mutations
- Every request must include validated user identity
- No trust in client-provided identifiers
- Ownership validated on every resource access

### Invariants

- Identity validated before business logic
- Unauthorized requests must not reach mutation layer


---

## 3. Authorization Strategy

Authorization is:

- Centralized
- Server-side
- Subscription-aware
- Quota-aware

### Enforcement Order

Every privileged action must follow:

1. Authenticate user
2. Validate ownership
3. Validate subscription state
4. Validate quota limits
5. Validate state invariants
6. Execute atomic mutation

Failure at any stage → reject with no mutation.


---

## 4. Rate Limiting Strategy

Rate limiting required for:

- AI requests
- Restart-from-step
- Project creation
- Authentication attempts
- Account creation

### Principles

- Per-user limits
- IP-level limits for auth endpoints
- Period-based AI limits
- Reject before expensive operation

Rate limiting protects:

- Cost exposure
- Abuse vectors
- System stability


---

## 5. Atomicity & Transaction Safety

The following operations MUST be atomic:

- Step lock
- Restart-from-step cascade
- Revision completion
- Project creation
- Subscription state update
- Export generation record

### Rules

- No partial restart cascade
- No partial revision completion
- No partial export state
- Entire transaction rolled back on failure

Atomic integrity is mandatory.


---

## 6. Logging & Observability

The system must log:

### Security Events

- Login attempts
- Failed authentication
- Ownership violations
- Quota violations
- Subscription transitions

### State Mutations

- Step lock
- Restart-from-step
- Revision completion
- Project soft delete
- Export generation

### AI Events

- AI call initiated
- AI call failed
- AI quota exceeded

### Logging Rules

- No sensitive step data in logs
- No secret tokens logged
- Structured logs for traceability
- Correlation IDs for mutation tracking


---

## 7. Failure Isolation Strategy

### AI Failure

- Does not block manual editing
- Does not corrupt state
- Does not mutate locked steps
- User may retry

### Billing Provider Failure

- Use last known subscription snapshot
- Do not mutate blueprint state
- Reconcile via webhook when available

### Database Failure

- Reject mutation
- Roll back entire transaction
- Never allow partial restart or revision completion

Consistency favored over convenience.


---

## 8. Backup & Recovery Model

System must support:

- Regular database backups
- Point-in-time recovery (if available)
- Restore without breaking referential integrity

Restore must preserve:

- Project hierarchy
- Revision immutability
- Step state integrity
- Export reproducibility

Data recovery must not violate invariants.


---

## 9. Deployment & Migration Discipline

### CI/CD Requirements

- All migrations backward-compatible
- No destructive schema change without review
- Automated invariant testing

Test coverage must include:

- Step state machine enforcement
- Restart-from-step atomicity
- Subscription downgrade behavior
- Export eligibility logic

### Production Deployment Must

- Prevent schema drift
- Support rollback
- Preserve determinism


---

## 10. Monitoring & Alerting (Intent-Level)

System should monitor:

- AI failure rate
- Export failure rate
- Step lock rejection rate
- Subscription webhook failures
- Quota rejection spikes
- Database error rates

Alert on abnormal patterns.

Monitoring protects both cost and correctness.


---

## 11. Manual Recovery Strategy

Admin-only capabilities may include:

- Subscription reconciliation
- SoftDeleted project restoration
- Revision activation correction (if invariant violation)
- Controlled repair scripts

Admin operations must never violate global invariants.


---

## 12. Security Model

Core protections:

- Strict ownership validation
- Parameterized queries
- No cross-user queries
- Server-side quota enforcement
- Strict structured validation on step_data
- AI prompt sanitization
- Output sanitization before persistence

Security must align with state integrity.


---

## 13. Production Safety Invariants

The following must ALWAYS hold in production:

- No partial restart cascade
- No partial revision completion
- No export without completed revision
- Subscription state change never mutates blueprint state
- SoftDeleted project cannot accept mutations
- Quota validated before mutation
- Logs never expose sensitive blueprint content
- Deterministic export reproducibility preserved

Operational integrity must not compromise architectural integrity.

