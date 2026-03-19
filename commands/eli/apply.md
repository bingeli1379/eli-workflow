---
name: "Eli: Apply"
description: Implement tasks from a spec change using agent team dispatch
category: Workflow
tags: [workflow, implement, apply]
---

Implement tasks from a spec change. Reads all spec artifacts and dispatches tasks to specialized agents. Does NOT ask questions during implementation — specs are the single source of truth.

**Usage**: `/eli:apply [change-name]`

Pipeline: implementation agents (parallel) → review-engineer + security-engineer → qa-engineer (E2E) → technical-writer

On completion, suggests `/eli:archive <name>`.

→ Delegates to skill `eli-apply-change` for full implementation.
