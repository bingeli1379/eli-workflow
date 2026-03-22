---
name: esdd-apply
description: >
  Implement tasks from a spec change using Agent Team dispatch.
  Use when the user wants to start or continue implementing a change.
  Reads spec artifacts and dispatches tasks to specialized agents.
user-invocable: true
---

Implement tasks from a spec change. Reads all spec artifacts, prepares context, then **becomes the orchestrator** — the main Claude assumes the orchestrator role directly so the user can interact naturally via chat.

**IMPORTANT**: Specs are the single source of truth. If specs are incomplete, suggest running `/esdd-validate` first.

---

**Input**: Optionally specify a change name (e.g., `/esdd-apply add-user-search`). If omitted, auto-detect.

**Steps**

1. **Select the change**

   If a name is provided, use it. Otherwise:
   - List directories under `feature-spec/changes/` (excluding `archive/`)
   - Auto-select if only one active change exists
   - If multiple, use **AskUserQuestion** to let the user choose
   - If none exist, report error: "No active changes found. Run `/esdd-propose` first."

   Always announce: "Implementing change: **<name>**"

2. **Confirm current branch**

   Use the current branch as-is. Do NOT create or switch branches — the user manages branches themselves.
   - Announce: "Branch: **<current-branch>**"

3. **Pre-lint and commit (clean slate)**

   If `lint_commands` are configured in `feature-spec/config.yaml`:
   1. Run all lint commands to fix any pre-existing formatting issues
   2. Check `git status` — if there are any changes produced by linting:
      - Stage all changed files
      - Commit with message: `chore: pre-lint cleanup before esdd-apply`
   3. If no changes, skip silently

   This ensures agents start from a clean state and their lint runs won't pick up unrelated formatting changes.

4. **Read all context files**

   Read these files from `feature-spec/changes/<name>/`:
   - `proposal.md` — scope and capabilities
   - `design.md` — technical decisions and approach
   - `tasks.md` — implementation checklist
   - `specs/*/spec.md` — all capability specs (acceptance criteria)

   Also read:
   - `feature-spec/config.yaml` — project context and `lint_commands` (if exists)

   **If any required file is missing** (proposal, design, tasks, or specs):
   - Show which files are missing
   - Suggest: "Run `/esdd-validate <name>` to check completeness, or `/esdd-propose` to generate missing artifacts."
   - Stop.

