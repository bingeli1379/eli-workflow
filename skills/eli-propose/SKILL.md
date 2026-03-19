---
name: eli-propose
description: >
  Generate spec artifacts (proposal, design, tasks, specs) for a new change.
  Use when the user wants to describe what they want to build and get a complete
  proposal with design, specs, and tasks ready for implementation.
user-invocable: true
---

Generate a complete set of spec artifacts for a new change — proposal, design, specs, and tasks — all in one step. Follows **SDD (Spec-Driven Development)** with **DDD (Domain-Driven Design)** domain modeling.

Artifacts created:
- `proposal.md` — what & why
- `design.md` — how (technical decisions, **domain model**, **API contract**, **shared types**)
- `specs/<capability>/spec.md` — acceptance criteria (WHEN/THEN)
- `tasks.md` — implementation checklist grouped by agent type (**TDD-style**: test first → implement → refactor)

After all artifacts are created, **automatically runs validation** (`validate` skill logic) and fixes any issues until all checks pass.

---

**Input**: The argument is a description of what the user wants to build, OR a kebab-case change name.

**Steps**

1. **If no clear input provided, ask what they want to build**

   Use the **AskUserQuestion tool** (open-ended, no preset options) to ask:
   > "What change do you want to work on? Describe what you want to build or fix."

   From their description, derive a kebab-case name (e.g., "add user authentication" → `add-user-auth`).

   **IMPORTANT**: Do NOT proceed without understanding what the user wants to build.

2. **Ensure eli-spec is initialized**

   Check if `eli-spec/config.yaml` exists. If not, execute the `init` skill logic to initialize the directory structure and auto-detect project context.

   If already initialized, read `eli-spec/config.yaml` for project context.

3. **Create the change directory**

   ```
   eli-spec/changes/<name>/
   ```

   If a change with that name already exists, use **AskUserQuestion** to ask if user wants to continue it or create a new one with a different name.

4. **Read existing context**

   - Read `eli-spec/config.yaml` for project context (tech stack, conventions, rules)
   - Read `eli-spec/specs/` for existing main specs (to understand what capabilities already exist)
   - These inform artifact generation but are NOT copied into artifact files

