# Eli Workflow

Claude Code plugin — spec-driven multi-agent development team for Vue/Nuxt + ASP.NET projects.

Combines **SDD** (Spec-Driven Development), **DDD** (Domain-Driven Design), and **TDD** (Test-Driven Development) into an automated pipeline.

## Workflow

```
/eli-workflow:init → /eli-workflow:propose (auto-validate) → /eli-workflow:apply → /eli-workflow:archive
```

1. **Init** — auto-detect project context, create `eli-spec/` directory
2. **Propose** — generate specs (SDD), domain model (DDD), API contract, tasks (TDD structure). Auto-validates and fixes until all checks pass.
3. **Apply** — orchestrator dispatches agent team to implement in parallel, review, and verify
4. **Archive** — sync specs and move completed change to archive

## Prerequisites

Enable Agent Teams (required for multi-agent dispatch):

```jsonc
// ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

## Install

**Step 1** — open Claude Code:

```bash
claude
```

**Step 2** — add the marketplace (one-time):

```
/plugin marketplace add bingeli1379/eli-workflow
```

**Step 3** — install the plugin:

```
/plugin install eli-workflow@eli-workflow --scope local
```

**Step 4** — restart Claude Code to load the plugin.

## Uninstall

```
/plugin uninstall eli-workflow@eli-workflow --scope local
```

## Usage

### 1. Initialize (once per project)

```
/eli-workflow:init
```

### 2. Propose a change

```
/eli-workflow:propose add user search feature for admin dashboard
```

Creates `eli-spec/changes/add-user-search/` with:
- `proposal.md` — what & why
- `design.md` — how (domain model, API contract, shared types, decisions)
- `specs/<capability>/spec.md` — acceptance criteria (WHEN/THEN)
- `tasks.md` — TDD-structured implementation checklist

Automatically validates and fixes all artifacts before completion.

### 3. Implement

```
/eli-workflow:apply add-user-search
```

The orchestrator dispatches agents through a 4-phase pipeline:

```
Phase 1 (parallel): QA writes E2E tests + Backend TDD + Frontend TDD
Phase 2 (parallel): Code Review + Security Review
Phase 3:            QA runs E2E → if FAIL → fix → retry (max 2)
Phase 4:            Documentation
```

No questions asked — specs are the single source of truth.

### 4. Archive

```
/eli-workflow:archive add-user-search
```

## Agents

| Agent | Role |
|---|---|
| `orchestrator` | Tech Lead — analyzes specs, dispatches agents, coordinates pipeline |
| `architect` | Software Architect — system design (integrated into `/eli-workflow:propose`) |
| `vue-engineer` | Frontend — Vue 3 / Nuxt 4, TDD, Atomic Design, TailwindCSS |
| `dotnet-engineer` | Backend — ASP.NET Core, TDD, Clean Architecture, EF Core |
| `electron-engineer` | Desktop — Electron main process, IPC, preload, native OS, packaging |
| `database-engineer` | Database — schema design, migration strategy, query optimization, indexing |
| `devops-engineer` | DevOps — Docker, Kubernetes, GitHub Actions CI/CD, infrastructure |
| `performance-engineer` | Performance — Core Web Vitals, bundle analysis, API profiling, caching |
| `review-engineer` | Code quality — architecture compliance, patterns, performance |
| `security-engineer` | Security — OWASP, injection, auth, dependency risks |
| `qa-engineer` | QA — Playwright E2E acceptance testing against spec scenarios |
| `technical-writer` | Documentation — API docs, changelogs, README, ADRs |

## Bundled Skills (44)

All skills are included — no additional installation needed.

### Vue / Frontend (21)

`vue` · `nuxt` · `pinia` · `vite` · `vitest` · `vue-best-practices` · `vue-development-guides` · `vue-debug-guides` · `vue-router-best-practices` · `vue-pinia-best-practices` · `vue-testing-best-practices` · `vue-jsx-best-practices` · `vueuse-functions` · `create-adaptable-composable` · `tailwindcss` · `shadcn-vue` · `accessibility` · `antfu` · `unocss` · `pnpm` · `web-design-guidelines`

### .NET / Backend (6)

`dotnet-clean-architecture` · `dotnet-ef-core` · `dotnet-testing` · `dotnet-minimal-api` · `dotnet-ddd` · `dotnet-error-handling`

### Electron (1)

`electron-dev`

### Database (3)

`sql-expert` · `sql-query-optimization` · `database-schema-design`

### DevOps (4)

`devops-engineer` · `kubernetes` · `dotnet-docker` · `dotnet-cicd`

### Performance (3)

`web-performance` · `core-web-vitals` · `dotnet-caching`

### E2E Testing (1)

`playwright`

### Security (4)

`security-differential-review` · `security-insecure-defaults` · `security-sharp-edges` · `security-supply-chain`

### Workflow (1)

`conventional-commits`

## Development Methodology

| Phase | Methodology | What Happens |
|---|---|---|
| Propose | **SDD** | Specs written before code — WHEN/THEN acceptance criteria |
| Propose | **DDD** | Domain model defined — aggregates, value objects, events |
| Propose | **Contract-First** | API contract + shared types enable parallel development |
| Apply | **TDD** | Frontend/backend write unit tests FIRST (Red → Green → Refactor) |
| Apply | **E2E** | QA writes Playwright tests from specs, runs after implementation |

## Spec Directory Structure

```
eli-spec/
  config.yaml               # Project context (auto-generated)
  specs/                     # Accumulated main specs
    <capability>/spec.md
  changes/
    <name>/
      proposal.md            # What & why
      design.md              # How (domain model, API contract, shared types, decisions)
      tasks.md               # TDD-structured implementation checklist
      specs/                 # Delta specs (acceptance criteria)
        <capability>/spec.md
    archive/
      YYYY-MM-DD-<name>/     # Archived changes
```

## Customization

- Edit `agents/` to adjust role definitions, tech stack, or coding standards
- Edit `skills/propose/templates/` to customize artifact templates
- Edit `eli-spec/config.yaml` in your project to set project-specific context and rules

## Credits

Skills bundled from:
- [anthropics/skills](https://github.com/anthropics/skills) — Vue ecosystem skills
- [codewithmukesh/dotnet-claude-kit](https://github.com/codewithmukesh/dotnet-claude-kit) — .NET skills
- [trailofbits/skills](https://github.com/trailofbits/skills) — Security skills
- [secondsky/claude-skills](https://github.com/secondsky/claude-skills) — Playwright, shadcn-vue
- [chrisvoncsefalvay/claude-skills](https://github.com/chrisvoncsefalvay/claude-skills) — Electron
- [Jamie-BitFlight/claude_skills](https://github.com/Jamie-BitFlight/claude_skills) — Conventional Commits
- [blencorp/claude-code-kit](https://github.com/blencorp/claude-code-kit) — TailwindCSS
- [airowe/claude-a11y-skill](https://github.com/airowe/claude-a11y-skill) — Accessibility
- [QuestForTech-Investments/claude-code-skills](https://github.com/QuestForTech-Investments/claude-code-skills) — SQL Expert
- [Jeffallan/claude-skills](https://github.com/Jeffallan/claude-skills) — DevOps, Kubernetes
- [addyosmani/web-quality-skills](https://github.com/addyosmani/web-quality-skills) — Web Performance, Core Web Vitals
