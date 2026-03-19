---
name: eli-apply-all
description: >
  Run /eli-apply sequentially on multiple changes. Use when the user has
  several prepared changes and wants to batch-implement them unattended.
user-invocable: true
---

Run `/eli-apply` on multiple changes sequentially. The main Claude acts as orchestrator for the entire batch — dispatching worker agents in the background while remaining responsive to user messages.

---

**Input**: Optionally specify change names in order (e.g., `/eli-apply-all add-user-registration add-user-profile add-user-roles`). If omitted, auto-detect.

**Steps**

1. **Discover active changes**

   List all directories under `eli-spec/changes/` (excluding `archive/`).
   Filter to only changes that have pending tasks (`- [ ]` in `tasks.md`).

   If no pending changes found:
   - Report: "No pending changes found. Run `/eli-propose` first."
   - Stop.

2. **Determine execution order**

   **If names are provided as arguments:** use that exact order.

   **If no arguments:** show the list and ask the user to confirm or reorder:

   ```
   ## 批次套用

   發現 N 個待處理的變更：

   1. add-user-registration（5 待處理 / 8 總計）
   2. add-user-profile（3 待處理 / 3 總計）
   3. add-user-roles（4 待處理 / 4 總計）

   按此順序執行？或指定不同順序（例如 "3, 1, 2"）：
   ```

   Use **AskUserQuestion** to let the user confirm or reorder.
   This is the **only** question asked — after confirmation, execution begins automatically.

3. **Run each change sequentially**

   For each change in order:

   a. Show progress header:
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   [N/M] Applying: <change-name>
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```

   b. Record start time for this change.

   c. Execute the full `/eli-apply` logic for this change (Steps 3-8 from `eli-apply/SKILL.md`):
      - Read context files
      - Parse tasks
      - **Act as orchestrator directly** — dispatch worker agents in background
      - Follow all phases (implementation → review → QA → docs)
      - Verify checkboxes after completion
      - **Do NOT ask questions about implementation decisions** — make reasonable choices and flag ambiguities in the report

   d. Record end time. Calculate duration for this change.

   e. Record the result: COMPLETE or PAUSED (with reason)

   f. Announce change result briefly, then **automatically proceed to next change**:
   ```
   [N/M] <change-name>: COMPLETE (8/8 tasks, 25m)
   Proceeding to next change...
   ```

   g. **If a change pauses (review/QA failure after retries):**
      - Record the failure reason
      - **Continue to the next change** — do NOT stop the entire batch
      - The user can fix paused changes later with `/eli-apply <name>`

4. **Show final batch report**

   ```
   ## Batch Apply Complete

   **Total duration:** Xh Ym

   **Results:**
   - [x] add-user-registration — COMPLETE (8/8 tasks, 25m)
   - [ ] add-user-profile — PAUSED (code review failed after 2 retries, 18m)
   - [x] add-user-roles — COMPLETE (4/4 tasks, 12m)

   **Summary:** 2/3 changes completed, 1 paused

   **Paused changes:**
   - `add-user-profile`: [reason]. Fix and re-run with `/eli-apply add-user-profile`
   ```

---

## Interactive Control

While the batch is running, the user can send messages at any time. Respond to them:

- **"status" / "進度"** — show batch progress: which change is active, current phase, running agents, overall N/M changes
- **"skip" / "跳過"** — skip the current change, move to next
- **"stop" / "停止"** — stop dispatching new agents/changes after current agents finish, show partial report
- **"skip <change-name>"** — remove a specific upcoming change from the queue
- **Any other message** — interpret as orchestrator instruction for the current change

After responding to the user, **resume batch execution automatically** — do NOT wait for further input.

Example interaction:
```
User: 進度？
You:  ## Batch Progress
      [2/3] Currently applying: add-user-profile
      Phase 1: vue-engineer running (2/3 tasks done), dotnet-engineer completed
      Overall: 1/3 changes complete

      (continuing...)
```

---

## Guardrails

- **You ARE the orchestrator** for the entire batch — do NOT spawn a separate orchestrator agent
- **All worker agents run in background** (`run_in_background: true`)
- Ask the user to confirm execution order **once** — then run automatically
- **Do NOT ask implementation questions** — make reasonable decisions and flag ambiguities in the report
- **Do NOT stop the batch if one change fails** — skip it and continue to next
- **After responding to user messages, resume automatically** — never wait for follow-up input unless the user explicitly says "stop"
- Each change follows the full `/eli-apply` pipeline (all phases mandatory)
- Each change runs on the current branch — do NOT create or switch branches
- If a change has no pending tasks (all `- [x]`), skip it and note in the report
- Track and report duration for each change and total batch time
