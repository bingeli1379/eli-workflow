---
name: "Eli: Propose"
description: Generate spec artifacts (proposal, design, specs, tasks) for a new change
category: Workflow
tags: [workflow, spec, propose]
---

Generate a complete set of spec artifacts for a new change — proposal, design, specs, and tasks — all in one step.

**Usage**: `/eli:propose <description or kebab-case name>`

Artifacts created:
- `proposal.md` — what & why
- `design.md` — how (technical decisions)
- `specs/<capability>/spec.md` — acceptance criteria (WHEN/THEN)
- `tasks.md` — implementation checklist grouped by agent type

When done, run `/eli:validate <name>` to check, then `/eli:apply <name>` to implement.

→ Delegates to skill `eli-propose` for full implementation.
