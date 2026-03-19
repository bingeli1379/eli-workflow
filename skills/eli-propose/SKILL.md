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

5. **Generate artifacts in dependency order**

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

6. **If an artifact requires user input** (unclear context, ambiguous requirements):
   - Use **AskUserQuestion tool** to clarify
   - Then continue with creation

7. **Auto-validate and fix**

   After all artifacts are created, **automatically** run the validation logic from `validate` skill:
   - Read all artifacts and check against all validation rules
   - If any errors found: **fix them immediately** (edit the artifact files to resolve issues)
   - Re-validate until **all checks pass** (max 3 rounds to avoid infinite loops)
   - If issues persist after 3 rounds, report remaining issues and ask user for input

8. **Show final summary**

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
- If context is critically unclear, ask the user — but prefer making reasonable decisions to keep momentum
- Verify each artifact file exists after writing before proceeding to next
- `config.yaml` context and rules are constraints for YOU, not content for artifact files
- Use Traditional Chinese for artifact content (matching user's communication language)
- Code examples and technical terms remain in English
