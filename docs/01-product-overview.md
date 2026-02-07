# Product Overview – DevPilot

This document defines the core purpose, scope, and assumptions of DevPilot.  
It intentionally avoids implementation, architecture, and infrastructure details.

---

## 1️⃣ Product One-Pager

### What is DevPilot?

**DevPilot** is an AI-assisted, step-by-step product blueprinting platform for developers and technical founders.

It guides users through a **structured, sequential workflow** to define product scope, features, permissions, data models, and system behavior **before writing production code**.

The core philosophy is **clarity before execution**.

---

### Who it’s for

DevPilot is designed for:
- Solo developers
- Early-stage startup founders
- Senior engineers designing new systems
- Small teams needing alignment before development

DevPilot is **not** intended for non-technical users or “one-click app builders”.

---

### What problem does it solve?

Many software projects fail or slow down due to:
- unclear requirements
- missing edge cases
- repeated re-design
- lack of a shared mental model

DevPilot addresses this by:
- enforcing a guided, step-by-step decision process
- making assumptions explicit
- preserving state across steps
- preventing premature or chaotic design decisions

The result is a **clear, structured product blueprint** that can be confidently implemented.

---

### What it intentionally does NOT solve

DevPilot does **not**:
- generate final production-ready code
- replace system design thinking
- make architectural decisions automatically
- guarantee the “best” or “optimal” design

AI assistance is **supportive, not authoritative**.

---

### Primary Assumptions

These assumptions define the product boundaries:

- **One user can have multiple projects**
- **Only one active project at a time**
  - Ensures focus, consistency, and clean state management
- **AI is assistive, not authoritative**
  - Users can accept, modify, or reject AI output
- **Steps are sequential and stateful**
  - Earlier decisions influence later steps
  - Steps may be locked after confirmation

Changing these assumptions would fundamentally change the product.

---

## 2️⃣ System Actors

Only real actors are listed below.  
No flowcharts or internal abstractions are included.

---

### 👤 End User

**Can do**
- Create and manage projects
- Progress through steps sequentially
- Provide inputs and make decisions
- Regenerate AI suggestions within defined limits
- Lock steps intentionally

**Cannot do**
- Skip locked or mandatory steps
- Bypass system constraints
- Treat AI output as final without confirmation

---

### 🛠 Admin

**Can do**
- Manage users and subscriptions
- Control system-wide settings
- Enable or disable features and steps
- Monitor system usage and limits

**Cannot do**
- Modify user project content directly
- Inject decisions into user workflows
- Act on behalf of the AI engine

---

### 🤖 AI Engine (External Dependency)

**Can do**
- Generate suggestions and summaries
- Propose structures and options
- Respond to user-initiated prompts

**Cannot do**
- Persist application state
- Make final decisions
- Act without an explicit trigger

---

### 🔐 Auth Provider (Auth.js / OAuth)

**Can do**
- Authenticate users
- Manage login sessions
- Provide identity context

**Cannot do**
- Control authorization rules
- Access product or project data
- Enforce business logic

---

## ✋ Stop Rule

This document intentionally stops here.

- No database schema  
- No APIs  
- No flows  
- No infrastructure  

Those belong to later design phases.
