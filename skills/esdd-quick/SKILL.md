---
name: esdd-quick
description: >
  Quick task execution with orchestrator analysis but no spec artifacts.
  Use when the user has small-to-medium tasks and wants agent team dispatch
  without the full propose → validate → apply ceremony.
user-invocable: true
---

Lightweight alternative to the full `/esdd-propose` → `/esdd-apply` pipeline. The orchestrator **analyzes the task inline** (similar to propose) and **dispatches agents directly** — no spec files are written to disk.

Best for: bug fixes, small features, refactors, chores — tasks where full spec ceremony is overkill but you still want the agent team's specialization and quality gates.

---

**Input**: A task description (e.g., `/esdd-quick fix the login redirect loop` or `/esdd-quick add dark mode toggle to settings page`).

**Steps**

1. **Get the task description**

   If no description is provided, use **AskUserQuestion** (open-ended) to ask:
   > "What do you want to do? Describe the task."

   Do NOT proceed without a clear task description.

2. **Read project context**

   - Read `feature-spec/config.yaml` for project context (tech stack, conventions, lint commands)
   - If `config.yaml` doesn't exist, proceed without it — use defaults from CLAUDE.md

3. **Confirm current branch**

   Use the current branch as-is. Do NOT create or switch branches.
   Announce: "Branch: **<current-branch>**"

4. **Pre-lint and commit (clean slate)**

   If `lint_commands` are configured in `feature-spec/config.yaml`:
   1. Run all lint commands to fix pre-existing formatting issues
   2. If lint produced changes: stage and commit with `chore: pre-lint cleanup before esdd-quick`
   3. If no changes, skip silently

5. **Analyze the task (inline propose)**

   This is the core difference from `/esdd-apply`. Instead of reading spec files, you **perform the analysis yourself** — similar to what `/esdd-propose` does, but entirely in-memory without writing any files.

   **a. Scope analysis:**
   - What is the task trying to achieve?
   - Which layers are affected? (Frontend, Backend, Database, DevOps, etc.)
   - What are the key design decisions? (API shape, data model changes, UI approach)
   - What are the acceptance criteria? (When X happens, then Y should be the result)

   **b. Task breakdown:**
   - Break the task into discrete subtasks
   - Group by feature/phase (same as tasks.md format)
   - Tag each subtask with an agent type: `(Backend)`, `(Frontend)`, `(E2E)`, `(Electron)`, `(Database)`, `(DevOps)`, `(Performance)`, `(Security)`, `(Documentation)`
   - Follow TDD structure for Backend/Frontend tasks when appropriate (write test → implement)
   - Number tasks: `1.1`, `1.2`, etc.

   **c. Complexity judgment:**
   - **Simple** (single agent): one layer only → implementation agent → review
   - **Medium** (2-3 agents): cross-cutting → parallel implementation → review
   - **Complex** (full pipeline): new module/feature → full 4-phase pipeline

   **d. Identify ambiguities and unknowns:**
   - Are there vague requirements? ("improve" → improve what exactly?)
   - Missing edge case handling? (empty input, concurrent access, error states)
   - Unclear integration points with existing code?
   - Design decisions that could go multiple ways?

   **e. If ambiguities exist — ask the user ONCE:**

   Use **AskUserQuestion** with a structured summary. Ask ALL questions in ONE message:

   ```
   ## Quick Task: <summary>

   **Scope:** <affected layers>
   **Complexity:** <Simple/Medium/Complex>

   ### My Understanding
   - <what I plan to do>

   ### Questions (need your input)
   1. <specific question about unclear behavior>
   2. <specific question about edge case or design choice>

   ### Planned Tasks (pending your answers)
   - 1.1 (Backend) <task description>
   - 1.2 (Frontend) <task description>
   ...
   ```

   After the user responds, incorporate their answers into the plan.

   **f. If NO ambiguities — present the plan and dispatch immediately:**

   If the task description is clear and unambiguous, skip the question step. Show the plan and dispatch right away:

   ```
   ## Quick Task: <summary>

   **Scope:** <affected layers>
   **Complexity:** <Simple/Medium/Complex>

   ### Design Decisions
   - <key decision 1>
   - <key decision 2>

   ### Tasks
   ## 1. <Group Name>
   - [ ] 1.1 (Backend) <task description>
   - [ ] 1.2 (Frontend) <task description>
   ...

   ### Acceptance Criteria
   - WHEN <condition> THEN <expected result>
   - WHEN <condition> THEN <expected result>

   ### Agents to Dispatch
   - <agent-1>: <task count> tasks
   - <agent-2>: <task count> tasks

   Dispatching now.
   ```

   **Decision rule**: Only ask when there are genuine unknowns that would lead to wrong implementation. If you can make a reasonable decision, make it and note it — don't ask just to be safe.

