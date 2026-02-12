# 06 — System State Model & Invariants


## 1. Purpose

This section formally defines:

- Core entity states
- Allowed transitions
- Forbidden transitions
- Atomic mutation rules
- Global invariants

All state transitions must be deterministic,
validated server-side,
and executed atomically.


---

## 2. Core Entity Hierarchy

User
→ Subscription
→ Project
→ Revision
→ Step

Each layer has independent state,
but must preserve referential and behavioral integrity.


---

## 3. Subscription State Model

Each user has exactly one Subscription record.

### SubscriptionState

- Free
- PaidActive
- PaidExpired

### 3.1 Free

- User has never subscribed
- Free-tier limits enforced
- Mutation permissions restricted
- One total project lifetime (SoftDeleted included)

### 3.2 PaidActive

- Active subscription
- Full blueprint lifecycle access
- Restart-from-step enabled
- Export enabled
- Revision creation enabled
- Concurrent project cap enforced
- Period-based quota enforced

### 3.3 PaidExpired

- Previously paid, now expired
- Behaves like Free for mutation permissions
- Retains full read access to historical data
- No state mutation occurs on downgrade

### Subscription Invariants

- Exactly one subscription per user
- Plan change must not mutate:
    - Project state
    - Revision state
    - Step state
- Permission checks evaluated at action time
- Downgrade affects future actions only


---

## 4. Project State Model

Each Project has:

ProjectStatus:
- Active
- SoftDeleted

### Allowed Transitions

Active → SoftDeleted  
SoftDeleted → Active (optional restore)

### Forbidden Transitions

- Hard delete (MVP)
- Mutation of SoftDeleted project
- Export from SoftDeleted project

### Project Invariants

- Project belongs to exactly one user
- Free quota based on total projects (SoftDeleted included)
- Paid quota based on concurrent Active projects
- SoftDeleted projects:
    - Cannot accept new revisions
    - Cannot be exported


---

## 5. Revision State Model

Each Revision has:

RevisionStatus:
- Draft
- Completed

Each project may have multiple revisions,
but only one revision may have:

is_active = true

### Allowed Transitions

Draft → Completed  
Completed → Draft (via restart-from-step only)  
Completed → New Draft Revision (clone)

### Forbidden Transitions

- Direct edit of Completed revision
- Multiple active revisions per project
- Completed if any Step ≠ Locked

### Revision Invariants

- If any Step ≠ Locked → RevisionStatus = Draft
- Completed revision must have all Steps Locked
- Completed revision is immutable unless restart invoked
- Revision creation does NOT consume project quota


---

## 6. Step State Model

Each Step has:

StepStatus:
- Draft
- Locked
- Invalidated

Each step also tracks:

- locked_at (nullable)
- invalidated_at (nullable)

### Allowed Transitions

Draft → Locked  
Locked → Invalidated (via upstream restart)  
Invalidated → Locked (after re-validation)

### Forbidden Transitions

- Locked → Draft (direct edit forbidden)
- Lock Step N if Step N-1 ≠ Locked
- Invalidated → Completed without re-lock

### Step Invariants

- Sequential progression enforced
- Locked steps immutable
- AI output cannot auto-lock step
- Step lock requires validation pass


---

## 7. Restart-from-Step Formal Rule

When restarting from Step K:

For each Step i ≥ K:

- StepStatus → Draft
- locked_at → null
- invalidated_at → current timestamp

For each Step i < K:

- No change

Additionally:

- RevisionStatus → Draft
- Export eligibility → disabled

Restart must execute atomically.
No partial cascade allowed.


---

## 8. Export State Model

Export allowed only if:

- ProjectStatus = Active
- RevisionStatus = Completed
- All Steps = Locked
- SubscriptionState = PaidActive

Export:

- Does not mutate revision
- Is tied to specific revision ID
- Must be reproducible from locked data
- Remains valid even after downgrade

Export must not call AI.


---

## 9. Quota Enforcement Model

Quota types:

- Free total project limit
- Paid active project limit
- AI suggestion quota
- AI regeneration quota
- Restart-from-step quota
- Storage limit

Quota Rules:

- Quota check occurs BEFORE state mutation
- Usage increments AFTER successful operation
- Quota failure results in no partial mutation
- Revision creation does not consume project quota
- AI usage only increments on successful response


---

## 10. Idempotency Guarantees

The following operations must be idempotent:

- Step Lock
- Restart-from-Step
- Export generation
- Revision completion

Repeated identical requests must not produce inconsistent state.


---

## 11. Concurrency Safety

MVP assumptions:

- Single-user per project
- No collaborative editing

However, system must guard against:

- Rapid duplicate lock requests
- Lock during restart
- Concurrent restart attempts

Mitigation:

- Transactional state transitions
- Validation against persisted state
- Optional optimistic concurrency control (e.g., version column)

No distributed locking required.


---

## 12. Global Non-Negotiable Invariants

The following must ALWAYS hold:

- Locked steps cannot be edited directly.
- Completed revision contains no Draft or Invalidated steps.
- Restart invalidates downstream steps atomically.
- Export requires Completed revision and PaidActive subscription.
- Subscription state changes never mutate blueprint state.
- SoftDeleted project cannot accept mutations.
- No step may be Locked unless previous step is Locked.
- Only one active revision per project.
- Quota checks occur before mutation.
- State transitions are atomic and deterministic.

These invariants define system integrity.

