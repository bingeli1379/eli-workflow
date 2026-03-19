---
name: eli-archive
description: >
  Archive a completed change. Checks task completion, syncs delta specs
  to main specs, and moves the change to the archive directory.
user-invocable: true
license: MIT
metadata:
  author: Eli
  version: "0.4.0"
---

Archive a completed change. Verifies completion status, offers to sync delta specs to main specs, then moves the change to the archive.

---

**Input**: Optionally specify a change name (e.g., `/eli-archive add-user-search`). If omitted, auto-detect.

**Steps**

1. **Select the change**

   If a name is provided, use it. Otherwise:
   - List directories under `eli-spec/changes/` (excluding `archive/`)
   - If only one active change exists, auto-select it
   - If multiple, use **AskUserQuestion** to let the user choose
   - If none exist, report error: "No active changes to archive."

   **IMPORTANT**: If multiple changes exist and no name is provided, always let the user choose.

2. **Check task completion status**

   Read `eli-spec/changes/<name>/tasks.md`:
   - Count tasks marked `- [ ]` (incomplete) vs `- [x]` (complete)
   - Display: "Tasks: N/M complete"

   **If incomplete tasks found:**
   - Display warning showing count and list of incomplete tasks
   - Use **AskUserQuestion** to confirm: "Archive with N incomplete tasks?" / "Cancel"
   - Proceed only if user confirms

   **If no tasks.md exists:** Proceed without task-related warning.

3. **Check and sync delta specs**

   Check for delta specs at `eli-spec/changes/<name>/specs/`.

   **If no delta specs exist:** Skip to step 4.

   **If delta specs exist:**
   - For each `eli-spec/changes/<name>/specs/<capability>/spec.md`:
     - Check if corresponding main spec exists at `eli-spec/specs/<capability>/spec.md`
     - Determine action needed: CREATE (new capability) or UPDATE (existing capability)
   - Show summary:
     ```
     Delta specs found:
     - user-search-api: CREATE (new spec)
     - user-search-ui: CREATE (new spec)
     ```
   - Use **AskUserQuestion** with options:
     - "Sync now (recommended)" — copy delta specs to main specs
     - "Archive without syncing" — keep delta specs only in archive

   **If user chooses sync:**
   - For CREATE: copy `eli-spec/changes/<name>/specs/<cap>/spec.md` to `eli-spec/specs/<cap>/spec.md`
   - For UPDATE: overwrite the main spec with the delta spec
   - Create `eli-spec/specs/<cap>/` directory if needed
   - Report: "✓ Synced N specs to main"

4. **Perform the archive**

   Generate target path: `eli-spec/changes/archive/YYYY-MM-DD-<change-name>/`

   **Check if target already exists:**
   - If yes: append a counter suffix (e.g., `YYYY-MM-DD-<name>-2`)
   - Move the change directory to archive:
     ```bash
     mkdir -p eli-spec/changes/archive
     mv eli-spec/changes/<name> eli-spec/changes/archive/YYYY-MM-DD-<name>
     ```

5. **Display summary**

   **On success:**
   ```
   ## Archive Complete

   **Change:** <change-name>
   **Archived to:** eli-spec/changes/archive/YYYY-MM-DD-<name>/
   **Tasks:** M/M complete ✓
   **Specs:** ✓ Synced to main specs (or "Sync skipped" or "No delta specs")
   ```

   **On success with warnings:**
   ```
   ## Archive Complete (with warnings)

   **Change:** <change-name>
   **Archived to:** eli-spec/changes/archive/YYYY-MM-DD-<name>/

   **Warnings:**
   - Archived with N incomplete tasks
   - Delta spec sync was skipped
   ```

---

## Guardrails

- Always prompt for change selection if not provided and multiple exist
- Don't block archive on warnings — just inform and confirm
- Preserve all files when moving to archive (the directory moves as-is)
- Show clear summary of what happened
- Use today's date (YYYY-MM-DD) for archive directory prefix
- Handle duplicate archive names gracefully (append counter)
- Never delete the original change directory — always move it
