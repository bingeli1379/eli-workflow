---
name: esdd-archive
description: >
  Archive completed changes. If a name is given, archive that specific change.
  If omitted, auto-scan and batch-archive all fully completed changes.
  Syncs delta specs to main specs by default.
user-invocable: true
---

Archive completed changes. Verifies completion status, syncs delta specs to main specs, then moves the change to the archive.

---

**Input**: Optionally specify a change name (e.g., `/esdd-archive add-user-search`). If omitted, auto-scan for all completed changes.

**Steps**

1. **Select change(s) to archive**

   **If a name is provided:** Use that single change. Go to step 2.

   **If no name is provided (batch mode):**
   - List all directories under `feature-spec/changes/` (excluding `archive/`)
   - If none exist, report error: "No active changes to archive."
   - For each change, read its `tasks.md` and count `- [ ]` vs `- [x]`
   - Collect changes where **all tasks are complete** (zero `- [ ]` remaining), or where `tasks.md` does not exist
   - If no changes qualify, report: "No fully completed changes found." and list each change with its completion status (e.g., `add-user-search: 3/5 tasks complete`)
   - If one or more qualify, display them and proceed to archive **all** of them sequentially (steps 2–4 for each)

   **IMPORTANT**: Batch mode does NOT ask for confirmation — it archives all fully completed changes automatically.

2. **Check task completion status**

   Read `feature-spec/changes/<name>/tasks.md`:
   - Count tasks marked `- [ ]` (incomplete) vs `- [x]` (complete)
   - Display: "Tasks: N/M complete"

   **If incomplete tasks found (only possible when name is explicitly provided):**
   - Display warning showing count and list of incomplete tasks
   - Use **AskUserQuestion** to confirm: "Archive with N incomplete tasks?" / "Cancel"
   - Proceed only if user confirms

   **If no tasks.md exists:** Proceed without task-related warning.

3. **Sync delta specs to main specs**

   Check for delta specs at `feature-spec/changes/<name>/specs/`.

   **If no delta specs exist:** Skip to step 4.

   **If delta specs exist — sync automatically (default behavior):**
   - For each `feature-spec/changes/<name>/specs/<capability>/spec.md`:
     - Check if corresponding main spec exists at `feature-spec/specs/<capability>/spec.md`
     - Determine action: CREATE (new capability) or UPDATE (existing capability)
   - Display summary:
     ```
     Delta specs synced:
     - user-search-api: CREATE (new spec)
     - user-search-ui: UPDATE (existing spec)
     ```
   - For CREATE: copy `feature-spec/changes/<name>/specs/<cap>/spec.md` to `feature-spec/specs/<cap>/spec.md`
   - For UPDATE: overwrite the main spec with the delta spec
   - Create `feature-spec/specs/<cap>/` directory if needed
   - Report: "✓ Synced N specs to main"

4. **Perform the archive**

   Generate target path: `feature-spec/changes/archive/YYYY-MM-DD-<change-name>/`

   **Check if target already exists:**
   - If yes: append a counter suffix (e.g., `YYYY-MM-DD-<name>-2`)
   - Move the change directory to archive:
     ```bash
     mkdir -p feature-spec/changes/archive
     mv feature-spec/changes/<name> feature-spec/changes/archive/YYYY-MM-DD-<name>
     ```

5. **Display summary**

   **Single change:**
   ```
   ## Archive Complete

   **Change:** <change-name>
   **Archived to:** feature-spec/changes/archive/YYYY-MM-DD-<name>/
   **Tasks:** M/M complete ✓
   **Specs:** ✓ Synced to main specs (or "No delta specs")
   ```

   **Single change with warnings (explicitly named, incomplete tasks):**
   ```
   ## Archive Complete (with warnings)

   **Change:** <change-name>
   **Archived to:** feature-spec/changes/archive/YYYY-MM-DD-<name>/

   **Warnings:**
   - Archived with N incomplete tasks
   ```

   **Batch mode:**
   ```
   ## Batch Archive Complete

   Archived N change(s):

   | Change | Tasks | Specs Synced | Archived To |
   |--------|-------|-------------|-------------|
   | add-user-search | 5/5 ✓ | 2 synced | archive/2026-03-22-add-user-search/ |
   | fix-login-bug | 3/3 ✓ | No delta specs | archive/2026-03-22-fix-login-bug/ |

   Skipped M change(s) with incomplete tasks:
   - refactor-auth: 2/4 tasks complete
   ```

---

## Guardrails

- Batch mode (no name provided) only archives fully completed changes — never archives incomplete ones without explicit naming
- When a name is explicitly provided, allow archiving incomplete changes with user confirmation
- Delta specs are always synced to main specs by default (no prompt)
- Preserve all files when moving to archive (the directory moves as-is)
- Show clear summary of what happened
- Use today's date (YYYY-MM-DD) for archive directory prefix
- Handle duplicate archive names gracefully (append counter)
- Never delete the original change directory — always move it
