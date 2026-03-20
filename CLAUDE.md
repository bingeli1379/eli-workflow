# Eli Workflow

Spec-driven multi-agent development team plugin for Claude Code.

## Workflow

```
/esdd-init → /esdd-propose → /esdd-validate → /esdd-apply → /esdd-archive
```

## Skills (User-Invocable)

| Command | Description |
|---|---|
| `/esdd-init` | Initialize feature-spec directory and auto-generate config.yaml from project context |
| `/esdd-propose <description>` | Generate spec artifacts (proposal, design, specs, tasks) for a new change |
| `/esdd-validate <change-name>` | Validate spec artifacts against structural and content rules |
| `/esdd-apply <change-name>` | Implement tasks using agent team dispatch (no questions asked) |
| `/esdd-apply-all [names...]` | Batch apply multiple changes sequentially, unattended |
| `/esdd-archive <change-name>` | Archive completed change, sync specs to main |

## Spec Directory Structure

```
feature-spec/
  config.yaml               # Project context (auto-generated on first run)
  specs/                     # Accumulated main specs
    <capability>/spec.md
  changes/
    <name>/
      proposal.md            # What & why
      design.md              # How (domain model, API contract, shared types, decisions)
      tasks.md               # Implementation checklist (grouped by agent)
      specs/                 # Delta specs (acceptance criteria)
        <capability>/spec.md
    archive/
      YYYY-MM-DD-<name>/     # Archived changes
```

## Agent Definitions

Agent role definitions live in `agents/`. The orchestrator reads these at dispatch time.

| Agent | Role |
|---|---|
| `orchestrator` | Tech Lead — task analysis, agent dispatch, progress tracking |
| `architect` | Software Architect — system design, API contracts, integration specs |
| `vue-engineer` | Frontend — Vue 3 / Nuxt 4, Atomic Design, Composable Pattern |
| `dotnet-engineer` | Backend — ASP.NET Core, Clean Architecture, EF Core |
| `review-engineer` | Code quality — architecture compliance, patterns, performance |
| `security-engineer` | Security — OWASP, injection, auth, dependency risks |
| `electron-engineer` | Electron — main process, IPC, preload, native OS, packaging |
| `database-engineer` | Database — schema design, migration strategy, query optimization, indexing |
| `devops-engineer` | DevOps — Docker, Kubernetes, GitHub Actions CI/CD, infrastructure |
| `performance-engineer` | Performance — Core Web Vitals, bundle analysis, API profiling, caching |
| `qa-engineer` | QA — Playwright E2E acceptance testing, spec scenario verification |
| `technical-writer` | Documentation — API docs, changelogs, README, ADRs |

## Bundled Skills

Skills in `skills/` provide domain knowledge that agents can reference.

