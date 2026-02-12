# 05 — Product Workflows & Failure Paths


## 1. Purpose

This section defines how DevPilot behaves under:

- Normal usage
- State transitions
- Subscription changes
- Quota exhaustion
- AI failure
- Validation failure

This is behavioral modeling — not API modeling.

The goal is to ensure all state transitions remain deterministic and safe.


---

## 2. Workflow 1 — Paid User Creates & Completes Blueprint

### Scenario

1. User subscribes (PaidActive).
2. User creates new project.
3. System creates initial revision (Draft).
4. User completes steps sequentially.
5. User locks each step after validation.
6. Final step locked.
7. Revision transitions to Completed.
8. Export becomes eligible.
9. User generates export artifact.

### Required Guarantees

- Cannot access Step N if Step N-1 not Locked.
- Locking requires validation.
- Export allowed only if:
    - RevisionStatus = Completed
    - All Steps = Locked
    - ProjectStatus = Active
    - SubscriptionState = PaidActive
- Export tied to specific revision.
- Export deterministic.
- No AI call during export.


---

## 3. Workflow 2 — Restart-from-Step (PaidActive)

### Scenario

User has completed up to Step 6 (Locked).
User selects: Restart from Step 3.

### System Behavior

Backend validates:

- SubscriptionState = PaidActive
- Step 3 ≤ highest locked step
- Restart quota available

Atomic transaction:

- Steps 3–N → Draft
- locked_at cleared
- invalidated_at set
- RevisionStatus → Draft
- Export disabled

Steps 1–2 remain Locked.

### Failure Conditions

- Free or PaidExpired user attempts restart → rejected
- Restart on step not yet locked → rejected
- Restart quota exceeded → rejected

No partial state mutation allowed.


---

## 4. Workflow 3 — Create New Revision

### Scenario

User has Completed Revision v1.
User creates new revision.

### System Behavior

Backend validates:

- SubscriptionState = PaidActive
- ProjectStatus = Active

System:

- Clones latest completed revision
- Creates new revision as Draft
- Copies all steps
- Sets all steps to Draft
- Marks new revision as active
- Previous revision remains immutable

### Guarantees

- Only one active revision per project
- Historical revisions preserved
- Revision creation does NOT consume project quota
- Revision creation does NOT consume AI quota


---

## 5. Workflow 4 — Free Tier User

### Scenario

Free user creates project.

### Behavior

- Access limited to early steps
- Cannot complete full revision
- Cannot export
- Cannot create new revision
- Cannot restart-from-step
- AI usage limited

### Failure Paths

If Free user attempts:

- Locking later steps → rejected
- Export → rejected
- Restart → rejected
- New revision → rejected

System must reject without mutating state.


---

## 6. Workflow 5 — Subscription Expiration (Downgrade)

### Scenario

PaidActive user subscription expires.

### System Behavior

User transitions to PaidExpired.

Rules:

- No project deleted
- No revision deleted
- No step invalidated
- No export removed
- No state mutation

Permissions change:

- Export disabled
- Restart disabled
- New revision creation disabled
- AI quota reduced
- New project creation limited by Free cap

### Critical Guarantee

Subscription changes never mutate blueprint state.


---

## 7. Workflow 6 — Subscription Reactivation

### Scenario

PaidExpired user re-subscribes.

### Behavior

- SubscriptionState → PaidActive
- Full capabilities restored
- Existing projects regain functionality
- No state reconstruction required
- Quotas reset for new billing period


---

## 8. Workflow 7 — AI Suggestion Failure

### Scenario

User requests AI suggestion.
AI provider:

- Times out
- Returns error
- Returns invalid structure

### System Behavior

- Step remains Draft
- No lock mutation
- No revision mutation
- No partial writes
- User may retry

AI failure must not block manual editing.


---

## 9. Workflow 8 — Validation Failure on Step Lock

### Scenario

User attempts to lock step with invalid structure.

### System Behavior

- Lock rejected
- Step remains Draft
- RevisionStatus unchanged
- User receives validation feedback

No state corruption allowed.


---

## 10. Workflow 9 — Export Attempt in Invalid State

### Invalid Scenarios

- Any Step = Draft
- Any Step = Invalidated
- RevisionStatus ≠ Completed
- ProjectStatus = SoftDeleted
- SubscriptionState ≠ PaidActive

### Behavior

- Export rejected
- No state mutation
- Deterministic response


---

## 11. Workflow 10 — Soft Delete Project

### Scenario

User soft deletes project.

### Behavior

- ProjectStatus → SoftDeleted
- Hidden from default dashboard
- Data retained
- Revisions preserved
- Export disabled

Quota Behavior:

- Free: still counts toward total project cap
- Paid: frees active project slot

SoftDeleted projects cannot accept new revisions.


---

## 12. System-Level Failure Invariants

The following must always hold:

- If any Step ≠ Locked → RevisionStatus = Draft
- Completed revision contains no Draft or Invalidated steps
- Restart invalidates downstream steps atomically
- Subscription changes never mutate blueprint state
- Export requires Completed revision and PaidActive subscription
- No partial restart cascade allowed
- No partial revision completion allowed
- Quota checks occur before mutation
- AI failure never mutates locked state


---

## 13. Behavioral Integrity Principles

DevPilot prioritizes:

- Determinism over convenience
- Atomic transitions over partial success
- State integrity over feature speed
- Permission gating without data mutation

All workflows must preserve structural integrity.

