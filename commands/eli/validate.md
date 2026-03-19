---
name: "Eli: Validate"
description: Validate spec artifacts against structural and content rules
category: Workflow
tags: [workflow, validate, check]
---

Validate spec artifacts for a change. Checks structural completeness, content quality, and referential integrity. All violations are errors — any failure blocks implementation.

**Usage**: `/eli:validate <change-name>`

On PASS, suggests `/eli:apply <name>`. On FAIL, lists specific fixes needed.

→ Delegates to skill `eli-validate` for full implementation.
