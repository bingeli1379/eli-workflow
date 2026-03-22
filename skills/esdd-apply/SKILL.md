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

10. **Consolidate commits (if configured)**

    Skip this step if `commit_squash` is not set to `by_group` in `feature-spec/config.yaml`, or if total esdd-apply commits ≤ 3.

    **Purpose**: Agent execution produces one commit per task for race-condition safety. This post-processing step squashes them into one commit per task group for a cleaner PR history. Agent behavior is completely unaffected.

    **Algorithm:**

    a. **Record the base commit** (`BASE_SHA`): the parent of the first esdd-apply commit in this session (noted from Step 3 — the commit before "chore: pre-lint cleanup" or the first agent commit).

    b. **Create safety backup**: `git tag esdd-pre-squash`

    c. **Map commits to task groups**:
       - `git log --oneline $BASE_SHA..HEAD` to list all commits
       - Extract task group number from each commit message: the **full integer** before the first dot in the task number (e.g., `1.1` → group `1`, `4.3` → group `4`, `10.2` → group `10`). Use regex: `(\d+)\.\d+`
       - **Non-task commits** (no task number pattern, e.g., `chore: pre-lint cleanup`, spec artifact commits, review/QA fix commits without task numbers) → **always `pick`, never squash**. These are distinct operations and must not be merged with each other or with task commits.
       - Get group headings from `tasks.md` (the `## N. <heading>` lines)

    d. **Analyze commit order and choose strategy**:

       Examine the commit sequence to determine if task groups are interleaved (from parallel agents):

       - **Non-interleaved** (all commits from each group are consecutive): safe to squash in place.
         Example: `1.1, 1.2, 1.3, 2.1, 2.2, 4.1, 4.2` → groups are contiguous blocks.
       - **Interleaved** (commits from different groups alternate): only squash consecutive runs.
         Example: `1.1, 3.1, 1.2, 3.2, 1.3` → groups 1 and 3 are interleaved.

       Detection: walk the commit list and check if any group appears in more than one contiguous run. If so, the sequence is interleaved.

    e. **Squash via non-interactive rebase**:
       - Write a shell script to `/tmp/esdd-squash-todo.sh` that transforms the rebase todo file:
         - Walk commits in order
         - Non-task commits → always `pick` (never squash)
         - Task commits: if in the **same group as the previous task commit** AND they are adjacent (no different-group commit in between) → mark as `fixup`
         - Otherwise → `pick`
       - Execute:
         ```bash
         chmod +x /tmp/esdd-squash-todo.sh
         GIT_SEQUENCE_EDITOR="/tmp/esdd-squash-todo.sh" git rebase -i $BASE_SHA
         ```
       - With `GIT_SEQUENCE_EDITOR`, the rebase is fully non-interactive — no user input required.

       **If commits are interleaved**, announce to the user after squash:
       ```
       ⚠ Parallel agent commits were interleaved — squashed consecutive runs only.
       Some groups have multiple commits (e.g., group 1 split into 2 commits).
       Run `git rebase -i` manually if you want to consolidate further.
       ```

    f. **Reword surviving task commits** with group-level summaries:
       - After squash, each group's commit still has its first task's message (e.g., `style: 1.1 remove empty <script setup> from Loading.vue`)
       - Write a message mapping file and an editor script:
         - The mapping maps task group numbers to desired messages (e.g., `1` → `style: remove empty script setup blocks`)
         - Derive the message from the task group heading in `tasks.md`, using the appropriate conventional commit type
       - Execute a second non-interactive rebase:
         ```bash
         GIT_SEQUENCE_EDITOR="sed -i.bak 's/^pick /reword /' \$1" \
         GIT_EDITOR="/tmp/esdd-reword.sh" \
         git rebase -i $BASE_SHA
         ```
         Where `/tmp/esdd-reword.sh` reads the current message from `$1`, determines the group from the task number, looks up the mapping, and writes the new message. Non-task commits keep their original message (the editor script should pass them through unchanged).

    g. **Verify integrity**:
       ```bash
       git diff esdd-pre-squash HEAD
       ```
       - Must produce **no output** (identical final state)
       - If diff is non-empty → `git reset --hard esdd-pre-squash` and report: "Commit consolidation failed, original commits preserved."
       - If clean → `git tag -d esdd-pre-squash` and show the consolidated commit log

    **Abort conditions** — keep original per-task commits and report why:
    - Total commits ≤ 3 (not worth squashing)
    - Rebase encounters merge conflicts
    - Verification diff is non-empty (safety net triggered)
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
- **One commit per task during execution** — each task gets its own commit using Conventional Commits: `<type>: <task-number> <description>`. If `commit_squash: by_group` is configured, these are consolidated post-execution in Step 10.
- Work on the current branch — do NOT create or switch branches
- Code review and QA are mandatory steps — do NOT skip them
- If review or QA fails, auto-dispatch fixes immediately (max 2 retry rounds). Only pause and report to user if still failing after retries.
- Pass only RELEVANT specs to each agent (not all specs) to keep context focused
- When background agents complete, briefly announce results to the user — don't wait for them to ask
