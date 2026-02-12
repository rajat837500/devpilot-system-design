# 02 — Users, Roles & Business Rules


## 1. Role Model (MVP)

DevPilot MVP supports exactly three user states.

There is no organization model.
There are no shared roles.
There is no role inheritance.

All ownership is single-user.


---

### 1.1 Unauthenticated Visitor

A non-logged-in user browsing the platform.

Capabilities:
- View marketing content
- View feature descriptions
- View pricing information

Restrictions:
- Cannot create projects
- Cannot access blueprint data
- Cannot invoke AI


---

### 1.2 Free User

An authenticated user without an active paid subscription.

Characteristics:
- Single identity
- Owns their projects
- Subject to strict structural and economic limits
- Intended for workflow demonstration only

Free Tier Constraints:
- Maximum 1 total project (including SoftDeleted)
- Limited access to early blueprint steps
- Cannot complete a full revision
- Cannot export blueprint artifacts
- Cannot create new revisions
- Cannot restart-from-step
- Limited AI quota
- Storage quota enforced

Quota Enforcement Model (Free):
Quota is based on total projects ever created:

COUNT(project WHERE user_id = X)

SoftDeleted projects count toward this limit.
Deletion does not free quota.


---

### 1.3 PaidActive User (Pro Plan)

An authenticated user with an active subscription.

Characteristics:
- Full blueprint lifecycle access
- Revision evolution enabled
- Restart-from-step enabled
- Higher AI quota
- Higher storage limits

Paid Tier Capabilities:
- Multiple active projects (concurrent cap)
- Full access to all blueprint steps
- Can complete revisions
- Can export completed revisions
- Can create new revisions
- Can restart-from-step
- Higher AI quota
- Restart quota per billing period

Active Project Enforcement (Paid):
Quota is based on concurrent active projects:

COUNT(project WHERE user_id = X AND project_status = Active)

SoftDeleted projects do NOT count toward active cap.
Soft delete frees an active slot.


---

### 1.4 PaidExpired User

A user whose paid subscription has expired.

PaidExpired is treated as Free for mutation permissions,
but retains full read access to all historical data.

Behavior:

- All projects remain accessible (read)
- No project or revision is deleted
- No state mutation occurs
- Export disabled
- Restart-from-step disabled
- New revision creation disabled
- AI quota reduced to Free limits
- New project creation blocked if exceeding Free cap

Important Principle:
Subscription state changes must never mutate
Project state
Revision state
Step state


---

## 2. Subscription Model (MVP)

MVP includes two plans:

- Free
- Pro

No trials.
No multi-tier paid plans (future scope).


---

### 2.1 Plan Resolution Rule

At any moment, a user has exactly one active plan.

Plan change behavior:

- Previous billing period closes
- New billing period begins immediately
- Quota resets to new plan limits
- No quota stacking
- No overlapping quota pools
- No carry-over of unused quota

This prevents billing ambiguity and quota fragmentation.


---

## 3. Quota Model

Quota is evaluated per billing period (PaidActive).

Quota Types:

- Active project cap (concurrent)
- AI suggestion quota
- AI regeneration quota
- Restart-from-step quota
- Storage limit

Quota Rules:

- Quota check occurs BEFORE mutation
- Usage increments only AFTER successful operation
- Quota failure results in no state mutation
- Revision creation does NOT consume project quota
- Revision creation does NOT consume AI quota


---

## 4. Project Lifecycle Rules

ProjectStatus:
- Active
- SoftDeleted

Allowed Transitions:
Active → SoftDeleted
SoftDeleted → Active (optional restore)

Rules:

- Free tier: SoftDeleted counts toward total project cap
- Paid tier: SoftDeleted does not count toward active cap
- SoftDeleted projects cannot accept new revisions
- SoftDeleted projects cannot be exported
- Hard delete not supported in MVP


---

## 5. Economic Abuse Modeling

### 5.1 Free Tier Quota Bypass

Risk:
User creates → deletes → recreates project.

Mitigation:
Free quota based on total project count (SoftDeleted included).


---

### 5.2 AI Cost Abuse

Risk:
User repeatedly regenerates AI output.

Mitigation:
- Per-user quota
- Per-period quota
- Regeneration cap
- Server-side enforcement


---

### 5.3 Restart Abuse

Risk:
User repeatedly restarts from early steps.

Mitigation:
Restart quota per billing period.
Server-side validation.


---

### 5.4 Storage Abuse

Risk:
User uses blueprint steps as arbitrary storage.

Mitigation:
- Per-project size limit
- Per-user storage cap
- Structured input validation


---

## 6. Business-Level Invariants

The following must always hold:

- Subscription state changes never mutate blueprint state.
- Free quota is lifetime-based (SoftDeleted included).
- Paid quota is concurrent-based.
- Revision creation does not affect project quota.
- Export requires PaidActive subscription.
- Restart-from-step requires PaidActive subscription.
- AI usage always metered server-side.
- Ownership is single-user only.

