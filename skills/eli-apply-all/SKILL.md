---
name: eli-apply-all
description: >
  Run /eli-apply sequentially on multiple changes. Use when the user has
  several prepared changes and wants to batch-implement them unattended.
user-invocable: true
---

Run `/eli-apply` on multiple changes sequentially. Designed for unattended execution — confirm the order once, then let it run through all changes automatically.

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
   ## Batch Apply

   Found N changes with pending tasks:

   1. add-user-registration (5 pending / 8 total)
   2. add-user-profile (3 pending / 3 total)
   3. add-user-roles (4 pending / 4 total)

   Run in this order? Or specify a different order (e.g., "3, 1, 2"):
   ```

   Use **AskUserQuestion** to let the user confirm or reorder.
   This is the **only** question asked — after this, everything runs unattended.

3. **Run each change sequentially**

   For each change in order:

   a. Show progress header:
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   [N/M] Applying: <change-name>
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```

   b. Record start time for this change.

   c. Execute the full `/eli-apply` logic for this change in **unattended mode**:
      - The change name is already known — do NOT use AskUserQuestion to select it
      - Read context files
      - Launch orchestrator agent
      - Wait for completion
      - Verify checkboxes
      - **Do NOT ask any questions** — if something would normally pause and ask the user, instead record the issue and move on

   d. Record end time. Calculate duration for this change.

   e. Record the result: COMPLETE or PAUSED (with reason)

   f. **If a change pauses (review/QA failure after retries):**
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

## Guardrails

- Ask the user to confirm execution order **once** — then run everything unattended
- **NEVER ask questions after the batch starts** — no AskUserQuestion, no "What would you like to do?", no pausing for input. If an issue occurs, record it and move to the next change.
- Do NOT stop the batch if one change fails — skip it and continue
- Each change follows the full `/eli-apply` pipeline (all phases mandatory)
- Each change runs on the current branch — do NOT create or switch branches
- If a change has no pending tasks (all `- [x]`), skip it and note in the report
- Track and report duration for each change and total batch time