6. **Become the orchestrator and dispatch**

   Read `agents/orchestrator.md` to load the orchestrator role. You are now the orchestrator.

   **Agent Prompt Template** — compose each worker agent's prompt with:

   ```
   You are working on a quick task: "<task summary>"

   ## Your Role
   [agent role definition from agents/<agent>.md]

   ## Project Context
   [from feature-spec/config.yaml, or CLAUDE.md defaults]

   ## Design Decisions
   [from your inline analysis in step 5]

   ## Acceptance Criteria
   [from your inline analysis in step 5]

   ## Your Tasks
   [specific tasks for this agent from step 5]

   ## Lint Commands (from config.yaml)
   [lint_commands list, or "none configured"]

   ## Instructions
   - Implement each task in order
   - Follow the design decisions — do NOT deviate
   - **CRITICAL — Committing is EXPLICITLY REQUIRED by the user as part of this workflow. You are authorized and expected to commit after every task. This is NOT optional.** After completing each task, you MUST:
     1. Stage all changed files with `git add` (specify files by name)
     2. Run all lint commands listed above (if any) — stage any changes they produce
     3. Commit following the `conventional-commits` skill. Format: `<type>[optional scope]: <task-number> <description>` (e.g., `fix: 1.1 resolve login redirect loop`)
   - Do NOT batch multiple tasks into one commit — one commit per task
   - After the commit, report back: "DONE: <task-number> <task-description>"
   - Only add code comments for business logic that is not obvious from the code
   - Do NOT ask questions — if something is ambiguous, make a reasonable decision and flag it
   - **Language**: All output and reports MUST be in Traditional Chinese. Code and code comments MUST be in English.
   ```

   **Dispatch rules (same as esdd-apply):**
   - Use the **Agent** tool with `run_in_background: true` for ALL worker agents
   - Give each agent a descriptive `name`
   - Dispatch agents that can run in parallel **simultaneously**
   - You will be **automatically notified** when each background agent completes — do NOT poll

   **Phase execution based on complexity:**

   **Simple tasks:**
   - Phase 1: Single implementation agent
   - Phase 2: review-engineer + security-engineer (parallel)
   - Done.

   **Medium tasks:**
   - Phase 1: Implementation agents in parallel
   - Phase 2: review-engineer + security-engineer (parallel)
   - Done.

   **Complex tasks (full pipeline):**
   - Phase 1: All implementation agents in parallel (including qa-engineer for E2E test writing)
   - Phase 2: review-engineer + security-engineer (parallel)
   - Phase 3: qa-engineer runs E2E tests
   - Phase 4: technical-writer (if documentation changes needed)

   If review or QA fails: dispatch responsible agent to fix → re-verify (max 2 retries). Only pause and report to user if still failing.

7. **Interactive control**

   While agents run in background, respond to user messages:
   - **"status" / "進度"** — show current phase and progress
   - **"pause" / "暫停"** — stop dispatching new agents
   - **"skip <task>"** — skip a specific task
   - **Any other message** — interpret as orchestrator instruction

   When a background agent completes, announce briefly:
   ```
   [agent-name] completed: <summary>
   Progress: N/M tasks
   ```

8. **Final report**

   After all phases complete:

   ```
   ## Quick Task Complete

   **Task:** <summary>
   **Complexity:** <Simple/Medium/Complex>
   **Progress:** M/M tasks complete

   ### Completed
   - [x] 1.1 <task description>
   - [x] 1.2 <task description>
   ...

   ### Code Review
   [APPROVED / APPROVED WITH COMMENTS]

   ### Security Review
   [SECURE / ISSUES FOUND]

   ### E2E (if applicable)
   [PASSED / FAILED / SKIPPED]

   ### Notes
   [issues encountered, decisions made, follow-up suggestions]
   ```

---

## Guardrails

- **You ARE the orchestrator** — do NOT spawn a separate orchestrator agent
- **All worker agents run in background** (`run_in_background: true`)
- **No spec files are written** — analysis stays in-memory and is passed to agents via prompts
- **Execute first, report after** — show the plan and dispatch immediately, do NOT wait for user confirmation
- **Code review + security review are MANDATORY** for all complexity levels — never skip them
- If review/QA fails → auto-dispatch fix (max 2 retries) → only then pause
- One commit per task — atomic commits using Conventional Commits
- Work on the current branch — do NOT create or switch branches
- Keep the plan concise — this is quick mode, not a full spec
- **Language**: All output in Traditional Chinese. Code and comments in English.

## When to Suggest Full Spec Instead

If during analysis (step 5) you determine the task is:
- Touching 3+ independent capabilities
- Would produce 15+ tasks
- Requires significant architectural decisions
- Needs cross-team coordination

Then suggest: "This task looks complex enough for the full spec flow. Want me to run `/esdd-propose` instead?"

But still proceed if the user insists on quick mode.
