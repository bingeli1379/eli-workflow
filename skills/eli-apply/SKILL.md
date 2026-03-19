---
name: eli-apply
description: >
  Implement tasks from a spec change using Agent Team dispatch.
  Use when the user wants to start or continue implementing a change.
  Reads spec artifacts and dispatches tasks to specialized agents.
user-invocable: true
---

Implement tasks from a spec change. Reads all spec artifacts and dispatches tasks to the appropriate specialized agents through the orchestrator.

**IMPORTANT**: This skill does NOT ask questions during implementation. All requirements should be fully specified in the spec artifacts. If specs are incomplete, suggest running `/eli-validate` first.

---

**Input**: Optionally specify a change name (e.g., `/eli-apply add-user-search`). If omitted, auto-detect.

**Steps**

1. **Select the change**

   If a name is provided, use it. Otherwise:
   - List directories under `eli-spec/changes/` (excluding `archive/`)
   - Auto-select if only one active change exists
   - If multiple, use **AskUserQuestion** to let the user choose
   - If none exist, report error: "No active changes found. Run `/eli-propose` first."

   Always announce: "Implementing change: **<name>**"

2. **Create or switch to feature branch**

   Branch name: `<change-name>` (same as the spec change directory name, e.g., `add-user-search`)

   - First, detect the main branch by checking which of `main` or `master` exists on remote
   - If branch `<change-name>` already exists: switch to it (this is a resume scenario)
   - If branch does not exist: create it from the detected main branch
     ```bash
     git checkout <main-branch> && git pull && git checkout -b <change-name>
     ```
   - Announce: "Branch: **<change-name>** (from `<main-branch>`)"

3. **Read all context files**

   Read these files from `eli-spec/changes/<name>/`:
   - `proposal.md` — scope and capabilities
   - `design.md` — technical decisions and approach
   - `tasks.md` — implementation checklist
   - `specs/*/spec.md` — all capability specs (acceptance criteria)

   Also read:
   - `eli-spec/config.yaml` — project context (if exists)

   **If any required file is missing** (proposal, design, tasks, or specs):
   - Show which files are missing
   - Suggest: "Run `/eli-validate <name>` to check completeness, or `/eli-propose` to generate missing artifacts."
   - Stop.