### Vue / Frontend Skills (from [anthropics/skills](https://github.com/anthropics/skills) community)
- `vue` — Vue 3 Composition API, script setup, reactivity, built-in components
- `nuxt` — Nuxt SSR, auto-imports, file-based routing, server routes
- `pinia` — Pinia store patterns, state/getters/actions
- `vite` — Vite config, plugin API, SSR, build optimization
- `vitest` — Vitest testing framework, Jest-compatible API
- `vue-best-practices` — Composition API + TypeScript standard approach
- `vue-development-guides` — Development best practices collection
- `vue-debug-guides` — Runtime errors, async failures, SSR/hydration issues
- `vue-router-best-practices` — Vue Router 4 patterns, navigation guards
- `vue-pinia-best-practices` — Pinia stores, state management patterns
- `vue-testing-best-practices` — Vitest + Vue Test Utils + Playwright E2E
- `vue-jsx-best-practices` — JSX syntax in Vue
- `vueuse-functions` — VueUse composables for concise, maintainable code
- `create-adaptable-composable` — MaybeRef/MaybeRefOrGetter composable pattern
- `tailwindcss` — Tailwind CSS v4 utility-first patterns, responsive, dark mode (from [blencorp/claude-code-kit](https://github.com/blencorp/claude-code-kit))
- `accessibility` — axe-core runtime a11y audit, WCAG 2.1 AA (from [airowe/claude-a11y-skill](https://github.com/airowe/claude-a11y-skill))
- `antfu` — Anthony Fu's opinionated tooling conventions
- `unocss` — UnoCSS atomic CSS engine, Tailwind superset
- `pnpm` — pnpm package manager, workspaces, catalogs
- `web-design-guidelines` — Web Interface Guidelines compliance

### Electron Skills (from [chrisvoncsefalvay/claude-skills](https://github.com/chrisvoncsefalvay/claude-skills))
- `electron-dev` — Electron scaffolding, IPC patterns, security hardening, auto-update, packaging

### Database Skills
- `sql-expert` — Complex SQL, EXPLAIN analysis, schema design, indexing, zero-downtime migrations (from [QuestForTech-Investments](https://github.com/QuestForTech-Investments/claude-code-skills))
- `sql-query-optimization` — Query optimization patterns, pg_stat_statements, cursor pagination (from [secondsky/claude-skills](https://github.com/secondsky/claude-skills))
- `database-schema-design` — Normalization, relationships, constraints, partitioning (from [secondsky/claude-skills](https://github.com/secondsky/claude-skills))

### DevOps Skills
- `devops-engineer` — Docker, K8s, GitHub Actions, Terraform, deployment strategies (from [Jeffallan/claude-skills](https://github.com/Jeffallan/claude-skills))
- `kubernetes-specialist` — K8s workloads, networking, Helm, troubleshooting, GitOps (from [Jeffallan/claude-skills](https://github.com/Jeffallan/claude-skills))
- `docker` — .NET multi-stage Docker builds, non-root, layer caching (from [codewithmukesh/dotnet-claude-kit](https://github.com/codewithmukesh/dotnet-claude-kit))
- `ci-cd` — GitHub Actions + Azure DevOps pipelines for .NET (from [codewithmukesh/dotnet-claude-kit](https://github.com/codewithmukesh/dotnet-claude-kit))
- `gitlab-glab` — GitLab CLI (glab) for MRs, pipelines, issues, releases (from [henricook/claude-glab-skill](https://github.com/henricook/claude-glab-skill))

### Performance Skills
- `performance` — Critical rendering path, code splitting, image/font optimization (from [addyosmani/web-quality-skills](https://github.com/addyosmani/web-quality-skills))
- `core-web-vitals` — LCP, INP, CLS diagnosis and optimization (from [addyosmani/web-quality-skills](https://github.com/addyosmani/web-quality-skills))
- `caching` — HybridCache, Output Caching, Redis, stampede protection (from [codewithmukesh/dotnet-claude-kit](https://github.com/codewithmukesh/dotnet-claude-kit))

### E2E Testing Skills
- `playwright` — Playwright E2E testing, cross-browser, visual regression, API testing (from [secondsky/claude-skills](https://github.com/secondsky/claude-skills))

### Workflow Skills
- `conventional-commits` — Conventional Commits v1.0.0, semantic versioning, changelog generation (from [Jamie-BitFlight/claude_skills](https://github.com/Jamie-BitFlight/claude_skills))

### .NET Skills (from [codewithmukesh/dotnet-claude-kit](https://github.com/codewithmukesh/dotnet-claude-kit))
- `clean-architecture` — 4-layer layout, dependency inversion, use case handlers
- `ef-core` — DbContext, migrations, interceptors, compiled queries, query optimization
- `minimal-api` — MapGroup, TypedResults, endpoint filters, OpenAPI, rate limiting
- `ddd` — Aggregates, value objects, domain events, strongly-typed IDs
- `error-handling` — Result pattern, ProblemDetails (RFC 9457), FluentValidation
- `resilience` — Polly v8 retry, circuit breaker, timeout, fallback, hedging, rate limiter (from [codewithmukesh/dotnet-claude-kit](https://github.com/codewithmukesh/dotnet-claude-kit))
- `authentication` — JWT Bearer, ASP.NET Identity, policy-based auth, OpenID Connect (from [codewithmukesh/dotnet-claude-kit](https://github.com/codewithmukesh/dotnet-claude-kit))
- `dotnet-grpc` — gRPC server/client, protobuf contracts, streaming, interceptors, health checks (from [managedcode/dotnet-skills](https://github.com/managedcode/dotnet-skills))
- `dotnet-nunit` — NUnit v4, constraint-based assertions, parameterized tests, async patterns (from [managedcode/dotnet-skills](https://github.com/managedcode/dotnet-skills))

### Security Skills (from [trailofbits/skills](https://github.com/trailofbits/skills))
- `differential-review` — Risk-first adaptive security review for PRs and diffs
- `insecure-defaults` — Detect fail-open defaults, hardcoded secrets, weak auth
- `sharp-edges` — Dangerous APIs, footgun designs, configuration cliffs
- `supply-chain-risk-auditor` — Dependency risk audit, single-maintainer detection

## Development Methodology

- **SDD (Spec-Driven Development)**: `/esdd-propose` produces complete specs before any code is written
- **DDD (Domain-Driven Design)**: Domain model (aggregates, value objects, events) defined in `design.md` during propose
- **TDD (Test-Driven Development)**: Frontend and backend agents write unit tests FIRST (Red → Green → Refactor)
- **Contract-First**: API contracts and shared types defined in `design.md` enable parallel frontend/backend development

## Implementation Pipeline

```
Phase 1 (parallel): QA writes E2E tests + Backend TDD + Frontend TDD
Phase 2 (parallel): Code Review + Security Review
Phase 3: QA runs E2E tests → if FAIL → fix agent → retry (max 2)
Phase 4: Documentation
```

## Team Standards

- **Frontend**: Vue 3 Composition API + Nuxt 4, Atomic Design, Composable Pattern, TailwindCSS, TypeScript strict
- **Backend**: ASP.NET Core .NET 8–10, Clean/Layered Architecture, EF Core + Dapper, Polly, Redis, C# 12–13
- **Unit Tests**: Written by frontend/backend agents themselves (TDD), new code 100% coverage
- **E2E Tests**: Written by QA agent with Playwright, verifies all spec WHEN/THEN scenarios
- **Language**: Traditional Chinese communication, English code and comments
- **Commits**: Each task = one commit, using Conventional Commits: `<type>: <task-number> <description>`
