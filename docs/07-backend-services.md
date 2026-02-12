# 07 — Backend Services & Responsibility Ownership


## 1. Purpose

This section defines logical backend service boundaries.

These are not microservices.
DevPilot MVP uses a modular monolithic backend.

Each service defines:

- What it owns
- What it validates
- What it is forbidden to mutate
- Where authority lives

This prevents state corruption and circular responsibility.


---

## 2. Service Structure Overview

The backend consists of the following logical services:

1. Authentication & Identity Service
2. Subscription & Billing Service
3. Project Service
4. Revision Service
5. Step Service (Core State Machine)
6. Export Service
7. AI Mediation Service
8. Quota & Usage Service

Each service owns a clearly defined responsibility domain.


---

## 3. Authentication & Identity Service

### Owns

- User authentication validation
- Session validation
- Identity extraction
- Ownership checks

### Enforces

- A user may only access their own resources
- No cross-user data access

### Does NOT

- Enforce subscription logic
- Mutate project or revision state
- Enforce quota

Identity must be validated before any business logic executes.


---

## 4. Subscription & Billing Service

### Owns

- Subscription state snapshot
- Plan resolution
- Subscription transitions
- Downgrade behavior (permission gating only)

### Enforces

SubscriptionState at action time.

PaidActive required for:

- Export
- Restart-from-step
- New revision creation
- Exceeding Free project limits

### Invariants

Subscription state change must never mutate:

- Project state
- Revision state
- Step state

This service gates actions, not data.


---

## 5. Project Service

### Owns

- Project creation
- Project soft deletion
- Active vs SoftDeleted state
- Project-level quota enforcement

### Validates Before Mutation

- Ownership
- Subscription state
- Quota limits
- Project not SoftDeleted (for mutation)

### Enforces

- Free total project cap
- Paid active project cap
- SoftDeleted project mutation restrictions

### Does NOT

- Modify revision internals
- Modify step state

ProjectService manages only project-level state.


---

## 6. Revision Service

### Owns

- Revision creation
- Active revision selection
- Revision status transitions
- Immutable Completed revision enforcement

### Validates Before Mutation

- Subscription state (for new revision)
- ProjectStatus = Active
- Ownership

### Enforces

- Only one active revision per project
- Completed revision immutability
- RevisionStatus aligns with step states

### Does NOT

- Mutate individual step state directly
- Enforce AI quota

Step-level transitions are delegated to StepService.


---

## 7. Step Service (Core State Machine Owner)

This is the most critical service in DevPilot.

### Owns

- Step draft updates
- Step lock transitions
- Restart-from-step cascade
- Step invalidation
- Sequential enforcement

### Enforces

- Cannot lock Step N if Step N-1 not Locked
- Locked steps immutable
- Restart invalidates downstream steps atomically
- RevisionStatus consistent with step states
- Validation required before lock

### Atomic Operations

- Lock step
- Restart-from-step cascade
- Revision completion

StepService is the guardian of deterministic progression.


---

## 8. Export Service

### Owns

- Export eligibility validation
- Export artifact generation
- Export record persistence

### Enforces

Export allowed only if:

- ProjectStatus = Active
- RevisionStatus = Completed
- All Steps = Locked
- SubscriptionState = PaidActive

### Guarantees

- Export does not mutate revision
- Export tied to specific revision ID
- Export deterministic and reproducible
- Export never invokes AI

ExportService does not modify step or revision state.


---

## 9. AI Mediation Service

AI is never directly accessible from client.

### Owns

- Prompt construction
- Context scoping
- AI quota enforcement
- AI response validation
- Failure isolation

### Enforces

- Subscription-based AI permissions
- Per-period quota limits
- AI output stored only as Draft
- AI cannot auto-lock steps
- AI cannot mutate Locked state

### Failure Handling

If AI fails:

- No state mutation
- No partial write
- Error returned
- User may retry

AI Service is stateless relative to blueprint authority.


---

## 10. Quota & Usage Service

### Owns

- Active project counting
- Free total project counting
- AI usage tracking
- Restart-from-step usage tracking
- Storage usage tracking

### Enforces

- Quota check BEFORE mutation
- No partial state mutation on quota failure
- Usage increment AFTER successful operation

### Must Be Consulted Before

- Project creation
- Revision creation (if gated)
- Restart-from-step
- AI request
- Storage-heavy mutation

Quota enforcement is centralized to prevent bypass.


---

## 11. Cross-Service Enforcement Order

Every privileged action follows this order:

1. Authenticate user
2. Verify ownership
3. Validate subscription state
4. Validate quota limits
5. Validate state invariants
6. Execute atomic mutation
7. Persist changes
8. Increment usage counters

Failure at any stage → no mutation.


---

## 12. Error Ownership Boundaries

| Failure Type | Responsible Service |
|--------------|--------------------|
| Invalid credentials | Authentication |
| Ownership violation | Authentication |
| Subscription inactive | Subscription Service |
| Quota exceeded | Quota Service |
| Step validation failure | Step Service |
| Revision invalid state | Revision Service |
| Export ineligible | Export Service |
| AI failure | AI Service |

Each service owns its domain errors.


---

## 13. Architectural Principles

- Backend is sole authority
- No client-side enforcement
- No circular service ownership
- Deterministic state transitions
- Subscription gates permission, not data
- AI is advisory only
- Export isolated from mutation
- Modular monolith intentionally chosen

No premature microservices.