5. **Parse tasks and show progress**

   Parse `tasks.md`:
   - Identify task groups (## headings) and their agent mapping
   - Count total tasks, completed (`- [x]`), and pending (`- [ ]`)
   - If all tasks are complete: congratulate, suggest `/esdd-archive <name>`

   Display:
   ```
   ## Implementing: <change-name>
   **Progress:** N/M tasks complete
   **Remaining groups:**
   - Backend - Search API (3 tasks)
   - Frontend - Search Page (5 tasks)
   ```

6. **Become the orchestrator**

   Read `agents/orchestrator.md` to load the orchestrator role definition. **You are now the orchestrator.** Do NOT spawn a separate orchestrator agent — you act as the orchestrator directly in the main conversation.

   This means:
   - The user can talk to you naturally at any time
   - You dispatch worker agents in the **background** (`run_in_background: true`)
   - You track progress and report back as agents complete
   - The user can ask for status, reprioritize, or give you new instructions mid-flight

   Announce to the user:
   ```
   Orchestrator ready. Dispatching agents now.
   You can talk to me anytime — ask for progress, reprioritize tasks, or adjust the plan.
   ```

7. **Dispatch worker agents (following orchestrator.md rules)**

   Follow the dispatch rules from `agents/orchestrator.md` (Spec-Driven Mode), but with these adaptations:

   **Agent Prompt Template** — compose each worker agent's prompt with:

   ```
   You are working on change "<change-name>".

   ## Your Role
   [agent role definition from agents/<agent>.md]

   ## Project Context
   [from feature-spec/config.yaml]

   ## Design Decisions
   [relevant sections from design.md]

   ## Your Specs (Acceptance Criteria)
   [relevant spec files — only the ones relevant to this agent's tasks]

   ## Your Tasks
   [specific tasks from tasks.md for this agent]

   ## Lint Commands (from config.yaml)
   [lint_commands list, or "none configured" if empty]

   ## Instructions
   - Implement each task in order
   - Follow the spec scenarios as acceptance criteria
   - Follow the design decisions — do NOT deviate
   - **CRITICAL — Committing is EXPLICITLY REQUIRED by the user as part of this workflow. You are authorized and expected to commit after every task. This is NOT optional.** After completing each task, you MUST:
     1. Stage all changed files with `git add` (specify files by name) — do NOT stage `tasks.md` yet
     2. Run all lint commands listed above (if any) to fix formatting — stage any changes they produce
     3. Update `tasks.md` checkbox (race-condition safe — multiple agents may run in parallel):
        - Run `git checkout tasks.md` to get the latest committed version (picks up other agents' checkboxes)
        - Use the **Edit** tool to change ONLY your task's `- [ ]` to `- [x]` — do NOT rewrite the whole file
        - Stage `tasks.md` with `git add`
     4. Commit ALL changes together (code + checkbox + lint fixes) following the `conventional-commits` skill (`skills/conventional-commits/SKILL.md`). Format: `<type>[optional scope]: <task-number> <description>` (e.g., `feat: 1.1 add UserSearch entity`, `test: 2.3 add unit tests for search service`). Choose the type that best matches the task — refer to the skill for the full type list and rules.
   - Do NOT batch multiple tasks into one commit — one commit per task, no exceptions
   - After the commit, report back: "DONE: <task-number> <task-description>"
   - Only add code comments for business logic that is not obvious from the code — if good naming makes it clear, skip the comment
   - Do NOT ask questions — specs should be complete. If something is genuinely ambiguous, skip it and flag it
   - **Language**: All output and reports MUST be in Traditional Chinese. Code and code comments MUST be in English.
   ```

   **Dispatch rules:**
   - Use the **Agent** tool with `run_in_background: true` for ALL worker agents
   - Give each agent a descriptive `name` (e.g., `"vue-engineer-group1"`, `"dotnet-engineer-search"`)
   - Dispatch agents that can run in parallel **simultaneously** (multiple Agent calls in one message)
   - For sequential phases (e.g., review after implementation), wait for background agents to complete before dispatching the next phase
   - You will be **automatically notified** when each background agent completes — do NOT poll or sleep

   **Phase execution (mandatory, in order):**

   - **Phase 1 — Parallel development**: Dispatch all implementation agents in background
   - **Phase 2 — Code Review + Security Review**: After Phase 1 completes, dispatch review-engineer + security-engineer in background (parallel)
   - **Phase 3 — E2E Verification**: After reviews pass, dispatch qa-engineer in background
   - **Phase 4 — Documentation**: After QA passes, dispatch technical-writer in background

   If review or QA fails: dispatch the responsible agent to fix, then re-verify (max 2 retries). Only pause and report to user if still failing.

8. **Interactive control — respond to user messages**

   While agents are running in the background, you remain available in the main conversation. Respond to user messages:

   - **"status" / "進度"** — show current phase, which agents are running, which tasks are done
   - **"pause" / "暫停"** — stop dispatching new agents (already-running agents will finish)
   - **"skip <task>"** — mark a task as skipped and continue
   - **"dispatch <agent> <instruction>"** — manually dispatch a specific agent with custom instructions
   - **"reprioritize"** — re-read tasks.md and adjust dispatch order
   - **Any other message** — interpret as orchestrator instruction and act accordingly

   When a background agent completes, announce briefly:
   ```
   [agent-name] completed: <summary of what was done>
   Progress: N/M tasks
   ```

9. **After all phases complete, verify and report**

   - Re-read `tasks.md` and verify all completed tasks are checked `- [x]`
   - If any completed task was missed, update it now as a safety net
   - Show final status:

   **On completion:**
   ```
   ## Implementation Complete

   **Change:** <change-name>
   **Progress:** M/M tasks complete

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

   All tasks complete! Run `/esdd-archive <name>` to archive this change.
   ```

   **On pause (issue encountered):**
   ```
   ## Implementation Paused

   **Change:** <change-name>
   **Progress:** N/M tasks complete

   ### Issue Encountered
   <description of the issue>

   ### Remaining Tasks
   [list of pending tasks]

   **Options:**
   1. Fix issues and re-run `/esdd-apply <name>`
   2. Update specs and re-run
   3. Other approach

   你想怎麼處理？
   ```

10. **Consolidate commits for reviewer**

    Skip if `commit_squash` is explicitly set to `none` in `feature-spec/config.yaml`, or if total esdd-apply commits ≤ 3. Runs by default — no configuration needed.

    **Purpose**: Agent execution produces one commit per task for race-condition safety. This step reorganizes them into reviewer-friendly groups. Agent behavior is completely unaffected.

    **Algorithm:**

    a. **Record base and create backup**:
       - `BASE_SHA`: parent of the first esdd-apply commit (noted from Step 3)
       - `git tag esdd-pre-squash`

    b. **Collect and classify commits**:
       - `git log --oneline --format="%H %s" $BASE_SHA..HEAD`
       - For each commit, also get changed files: `git show --stat --format="" <sha>`
       - Read task group headings from `tasks.md` (`## N. <heading>` lines)

       Classify every commit into a **group**:
       1. **Task commits** (message matches `(\d+)\.\d+`) → map to task group by the integer before the dot
       2. **Non-task commits** — analyze their content and merge into the most relevant task group:
          - Review/security fix commits → merge into the group whose files they touched most
          - Pre-lint cleanup → merge into the first task group
          - QA fix commits → merge into the group whose tests/code they fixed
          - If genuinely unrelated to any group (e.g., spec-only changes) → keep as standalone `pick`

    c. **Determine reorder strategy**:

       Check if commits are interleaved (parallel agents produced alternating group commits):
       - Walk the classified commit list; if any group appears in more than one contiguous run → **interleaved**

       **If interleaved — attempt reorder**:
       1. Build a reordered sequence: group all same-group commits together, preserving relative order within each group, ordered by first appearance of each group
       2. Write a `GIT_SEQUENCE_EDITOR` script that outputs this reordered todo (all as `pick`)
       3. Execute: `GIT_SEQUENCE_EDITOR="/tmp/esdd-reorder.sh" git rebase -i $BASE_SHA`
       4. If rebase fails (conflict) → `git rebase --abort` and fall back to **consecutive-only mode** (skip reorder, proceed to step d with original order)

       **If non-interleaved** → proceed directly to step d.

    d. **Squash within groups (single-pass rebase)**:
       - Write `/tmp/esdd-squash.sh` that transforms the rebase todo:
         - Track current group as we walk commits top-to-bottom
         - First commit of each group → `pick`
         - Subsequent consecutive commits in the same group → `fixup`
         - Standalone non-task commits (not merged into any group) → `pick`
       - After each group's last `fixup`, insert an `exec` line to reword the commit:
         ```
         exec git commit --amend -m "<new message>"
         ```
         Where `<new message>` is a reviewer-friendly summary written by the orchestrator:
         - Format: `<type>: <concise summary of all changes in this group>`
         - Type follows conventional commits (`feat`, `fix`, `test`, `docs`, `style`, `chore`, etc.)
         - Derive from the task group heading + actual changes, NOT from individual task numbers
         - Example: `feat: add user search API with filtering and pagination`
         - Example: `feat: add search page with input, results list, and empty state`
         - Example: `test: add unit and E2E tests for user search`
         - Non-task standalone commits keep their original message
       - Execute:
         ```bash
         chmod +x /tmp/esdd-squash.sh
         GIT_SEQUENCE_EDITOR="/tmp/esdd-squash.sh" git rebase -i $BASE_SHA
         ```

    e. **Verify integrity**:
       ```bash
       git diff esdd-pre-squash HEAD
       ```
       - Empty diff → success. Delete tag: `git tag -d esdd-pre-squash`
       - Non-empty diff → `git reset --hard esdd-pre-squash`, report: "Commit consolidation failed, original commits preserved."

       Show the consolidated commit log and announce:
       ```
       ## Commits consolidated for review

       <git log --oneline $BASE_SHA..HEAD>

       (original per-task commits squashed into N reviewer-friendly commits)
       ```

    **Abort conditions** — keep original per-task commits and report why:
    - Total commits ≤ 3 (not worth squashing)
    - Rebase encounters merge conflicts (after reorder fallback already attempted)
    - Verification diff is non-empty
    - Any step fails unexpectedly

---

## Guardrails

- **You ARE the orchestrator** — do NOT spawn a separate orchestrator agent. You dispatch worker agents directly.
- **All worker agents run in background** (`run_in_background: true`) — this keeps the main conversation responsive to user input.
- **Specs are the single source of truth** — avoid asking questions unless something is truly blocking and cannot be reasonably inferred. When in doubt, make a reasonable decision and flag it in the report.
- Always read ALL context files before dispatching agents
- Only dispatch agents for PENDING tasks (skip completed `- [x]` tasks)
- Agents MUST update the task checkbox in `tasks.md`, run lint commands, and include everything in the same commit — one atomic commit per task
- If `lint_commands` are configured in `config.yaml`, agents MUST run them before every commit — no exceptions
- If a task genuinely cannot be implemented (missing dependency, unclear spec), skip it and flag it in the report — do NOT block the entire pipeline
- Keep code changes minimal and scoped to each task
- **One commit per task during execution** — each task gets its own commit using Conventional Commits: `<type>: <task-number> <description>`. Unless `commit_squash: none`, these are automatically reorganized into reviewer-friendly groups in Step 10 (reorder interleaved commits, squash per group, reword with clear summaries, merge non-task commits into relevant groups).
- Work on the current branch — do NOT create or switch branches
- Code review and QA are mandatory steps — do NOT skip them
- If review or QA fails, auto-dispatch fixes immediately (max 2 retry rounds). Only pause and report to user if still failing after retries.
- Pass only RELEVANT specs to each agent (not all specs) to keep context focused
- When background agents complete, briefly announce results to the user — don't wait for them to ask
