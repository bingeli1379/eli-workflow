---
name: eli-apply
description: >
  Implement tasks from a spec change using Agent Team dispatch.
  Use when the user wants to start or continue implementing a change.
  Reads spec artifacts and dispatches tasks to specialized agents.
user-invocable: true
---

Implement tasks from a spec change. Reads all spec artifacts, prepares context, then launches the orchestrator agent to coordinate the team.

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

2. **Confirm current branch**

   Use the current branch as-is. Do NOT create or switch branches — the user manages branches themselves.
   - Announce: "Branch: **<current-branch>**"

3. **Read all context files**

   Read these files from `eli-spec/changes/<name>/`:
   - `proposal.md` — scope and capabilities
   - `design.md` — technical decisions and approach
   - `tasks.md` — implementation checklist
   - `specs/*/spec.md` — all capability specs (acceptance criteria)

   Also read:
   - `eli-spec/config.yaml` — project context and `lint_commands` (if exists)

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

5. **Launch the orchestrator agent**

   Use the **Agent** tool to spawn the orchestrator as a **named, foreground agent**:
   - `name`: `"orchestrator"`
   - `subagent_type`: `"eli-workflow:orchestrator"`

   Pass ALL the context you read in Step 3 as the agent prompt:

   ```
   You are the orchestrator for change "<change-name>".

   ## Spec Artifacts

   ### Proposal
   [full proposal.md content]

   ### Design
   [full design.md content]

   ### Tasks
   [full tasks.md content]

   ### Specs
   [all spec files content, labeled by capability]

   ### Project Config
   [config.yaml content, including lint_commands]

   ## Agent Prompt Template

   When dispatching agents, compose their prompt with:

   """
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

   ## Lint Commands (from config.yaml)
   [lint_commands list, or "none configured" if empty]

   ## Instructions
   - Implement each task in order
   - Follow the spec scenarios as acceptance criteria
   - Follow the design decisions — do NOT deviate
   - **After completing each task**, you MUST:
     1. Update `tasks.md`: change that task's `- [ ]` to `- [x]`
     2. Run all lint commands listed above (if any) to fix formatting — stage any changes they produce
     3. Commit ALL changes together (code + checkbox + lint fixes) using Conventional Commits: `<type>(scope): <task-number> <description>` (e.g., `feat(domain): 1.1 add UserSearch entity`)
   - Do NOT batch multiple tasks into one commit
   - After the commit, report back: "DONE: <task-number> <task-description>"
   - Only add code comments for business logic that is not obvious from the code — if good naming makes it clear, skip the comment
   - Do NOT ask questions — specs should be complete. If something is genuinely ambiguous, skip it and flag it
   """

   ## Your Instructions
   Begin implementing now. Follow your Spec-Driven Mode dispatch process.
   ```

6. **After orchestrator completes, verify and report**

   When the orchestrator agent returns:
   - Read `tasks.md` and verify all completed tasks are checked `- [x]`
   - If any completed task was missed, update it now as a safety net
   - Show final status (see templates below)

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

7. **User can interact with orchestrator at any time**

   The orchestrator agent is named `"orchestrator"`. While it is running, the user can send messages to it via `SendMessage(to: "orchestrator")` to:
   - Ask for progress updates
   - Reprioritize tasks
   - Ask it to dispatch a specific agent
   - Pause or adjust the plan

   Tell the user: "Orchestrator is running as **orchestrator**. You can talk to it anytime."

---

## Guardrails

- **Never ask questions during implementation** — specs are the single source of truth
- Always read ALL context files before launching the orchestrator
- Only dispatch agents for PENDING tasks (skip completed `- [x]` tasks)
- Agents MUST update the task checkbox in `tasks.md`, run lint commands, and include everything in the same commit — one atomic commit per task
- If `lint_commands` are configured in `config.yaml`, agents MUST run them before every commit — no exceptions
- If a task genuinely cannot be implemented (missing dependency, unclear spec), skip it and flag it in the report — do NOT block the entire pipeline
- Keep code changes minimal and scoped to each task
- **One commit per task** — each task gets its own commit using Conventional Commits: `<type>(scope): <task-number> <description>`
- Work on the current branch — do NOT create or switch branches
- Code review and QA are mandatory steps — do NOT skip them
- If review or QA fails, orchestrator auto-dispatches fixes immediately (max 2 retry rounds). Only pause and report to user if still failing after retries.
- Pass only RELEVANT specs to each agent (not all specs) to keep context focused
