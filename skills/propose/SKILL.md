---
name: propose
description: >
  Generate spec artifacts (proposal, design, tasks, specs) for a new change.
  Use when the user wants to describe what they want to build and get a complete
  proposal with design, specs, and tasks ready for implementation.
user-invocable: true
license: MIT
metadata:
  author: Eli
  version: "0.3.0"
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

   **b. design.md** (acts as the **Architect's output** — defines everything needed for parallel development)
   - Read proposal.md first
   - Fill in Context, Goals/Non-Goals, Decisions (with alternatives considered), Risks/Trade-offs
   - **Domain Model (DDD)**: Identify bounded contexts, aggregates (root + children + invariants), value objects, domain events. This drives the backend domain layer design.
   - **API Contract**: Define every endpoint (METHOD, path, request/response schema, status codes, auth). This is the contract that enables frontend and backend to develop in parallel.
   - **Shared Types**: Define TypeScript interfaces and C# DTOs that both sides must agree on. These serve as the integration contract.
   - Each decision should justify the chosen approach and explain why alternatives were rejected
   - Risks should include mitigation strategies

   **c. specs/<capability>/spec.md** (one per capability from proposal)
   - Read proposal.md and design.md first
   - For EACH capability listed in proposal's "New Capabilities" and "Modified Capabilities":
     - Create `eli-spec/changes/<name>/specs/<capability-name>/spec.md`
   - Each requirement MUST use `SHALL` or `MUST` keyword
   - Each requirement MUST have at least 2 Scenarios with WHEN/THEN (happy path + edge case minimum)
   - Scenarios should also cover: error cases, authorization (if applicable)

   **d. tasks.md** (follows **TDD** — Red/Green/Refactor cycle)
   - Read proposal.md, design.md, and all specs first
   - Group tasks by agent type with clear prefixes:
     - `Backend - [Area]` → dispatches to dotnet-engineer agent (includes unit tests, TDD style)
     - `Frontend - [Area]` → dispatches to vue-engineer agent (includes unit tests, TDD style)
     - `Electron - [Area]` → dispatches to electron-engineer agent (main process, IPC, preload, packaging)
     - `Database - [Area]` → dispatches to database-engineer agent (schema, migration, indexing)
     - `DevOps - [Area]` → dispatches to devops-engineer agent (Docker, CI/CD, K8s)
     - `Performance - [Area]` → dispatches to performance-engineer agent
     - `Security - [Area]` → dispatches to security-engineer agent (security audit, hardening)
     - `Documentation - [Area]` → dispatches to technical-writer agent (API docs, changelog, ADR)
     - `E2E - [Area]` → dispatches to qa-engineer agent (Playwright E2E tests for AC verification)
     - `Integration` → orchestrator coordinates multi-agent work
   - **TDD task structure for Backend/Frontend groups**: Each feature should follow RED → GREEN → REFACTOR:
     1. Write failing unit test first (RED)
     2. Implement minimum code to pass (GREEN)
     3. Refactor if needed (REFACTOR)
   - **E2E group**: Each spec WHEN/THEN scenario becomes a Playwright E2E test case
   - Each task: starts with a verb, actionable, scoped to one logical unit
   - Use numbered groups and sub-items: `## 1. Backend - Search API` → `- [ ] 1.1 Write unit test for ...`
   - Tasks must cover ALL requirements from specs — every spec scenario should be traceable to at least one task
   - **Do NOT create `Test` groups** — unit tests belong inside Backend/Frontend groups; E2E tests are in `E2E` groups

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

   Ready for implementation. Run `/eli-workflow:apply <name>` to start.
   ```

**Guardrails**

- Create ALL 4 artifact types (proposal, design, specs, tasks). Do NOT skip any.
- Always read dependency artifacts before creating the next one
- Capability names in proposal MUST match `specs/<name>/` directory names exactly
- Tasks MUST be grouped with clear agent-type prefixes (Backend/Frontend/Electron/Database/DevOps/Performance/Security/Documentation/E2E/Integration)
- Every spec requirement MUST have SHALL/MUST and at least 2 WHEN/THEN scenarios (happy path + edge case)
- If context is critically unclear, ask the user — but prefer making reasonable decisions to keep momentum
- Verify each artifact file exists after writing before proceeding to next
- `config.yaml` context and rules are constraints for YOU, not content for artifact files
- Use Traditional Chinese for artifact content (matching user's communication language)
- Code examples and technical terms remain in English
