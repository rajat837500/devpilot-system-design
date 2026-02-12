# 08 — Data Model & Ownership


## 1. Purpose

This section defines:

- Core entities
- Required fields
- Referential integrity rules
- Ownership hierarchy
- Quota modeling structure
- Determinism safeguards

The data model must support:

- Atomic state transitions
- Downgrade safety
- Deterministic exports
- Quota enforcement
- Clear ownership boundaries


---

## 2. Core Entity Hierarchy

User
→ Subscription
→ Project
→ Revision
→ Step

Supporting entities:

- PlanDefinition
- ExportRecord
- UsageRecord

The hierarchy must remain strictly tree-shaped.
No cross-user relationships allowed.


---

## 3. PlanDefinition Entity

Defines plan capabilities.

### Required Fields

- plan_id (PK)
- plan_name
- billing_interval (Monthly | Yearly | null for Free)
- max_active_projects (nullable)
- max_total_projects (nullable)
- ai_suggestion_quota_per_period
- ai_regeneration_quota_per_period
- restart_quota_per_period
- storage_limit
- revision_limit_per_project (nullable)
- is_active (boolean)
- created_at
- updated_at

### Invariants

- Free plan must exist
- Plan changes must not mutate:
  - Projects
  - Revisions
  - Steps
- Plan limits resolved dynamically at action time
- No quota stacking across plans


---

## 4. User Entity

### Required Fields

- user_id (PK)
- email (unique)
- account_status
- created_at
- updated_at

### Invariants

- Each Project must reference exactly one user_id
- No shared ownership model in MVP


---

## 5. Subscription Entity

Represents user’s current billing state.

### Required Fields

- subscription_id (PK)
- user_id (unique FK)
- plan_id (FK → PlanDefinition)
- subscription_state (Free | PaidActive | PaidExpired)
- current_period_start (nullable)
- current_period_end (nullable)
- cancel_at_period_end (boolean)
- created_at
- updated_at

### Invariants

- Exactly one subscription per user
- subscription_state governs permission checks
- Plan change resets billing period
- Subscription state transitions must not mutate blueprint data


---

## 6. Project Entity

### Required Fields

- project_id (PK)
- user_id (FK → User)
- project_name
- project_status (Active | SoftDeleted)
- created_at
- deleted_at (nullable)
- updated_at

### Constraints

- user_id indexed
- project_status indexed
- Unique ownership

### Invariants

- SoftDeleted projects remain stored
- Free quota counts SoftDeleted
- Paid quota counts only Active
- SoftDeleted cannot accept new revisions
- No hard delete in MVP


---

## 7. Revision Entity

Each Project may contain multiple revisions.

### Required Fields

- revision_id (PK)
- project_id (FK → Project)
- revision_number (monotonic per project)
- revision_status (Draft | Completed)
- is_active (boolean)
- created_at
- completed_at (nullable)

### Constraints

- Unique (project_id, revision_number)
- Only one revision per project where is_active = true
- revision_status indexed

### Invariants

- Completed revision immutable
- Only one active revision
- Revision creation does NOT affect project quota
- RevisionStatus must align with Step states


---

## 8. Step Entity

Each Revision contains ordered steps.

### Required Fields

- step_id (PK)
- revision_id (FK → Revision)
- step_order (integer)
- step_status (Draft | Locked | Invalidated)
- step_data (structured JSON or equivalent)
- locked_at (nullable)
- invalidated_at (nullable)
- updated_at

### Constraints

- Unique (revision_id, step_order)
- step_order fixed by system definition
- step_status indexed

### Invariants

- Cannot lock Step N if Step N-1 not Locked
- Locked steps immutable
- Restart-from-step clears locked_at and sets invalidated_at
- Step transitions must be atomic


---

## 9. ExportRecord Entity

Tracks deterministic export history.

### Required Fields

- export_id (PK)
- revision_id (FK → Revision)
- export_template_version
- export_hash (deterministic fingerprint)
- artifact_location_reference
- exported_at

### Invariants

- Export tied to specific revision_id
- Export allowed only if revision Completed
- Export does not mutate revision
- export_hash must depend only on:
  - Locked step_data
  - Revision metadata
  - export_template_version
- Export reproducible


---

## 10. UsageRecord Entity

Tracks quota consumption per billing period.

### Required Fields

- usage_id (PK)
- user_id (FK → User)
- usage_type (AI_SUGGESTION | AI_REGENERATION | RESTART | STORAGE)
- period_start
- period_end
- usage_count
- updated_at

### Invariants

- Usage evaluated per billing period
- Usage reset occurs by creating new period record
- Usage increment only after successful operation
- Free plan may ignore billing period model (lifetime cap)


---

## 11. Quota Modeling Rules

### Free Plan

Project limit enforced via:

COUNT(project WHERE user_id = X)

SoftDeleted included.

### Paid Plan

Active project limit enforced via:

COUNT(project WHERE user_id = X AND project_status = Active)

AI and Restart limits enforced via:

UsageRecord filtered by current billing period.


---

## 12. Referential Integrity Rules

Must always hold:

- Project.user_id → User.user_id
- Subscription.user_id → User.user_id
- Subscription.plan_id → PlanDefinition.plan_id
- Revision.project_id → Project.project_id
- Step.revision_id → Revision.revision_id
- ExportRecord.revision_id → Revision.revision_id
- UsageRecord.user_id → User.user_id

No orphan records allowed.

Soft delete does not cascade delete children.


---

## 13. Determinism Safeguard

Export artifact must depend only on:

- Locked step_data
- Revision metadata
- Export template version

Must NOT depend on:

- Draft steps
- Live AI calls
- External system state

This guarantees reproducibility.


---

## 14. Data Mutation Safety Rules

All structural mutations must:

- Pass authentication validation
- Pass ownership validation
- Pass subscription validation
- Pass quota validation
- Pass state invariant validation
- Execute atomically
- Preserve referential integrity

Downgrade must never mutate blueprint data.

