<!-- Task groups are organized by feature/phase, NOT by agent type. -->
<!-- Each task is tagged with an agent type in parentheses: (Backend), (Frontend), (E2E), etc. -->
<!-- The orchestrator dispatches multiple agents per group when tasks have different agent tags. -->
<!-- Valid agent tags: Backend, Frontend, Electron, Database, DevOps, Performance, Security, Documentation, E2E -->
<!-- NOTE: Unit tests are included within Backend/Frontend tasks (TDD). -->

<!-- Example: a "User Search" feature -->

## 1. User Search

- [ ] 1.1 (Backend) Write unit test for search endpoint (RED)
- [ ] 1.2 (Backend) Implement search endpoint to pass test (GREEN)
- [ ] 1.3 (Frontend) Write unit test for SearchPage (RED)
- [ ] 1.4 (Frontend) Implement SearchPage to pass test (GREEN)
- [ ] 1.5 (E2E) Write E2E test for user searches by keyword

## 2. Search Suggestions

- [ ] 2.1 (Backend) Write unit test for suggestions endpoint (RED)
- [ ] 2.2 (Backend) Implement suggestions endpoint to pass test (GREEN)
- [ ] 2.3 (Frontend) Write unit test for SearchSuggestions composable (RED)
- [ ] 2.4 (Frontend) Implement SearchSuggestions composable to pass test (GREEN)
- [ ] 2.5 (E2E) Write E2E test for autocomplete suggestions