5. **Clarify requirements and define feature boundaries**

   Before generating any artifact, analyze the user's description and proactively clarify:

   **a. Identify ambiguities** — look for:
   - Vague scope ("improve search" → what aspects? full-text? filters? sorting?)
   - Undefined behavior ("handle errors" → which errors? how should the UI respond?)
   - Missing edge cases (what if no results? what if input is empty? concurrent access?)
   - Implicit assumptions about existing systems

   **b. Define feature boundaries** — explicitly call out:
   - What IS included in this change (in-scope)
   - What is NOT included (out-of-scope) — related features that should be separate changes
   - Where this change interfaces with existing systems (integration points)

   **c. Assess change size** — estimate the scope and flag if it's too large:
   - Count the number of distinct capabilities / features involved
   - Consider how many layers are affected (frontend, backend, database, infrastructure)
   - If the change touches 3+ independent capabilities, or would produce 15+ tasks, it is likely too large
   - Suggest splitting into smaller changes that are **easy for humans to review** — each change should be:
     - Reviewable in one sitting (a reviewer can understand the full diff without losing context)
     - Self-contained (makes sense on its own, doesn't leave the codebase in a broken state)
     - Focused on one logical concern (one feature, one refactor, one migration — not mixed)
   - Each split becomes its own `/eli-propose` → `/eli-apply` cycle
   - Example: "Add user management" could split into: `add-user-registration`, `add-user-profile`, `add-user-roles`

   **d. Ask the user** using **AskUserQuestion** with a structured summary:
   ```
   Before I generate the spec, I want to confirm the scope:

   **In-Scope:**
   - [feature 1]
   - [feature 2]

   **Out-of-Scope (separate changes):**
   - [related feature that should be its own change]

   **Scope Assessment:** [OK / Too Large]
   [If too large: suggest how to split, e.g.:]
   > This change covers X independent capabilities. I'd recommend splitting into:
   > 1. `/eli-propose add-xxx` — [description]
   > 2. `/eli-propose add-yyy` — [description]
   > Want to proceed as-is or split?

   **Questions:**
   1. [specific question about unclear behavior]
   2. [specific question about edge case]
   ```

   - Ask all questions in ONE message, not one at a time
   - If the user's description is already detailed and unambiguous, confirm the scope briefly and proceed
   - After the user responds, incorporate their answers before generating artifacts

   **e. If user agrees to split** — generate all changes sequentially in this same session:
   - Scope and boundaries for ALL changes were already defined in step 5c/5d, so no further questions needed
   - For each sub-change, run Step 6 (generate artifacts) in order
   - Generate them **sequentially, not in parallel** — earlier changes inform later ones (shared types, API boundaries, domain model consistency)
   - After all changes are generated, show a combined summary listing all created changes
   - The user can then run `/eli-apply-all` to implement them in batch

6. **Generate artifacts in dependency order**

   Generate each artifact following the templates in this skill's `templates/` directory. The dependency order is:

   ```
   proposal (standalone)
       ↓
   design (depends on: proposal)
       ↓
   specs (depends on: proposal, design)
       ↓
   tasks (depends on: proposal, design, specs)
   ```

   For each artifact:
   - Read the corresponding template from `templates/` for structure guidance
   - Read completed dependency artifacts for context
   - Apply project context from `config.yaml` as constraints (do NOT copy into the file)
   - Write the artifact file
   - Show brief progress: "Created `<artifact>`"

   **a. proposal.md**
   - Fill in Why (motivation, min 50 chars), What Changes, Capabilities (new/modified), Impact
   - Capabilities must use kebab-case names (these become `specs/<name>/` directories)
   - Impact should clearly indicate which layers are affected (Backend, Frontend, API, Database, etc.)

   **b. design.md** — dispatch to **architect agent** (`agents/architect.md`)

   Use the **Agent** tool to spawn the architect agent with:
   - `subagent_type`: `"eli-workflow:architect"`
   - Prompt: the proposal.md content, project context from config.yaml, existing specs from `eli-spec/specs/`, and the design.md template from `templates/`
   - Instruct the architect to write `design.md` directly to `eli-spec/changes/<name>/design.md`

   The architect agent will produce:
   - Context, Goals/Non-Goals, Decisions (with alternatives considered), Risks/Trade-offs
   - **Domain Model (DDD)**: Bounded contexts, aggregates (root + children + invariants), value objects, domain events
   - **API Contract**: Every endpoint (METHOD, path, request/response schema, status codes, auth)
   - **Shared Types**: TypeScript interfaces and C# DTOs as integration contract
   - Each decision with justification and rejected alternatives
   - Risks with mitigation strategies

   Wait for the architect agent to complete, then read the generated `design.md` before proceeding to specs.

   **c. specs/<capability>/spec.md** (one per capability from proposal)
   - Read proposal.md and design.md first
   - For EACH capability listed in proposal's "New Capabilities" and "Modified Capabilities":
     - Create `eli-spec/changes/<name>/specs/<capability-name>/spec.md`
   - Each requirement MUST use `SHALL` or `MUST` keyword
   - Each requirement MUST have at least 2 Scenarios with WHEN/THEN (happy path + edge case minimum)
   - Scenarios should also cover: error cases, authorization (if applicable)

   **d. tasks.md** (follows **TDD** — Red/Green/Refactor cycle)
   - Read proposal.md, design.md, and all specs first
   - **Group by feature/phase**, NOT by agent type. Each group is an independent functional unit (e.g., `User Search`, `Search Suggestions`).
     - GOOD: `## 1. User Search` (contains Backend + Frontend + E2E tasks)
     - BAD: `## 1. Backend - All` (groups by agent, blocks parallelism)
   - **Tag each task with an agent type** in parentheses:
     - `(Backend)` → dotnet-engineer (includes unit tests, TDD style)
     - `(Frontend)` → vue-engineer (includes unit tests, TDD style)
     - `(Electron)` → electron-engineer (main process, IPC, preload, packaging)
     - `(Database)` → database-engineer (schema, migration, indexing)
     - `(DevOps)` → devops-engineer (Docker, CI/CD, K8s)
     - `(Performance)` → performance-engineer
     - `(Security)` → security-engineer (security audit, hardening)
     - `(Documentation)` → technical-writer (API docs, changelog, ADR)
     - `(E2E)` → qa-engineer (Playwright E2E tests for AC verification)
   - The orchestrator reads agent tags and dispatches **multiple agents per group** in parallel (e.g., Backend + Frontend + E2E agents all work on the same group concurrently).
   - **TDD task structure for Backend/Frontend tasks**: Each feature should follow RED → GREEN → REFACTOR:
     1. Write failing unit test first (RED)
     2. Implement minimum code to pass (GREEN)
     3. Refactor if needed (REFACTOR)
   - **E2E tasks**: Each spec WHEN/THEN scenario becomes a Playwright E2E test case
   - Each task: starts with a verb, actionable, scoped to one logical unit
   - Use numbered groups and sub-items: `## 1. User Search` → `- [ ] 1.1 (Backend) Write unit test for ...`
   - Tasks must cover ALL requirements from specs — every spec scenario should be traceable to at least one task
   - **Do NOT create `Test` groups** — unit tests belong inside Backend/Frontend tasks; E2E tests are tagged `(E2E)`
   - **Keep groups independent** — if two groups have cross-dependencies, note the dependency order but prefer designing them to be parallelizable

7. **If an artifact requires user input** (unclear context, ambiguous requirements):
   - Use **AskUserQuestion tool** to clarify
   - Then continue with creation

8. **Auto-validate and fix**

   After all artifacts are created, **automatically** run the validation logic from `validate` skill:
   - Read all artifacts and check against all validation rules
   - If any errors found: **fix them immediately** (edit the artifact files to resolve issues)
   - Re-validate until **all checks pass** (max 3 rounds to avoid infinite loops)
   - If issues persist after 3 rounds, report remaining issues and ask user for input

9. **Show final summary**

   ```
   ## Spec Created: <change-name>

   **Location:** eli-spec/changes/<name>/

   ### Artifacts
   - proposal.md — [one-line summary]
   - design.md — [one-line summary, includes domain model + API contract]
   - specs/<cap-1>/spec.md — [one-line summary]
   - specs/<cap-2>/spec.md — [one-line summary]
   - tasks.md — [N tasks in M groups, TDD structure]

   ### Validation
   ✓ PASS — all checks passed

   Ready for implementation. Run `/eli-apply <name>` to start.
   ```

**Guardrails**

- Create ALL 4 artifact types (proposal, design, specs, tasks). Do NOT skip any.
- Always read dependency artifacts before creating the next one
- Capability names in proposal MUST match `specs/<name>/` directory names exactly
- Tasks MUST be grouped by feature/phase, with each task tagged by agent type in parentheses: `(Backend)`, `(Frontend)`, `(E2E)`, etc.
- Every spec requirement MUST have SHALL/MUST and at least 2 WHEN/THEN scenarios (happy path + edge case)
- Proactively clarify ambiguities and define feature boundaries BEFORE generating artifacts — do NOT guess when scope is unclear
- Ask all clarification questions in one structured message, not one at a time
- Verify each artifact file exists after writing before proceeding to next
- `config.yaml` context and rules are constraints for YOU, not content for artifact files
- Use Traditional Chinese for artifact content (matching user's communication language)
- Code examples and technical terms remain in English