4. **Parse tasks and show progress**

   Parse `tasks.md`:
   - Identify task groups (## headings) and their agent mapping
   - Count total tasks, completed (`- [x]`), and pending (`- [ ]`)
   - If all tasks are complete: congratulate, suggest `/eli-archive <name>`

   Display:
   ```
   ## Implementing: <change-name>
   **Progress:** N/M tasks complete
   **Remaining groups:**
   - Backend - Search API (3 tasks)
   - Frontend - Search Page (5 tasks)
   ```

5. **Dispatch tasks to agents via orchestrator**

   The orchestrator reads all spec artifacts and dispatches agents based on task groups.

   **Agent mapping from task group names:**

   | Group keyword | Agent | Role file |
   |--------------|-------|-----------|
   | `Backend`, `API`, `Domain`, `Infrastructure` | dotnet-engineer | `agents/dotnet-engineer.md` |
   | `Frontend`, `UI`, `Component`, `Page` | vue-engineer | `agents/vue-engineer.md` |
   | `Electron`, `Main Process`, `IPC`, `Preload` | electron-engineer | `agents/electron-engineer.md` |
   | `Database`, `Migration`, `Schema`, `Index` | database-engineer | `agents/database-engineer.md` |
   | `DevOps`, `Docker`, `CI`, `CD`, `K8s`, `Pipeline` | devops-engineer | `agents/devops-engineer.md` |
   | `Performance`, `Optimization`, `Caching`, `Bundle` | performance-engineer | `agents/performance-engineer.md` |
   | `Security` | security-engineer | `agents/security-engineer.md` |
   | `Documentation`, `Docs` | technical-writer | `agents/technical-writer.md` |
   | `E2E` | qa-engineer | `agents/qa-engineer.md` |
   | `Integration` | Orchestrator coordinates multiple agents |

   **Dispatch strategy:**

   a. **Phase 1 — Parallel development**: Dispatch these agents **in parallel**:
      - QA agent: writes E2E tests (Playwright) from spec WHEN/THEN scenarios
      - Backend agent(s): implements features with TDD (unit tests first)
      - Frontend agent(s): implements features with TDD (unit tests first)
      - All three can work simultaneously because design.md provides API contract and shared types

   b. **For each group, dispatch the corresponding agent** with this context:
      - The agent's role definition (from `agents/<agent>.md`)
      - Relevant spec files (only the specs related to this group's capability)
      - The design decisions that affect this group
      - The specific tasks assigned to this agent from `tasks.md`
      - Project context from `config.yaml`

   c. **Agent prompt template:**
      ```
      You are working on change "<change-name>".

      ## Your Role
      [agent role definition from agents/<agent>.md]

      ## Project Context
      [from eli-spec/config.yaml]

      ## Design Decisions
      [relevant sections from design.md]

      ## Your Specs (Acceptance Criteria)
      [relevant spec files]

      ## Your Tasks
      [specific tasks from tasks.md for this group]

      ## Instructions
      - Implement each task in order
      - Follow the spec scenarios as acceptance criteria
      - Follow the design decisions — do NOT deviate
      - After completing each task, report what was done
      - Do NOT ask questions — specs should be complete. If something is genuinely ambiguous, skip it and flag it
      - Commit after EACH task (not grouped) using Conventional Commits format: `<type>(scope): <task-number> <description>` (e.g., `feat(domain): 1.1 add UserSearch entity`, `test(api): 1.2 add search endpoint unit tests`, `fix(auth): 2.1 correct token validation`)
      - Do NOT batch multiple tasks into one commit
      ```

   d. **Phase 1 — Parallel dispatch using Agent tool**:
      - QA (E2E test writing) + Backend groups + Frontend groups → all in parallel
      - After all complete → Integration group (if any)

   e. **Phase 2 — After implementation agents complete, dispatch reviews in parallel**:
      - review-engineer (`agents/review-engineer.md`) — architecture compliance, code quality
      - security-engineer (`agents/security-engineer.md`) — vulnerabilities, auth, injection
      - If either returns REQUEST CHANGES / ISSUES FOUND: show issues, pause for user decision

   f. **Phase 3 — After reviews pass, dispatch qa-engineer to run E2E tests**:
      - QA agent runs the Playwright E2E tests written in Phase 1
      - Tests verify ALL spec WHEN/THEN scenarios pass end-to-end
      - If QA returns FAILED:
        1. Parse which scenarios failed and which agent is responsible
        2. Dispatch the responsible agent (frontend/backend) with failure details
        3. After fix, re-run QA E2E tests (max 2 retry rounds)
        4. If still failing, pause and report to user

   g. **Phase 4 — After QA passes, dispatch technical-writer**:
      - technical-writer generates/updates API docs, changelog, README as needed

6. **Update task checkboxes**

   After each task group completes successfully:
   - Update `tasks.md`: change `- [ ]` to `- [x]` for completed tasks
   - Show progress update: "✓ Group N complete (X/M total)"

7. **On completion or pause, show status**

   **On completion:**
   ```
   ## Implementation Complete

   **Change:** <change-name>
   **Progress:** M/M tasks complete ✓

   ### Completed This Session
   - [x] 1.1 Task description
   - [x] 1.2 Task description
   ...

   ### Code Review
   [APPROVED / APPROVED WITH COMMENTS]

   ### Security Review
   [SECURE / ISSUES FOUND]

   ### E2E Acceptance
   [PASSED — X/Y spec scenarios verified via Playwright]

   All tasks complete! Run `/eli-archive <name>` to archive this change.
   ```

   **On pause (issue encountered):**
   ```
   ## Implementation Paused

   **Change:** <change-name>
   **Progress:** N/M tasks complete

   ### Issue Encountered
   <description of the issue>

   ### Code Review Feedback (if applicable)
   [Must Fix items from reviewer]

   ### Security Review Feedback (if applicable)
   [Security issues found]

   ### E2E Failures (if applicable)
   [Failed spec scenarios with responsible agent identified]
   [Retry attempts: N/2]

   **Options:**
   1. Fix issues and re-run `/eli-apply <name>`
   2. Update specs and re-run
   3. Other approach

   What would you like to do?
   ```

---

## Guardrails

- **Never ask questions during implementation** — specs are the single source of truth
- Always read ALL context files before dispatching any agent
- Only dispatch agents for PENDING tasks (skip completed `- [x]` tasks)
- Update task checkboxes in `tasks.md` immediately after each group completes
- If a task genuinely cannot be implemented (missing dependency, unclear spec), skip it and flag it in the report — do NOT block the entire pipeline
- Keep code changes minimal and scoped to each task
- **One commit per task** — each task gets its own commit using Conventional Commits: `<type>(scope): <task-number> <description>`
- All work happens on the `<change-name>` branch — never commit directly to main/master
- Code review and QA are mandatory steps — do NOT skip them
- If review or QA fails, pause and report — do NOT auto-fix without user consent
- Pass only RELEVANT specs to each agent (not all specs) to keep context focused
