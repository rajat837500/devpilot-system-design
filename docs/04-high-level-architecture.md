# 04 — High-Level Architecture


## 1. Purpose

This section defines the structural architecture of DevPilot.

It answers:

- What are the major system components?
- Where are the trust boundaries?
- What is authoritative?
- Where does AI sit?
- Where does subscription logic live?
- What is synchronous vs asynchronous?

This is system shape — not implementation detail.


---

## 2. System Context (Top-Level View)

DevPilot consists of five logical components:

1. Web Client
2. Application Backend (Authoritative Core)
3. Primary Data Store
4. AI Provider (External Dependency)
5. Subscription/Billing Provider (External Dependency)

The backend is the sole authority for all state transitions.


---

## 3. Trust Boundaries

DevPilot operates across three trust zones.

### 3.1 Zone 1 — User Environment (Untrusted)

Includes:
- Browser
- Local state
- Client-side caches

Never trusted for:
- Subscription validation
- Step locking enforcement
- Quota enforcement
- Revision state transitions
- Export eligibility

Client is presentation + request initiator only.


---

### 3.2 Zone 2 — Application Backend (Trusted Core)

This is the deterministic state machine.

Responsible for:

- Authentication validation
- Authorization enforcement
- Subscription state validation
- Quota enforcement
- Project lifecycle control
- Revision lifecycle control
- Step locking & restart logic
- Export eligibility validation
- Export artifact generation
- AI mediation
- Atomic state transitions

All invariants are enforced here.


---

### 3.3 Zone 3 — External Dependencies (Untrusted but Integrated)

Includes:

- AI provider
- Billing/subscription provider

These systems:

- Are not authoritative
- May fail
- Must not mutate blueprint state directly
- Are isolated behind backend mediation

The client never communicates directly with external dependencies.


---

## 4. Component Responsibilities


### 4.1 Web Client

Responsibilities:

- Render logical surfaces
- Collect structured user input
- Display validation feedback
- Display quota usage
- Initiate backend actions

Non-responsibilities:

- Cannot enforce step lock rules
- Cannot validate subscription
- Cannot determine export eligibility
- Cannot perform state transitions


---

### 4.2 Application Backend (Core Authority)

The backend is a modular monolith with clear domain boundaries.

Responsibilities:

- Authenticate user identity
- Validate ownership
- Enforce subscription gating
- Enforce quota limits
- Execute atomic state transitions
- Maintain revision determinism
- Enforce sequential step progression
- Mediate AI calls
- Generate deterministic exports

The backend owns all business invariants.


---

### 4.3 Primary Data Store

Stores:

- Users
- Subscriptions
- Plan definitions
- Projects
- Revisions
- Steps
- Export records
- Usage records

Requirements:

- Transactional consistency
- Referential integrity
- Deterministic reads
- Atomic writes

The database does not enforce business rules.
The backend enforces invariants.


---

### 4.4 AI Provider (External)

Position:

- Outside trust boundary
- Invoked only by backend

Rules:

- AI cannot mutate stored state directly
- AI output is treated as Draft only
- AI cannot lock steps
- AI failure must not corrupt blueprint state

AI interactions are synchronous from user perspective,
but isolated behind backend mediation.


---

### 4.5 Subscription/Billing Provider (External)

Responsibilities:

- Billing
- Subscription lifecycle reporting

Backend Responsibilities:

- Store subscription state snapshot
- Validate subscription before privileged actions
- Handle downgrade behavior
- Prevent state mutation on billing events

Billing provider never controls blueprint state.


---

## 5. Data Ownership Model

Ownership hierarchy:

User
→ Project
→ Revision
→ Step

Rules:

- Every entity belongs to exactly one user
- No cross-user references
- No shared projects
- No shared revisions
- Soft delete does not break referential integrity

This simplifies authorization and prevents data leakage.


---

## 6. State Authority Model

All state transitions must occur in backend.

Examples:

### Step Lock

Client requests → Backend validates:

- Previous step locked?
- Validation rules satisfied?
- Subscription state valid?
- Quota available?
- Step not already locked?

If valid → Transition executed atomically.


---

### Restart-from-Step

Client requests → Backend:

- Validates PaidActive subscription
- Validates step position
- Performs atomic cascade:
    - Steps K–N → Draft
    - locked_at cleared
    - invalidated_at set
    - Revision → Draft
    - Export disabled

No partial transitions allowed.


---

## 7. Sync vs Async Boundaries

### Synchronous Operations

- Save draft
- Lock step
- Restart-from-step
- Create revision
- Trigger export
- AI suggestion request

### Asynchronous Possibilities (Future Scope)

- Export generation for large artifacts
- Subscription webhook reconciliation
- AI retry processing

MVP assumes request-response simplicity.


---

## 8. Determinism Guarantee

Export artifacts must be:

- Fully derived from Locked step data
- Independent of AI at export time
- Revision-scoped
- Immutable once generated

Export must never re-call AI.


---

## 9. Failure Isolation Strategy

### AI Failure

- Step remains Draft
- No state mutation
- User may retry
- No impact on revision integrity


### Subscription Check Failure

- Action rejected
- No state mutation


### Database Failure

- Entire operation rolled back
- No partial restart cascade
- No partial revision completion


---

## 10. Architectural Simplicity Principles

The MVP deliberately avoids:

- Microservices
- Event sourcing
- Distributed locks
- CRDT models
- Branching revision graphs
- Real-time collaboration engines

DevPilot is implemented as:

A structured monolithic backend
with deterministic state machine enforcement.

Complexity is introduced only when justified.

