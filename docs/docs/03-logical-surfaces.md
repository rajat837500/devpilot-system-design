# 03 — Logical Surfaces & Responsibility Boundaries


## 1. Purpose

This section defines the logical interaction surfaces of DevPilot.

These are not UI routes.
These are not API endpoints.

They are responsibility domains describing:

- What the system reads
- What the system writes
- What actions are allowed
- What invariants must be enforced

This ensures clean separation of concerns before implementation.


---

## 2. Overview of Logical Surfaces (MVP)

DevPilot contains six primary logical surfaces:

1. Authentication & Subscription Surface
2. Project Dashboard Surface
3. Project Overview Surface
4. Step Detail Surface (Core Engine)
5. Export Surface
6. Account & Usage Surface


---

## 3. Authentication & Subscription Surface

### Purpose
Manage user identity and subscription lifecycle.

### Reads
- User identity
- Subscription state
- Current plan
- Quota state

### Writes
- Account creation
- Session creation
- Subscription state transitions

### Allowed Actions
- Register
- Login / Logout
- Upgrade subscription
- Cancel subscription

### Enforcement Rules
- Subscription validation must occur server-side
- Downgrade does not mutate blueprint state
- Quota enforcement applies to future actions only
- No blueprint state logic lives here


---

## 4. Project Dashboard Surface

### Purpose
Manage project lifecycle.

### Reads
- All user-owned projects
- Project status (Active / SoftDeleted)
- Active project count
- Subscription limits

### Writes
- Create project
- Soft delete project
- Restore project (optional)

### Allowed Actions
- Create project (quota-gated)
- Open project
- Soft delete project

### Enforcement Rules
- Free tier: max 1 total project (SoftDeleted included)
- Paid tier: active project cap enforced
- SoftDeleted projects cannot accept mutations
- Ownership validated server-side
- No blueprint editing occurs here


---

## 5. Project Overview Surface

### Purpose
Coordinate revision lifecycle within a project.

### Reads
- Project metadata
- Revision list
- Active revision
- Step completion states
- Export eligibility

### Writes
- Create new revision (PaidActive only)
- Set active revision

### Allowed Actions
- View revision history
- Start new revision
- Navigate to specific step
- Trigger export (if eligible)

### Enforcement Rules
- Only one active revision per project
- Completed revisions are immutable
- Revision creation requires PaidActive subscription
- SoftDeleted projects cannot create revisions


---

## 6. Step Detail Surface (Core Engine)

This is the structural heart of DevPilot.

### Purpose
Enforce step-level blueprint editing, locking, and restart logic.

### Reads
- Step data
- Step status (Draft / Locked / Invalidated)
- Upstream locked steps (read-only)
- Revision status
- Subscription state
- AI quota state

### Writes
- Step draft updates
- Step lock transition
- Restart-from-step cascade

### Allowed Actions
- Save draft
- Request AI suggestion (quota-bound)
- Lock step (if validation passes)
- Restart-from-step (PaidActive only)

### Enforcement Rules
- Cannot access step N if step N-1 is not Locked
- Locked steps cannot be directly edited
- Restart-from-step K:
    - Steps K–N → Draft
    - locked_at cleared
    - invalidated_at set
    - Revision → Draft
    - Export disabled
- Restart must execute atomically
- AI output cannot auto-lock step
- Validation must pass before lock
- All transitions enforced server-side

This surface owns deterministic progression.


---

## 7. Export Surface

### Purpose
Generate deterministic blueprint artifacts per revision.

### Reads
- Locked step data
- Revision metadata
- Subscription state
- Project status

### Writes
- Export record

### Allowed Actions
- Generate export
- Download export

### Enforcement Rules
Export allowed only if:

- ProjectStatus = Active
- RevisionStatus = Completed
- All Steps = Locked
- SubscriptionState = PaidActive

Export:
- Is tied to specific revision
- Does not mutate revision state
- Must be reproducible from locked data


---

## 8. Account & Usage Surface

### Purpose
Provide visibility into subscription and quota usage.

### Reads
- Subscription state
- Plan limits
- AI usage
- Restart usage
- Storage usage
- Active project count

### Writes
- Account metadata updates

### Allowed Actions
- View usage metrics
- Upgrade subscription
- Cancel subscription

### Enforcement Rules
- Downgrade does not delete projects
- Usage metrics are informational
- Quota enforcement occurs before state mutation
- All limits resolved server-side


---

## 9. Structural Guarantees

From these surfaces, the following structural properties are enforced:

- Backend is sole authority for state transitions
- No client-side lock enforcement
- Deterministic revision completion
- Subscription gating at mutation boundaries
- Atomic restart cascade
- Immutable completed revisions
- Clear separation between economic logic and blueprint logic


---

## 10. Explicit Non-Surfaces (MVP Exclusions)

The following surfaces do NOT exist in MVP:

- Admin panel
- Organization management
- Collaboration interface
- Branch comparison
- Diff viewer
- Template marketplace
- Activity feed
- Analytics dashboard

These are deliberate exclusions to preserve structural clarity.

