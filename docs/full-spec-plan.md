# Plan: Checklist Execution System

## TL;DR
Build a single-user Runbook + Todo manager for engineers performing repeatable operational tasks. The system has three functional areas (Templates, Runs, Todos) with a guided "Today" dashboard. Recommended stack: **Angular 19** (frontend) + **NestJS with TypeScript** (backend) + **SQLite** (database), deployed locally and via Docker.

---

## Stack Recommendation

### Backend: NestJS + TypeScript
- **Why**: NestJS mirrors Angular's architecture (decorators, DI, modules) — minimal mental context-switching. TypeScript across the full stack means one language to master.
- **ORM**: TypeORM (native NestJS support, good SQLite driver, migration support)
- **SQLite driver**: `better-sqlite3` (synchronous, no native compilation issues on most platforms)
- **Testing**: Jest (ships with NestJS)
- **Alternative considered**: Go (leaner binary, but TypeScript consistency wins for productivity)

### Frontend: Angular 19 + TypeScript
- **Why**: User has Angular experience. Modern Angular (standalone components, signals) is lightweight enough for this scope. The spec was designed around Angular.
- **UI**: Angular Material or Tailwind CSS + custom components
- **Markdown**: `ngx-markdown` (as spec'd) with Prism.js for syntax highlighting
- **Drag-and-drop**: Angular CDK `@angular/cdk/drag-drop`
- **Build**: Angular CLI with Vite (default in Angular 19)

### Database: SQLite
- **Why**: Single user, local-first, zero config. Perfect fit.
- **Migrations**: TypeORM migration system

### Deployment
- **Dev**: `ng serve` (frontend) + `nest start --watch` (backend) — separate processes
- **Production**: Docker Compose with two containers (or single container with nginx serving frontend + proxying to backend)

---

## Spec Analysis — Issues & Improvements

### Design gaps to address before implementation

1. **Instance status lifecycle**: Spec only defines `IN_PROGRESS` default. Need `COMPLETED` and `ABANDONED` statuses. Instance should auto-complete when all steps are done.

2. **CASCADE/delete behavior**: Not specified. Recommended:
   - Delete template → delete template_steps (CASCADE). Instances survive (steps are copied).
   - Delete instance → delete instance_steps (CASCADE).
   - Delete template with active instances → either block or allow (instances are self-contained).

3. **Step un-completion**: Spec doesn't address unchecking a completed step. Should be supported (toggle `completed`, recalculate `next_step_id`).

4. **Gap ordering exhaustion**: Position gaps (100, 200, 300) eventually run out with many inserts. Need a rebalancing strategy when gap < 1.

5. **"Today" todo filtering**: Dashboard shows todos but no concept of which todos are "today" vs. all. Options: show all incomplete, or add a `pinned`/`today` flag.

6. **Instance `next_step_id` as FK**: If the column references `instance_step.id`, it creates a circular dependency during instance creation (instance needs step ID, step needs instance ID). Solution: set `next_step_id` after steps are created, in a transaction.

7. **Template deletion with instances**: Spec doesn't specify behavior. Recommend: allow deletion since instances are self-contained snapshots. Optionally soft-delete templates.

8. **Error responses**: Spec doesn't define error format. Recommend standard `{ "statusCode": 400, "message": "...", "error": "Bad Request" }` (NestJS default).

9. **Todo ordering**: No `position` field on todos. Consider adding one for drag-and-drop reordering, or sort by `created_at`.

---

## Phased Implementation Plan

### Phase 0: Copilot Agent Skill Preparation
> Set up AI-assisted development guardrails before any code is written

**0A — Evaluate & Install Existing Skills**

| # | Task | Details |
|---|------|---------|
| 0A.1 | Install `web-design-guidelines` | `npx skills add vercel-labs/agent-skills --skill web-design-guidelines`. Audits UI code against Vercel's Web Interface Guidelines (191K installs, security PASS). Use for reviewing Angular component templates for accessibility, layout, and UX compliance. |
| 0A.2 | Install `tdd` | `npx skills add mattpocock/skills --skill tdd`. TDD with vertical slices and behavior-focused testing (6.9K installs, all security audits PASS). Framework-agnostic — works for both Jest/NestJS and Angular TestBed tests. |
| 0A.3 | Evaluate `ui-ux-pro-max` rules (DO NOT INSTALL) | `nextlevelbuilder/ui-ux-pro-max-skill` has excellent UX rules (99 guidelines, 10 priority categories) BUT: **fails Gen Agent Trust Hub security audit** and is React Native-focused. Extract useful rules (accessibility, forms, navigation, animation) into our custom UI/UX skill instead. |
| 0A.4 | Skip `frontend-design` | `anthropics/skills/frontend-design` is too creative/avant-garde for a developer productivity tool. We need consistency and usability, not "unforgettable" aesthetics. |
| 0A.5 | Skip `webapp-testing` | `anthropics/skills/webapp-testing` is Python Playwright E2E only — not relevant for Jest unit/integration testing. |

**0B — Create Custom Angular 19 Skill (single monolithic SKILL.md)**

| # | Task | Details |
|---|------|---------|
| 0B.1 | Research Angular 19 best practices | Gather official Angular guidelines: standalone components (no NgModules), signals & computed signals, new control flow (`@if`, `@for`, `@switch`), `inject()` over constructor DI, `input()`/`output()`/`model()` signal APIs, zoneless change detection, functional guards/resolvers, typed reactive forms, `HttpClient` with `withFetch()`. |
| 0B.2 | Create `angular-19-best-practices` SKILL.md | Custom skill covering: project structure conventions, component patterns (standalone, signals, OnPush), routing patterns (lazy loading, functional guards), service patterns (inject(), HttpClient), template patterns (new control flow, ngx-markdown integration), testing patterns (TestBed with standalone, component harnesses), Angular CDK usage (drag-drop, a11y). Install into frontend repo `.copilot/skills/`. |

**0C — Create Custom NestJS Skill (single monolithic SKILL.md)**

| # | Task | Details |
|---|------|---------|
| 0C.1 | Research NestJS best practices | Gather official NestJS patterns: module organization, controller/service separation, DTOs with class-validator, TypeORM entity patterns, repository pattern, exception filters, pipes for validation, interceptors, testing with @nestjs/testing (createTestingModule, overrideProvider), SQLite-specific patterns, migration strategies. |
| 0C.2 | Create `nestjs-best-practices` SKILL.md | Custom skill covering: module structure (feature modules), entity definitions (TypeORM decorators), service layer patterns, controller patterns (decorators, response types), DTO validation (class-validator + class-transformer), error handling (built-in exception filters), testing patterns (unit tests with mocked services, integration tests with test database), migration workflow, SQLite considerations. Install into backend repo `.copilot/skills/`. |

**0D — Create Custom UI/UX Skill (Project-Specific)**

| # | Task | Details |
|---|------|---------|
| 0D.1 | Extract rules from ui-ux-pro-max | Cherry-pick the web-relevant rules from ui-ux-pro-max's Quick Reference: Accessibility (CRITICAL), Layout & Responsive (HIGH), Typography & Color (MEDIUM), Animation (MEDIUM), Forms & Feedback (MEDIUM), Navigation Patterns (HIGH). Strip React Native-specific rules. |
| 0D.2 | Create `ui-ux-guidelines` SKILL.md | Custom skill combining: extracted rules from ui-ux-pro-max (adapted for web/Angular), Tailwind CSS patterns, Angular Material/CDK component guidelines, project-specific design tokens (colors, spacing, typography), dark/light mode approach, component patterns for this app (step cards, progress bars, checklists, markdown previews), accessibility checklist. Install into frontend repo `.copilot/skills/`. |

**0E — Create Custom Testing Skill (Supplement TDD Skill)**

| # | Task | Details |
|---|------|---------|
| 0E.1 | Create `testing-guidelines` SKILL.md | Supplement the `tdd` skill with project-specific testing guidance: Jest configuration for NestJS (supertest for integration tests, in-memory SQLite for test DB), Angular TestBed patterns (standalone component testing, HttpClientTestingModule, RouterTestingModule), test file naming conventions, test data factories, what to test vs what to skip, coverage expectations. Install into both repos' `.copilot/skills/`. |

### Phase 1: Project Setup
> Foundation — both repos, tooling, Docker scaffolding

| # | Task | Details |
|---|------|---------|
| 1.1 | Create backend repo | NestJS project via `nest new`, configure TypeORM + SQLite, add `.gitignore`, README |
| 1.2 | Create frontend repo | Angular 19 via `ng new`, configure routing, add Tailwind/Material, README |
| 1.3 | Lock Node version | Add `.nvmrc` file to both repos pinning the Node LTS version (use existing nvm setup) |
| 1.4 | VS Code workspace file | `.code-workspace` file referencing both repos, include recommended extensions (SonarLint, Angular Language Service, ESLint, Prettier) |
| 1.5 | SonarQube integration | Configure SonarLint VS Code extension to connect to existing local SonarQube instance. Add `sonar-project.properties` to both repos with project keys, sources, exclusions, and coverage paths. |
| 1.6 | Docker setup | `Dockerfile` for each repo + `docker-compose.yml` in a separate infra repo or root |
| 1.7 | Database migrations | Initial migration with all 5 tables (template, template_step, instance, instance_step, todo) |
| 1.8 | API design doc | Finalize OpenAPI/Swagger spec from the existing endpoint definitions |

### Phase 2: Backend — Templates & Steps
> Core CRUD for templates and their steps

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 2.1 | Template entity + module | 1.1, 1.7 | `TemplateModule` with entity, service, controller |
| 2.2 | Template CRUD endpoints | 2.1 | `GET/POST/PUT/DELETE /api/templates` |
| 2.3 | Template Step entity + module | 2.1 | `TemplateStepModule` with gap ordering logic |
| 2.4 | Template Step CRUD endpoints | 2.3 | `POST/PUT/DELETE /api/templates/:id/steps` |
| 2.5 | Step reordering (move) | 2.4 | `PATCH .../steps/:id/move` with gap recalculation + rebalancing |
| 2.6 | Unit + integration tests | 2.2, 2.4, 2.5 | Test all template/step endpoints |

### Phase 3: Backend — Instances & Runs
> Instance creation, variable rendering, step completion

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 3.1 | Instance entity + module | 1.7 | `InstanceModule` with entity, service, controller |
| 3.2 | Variable extraction service | 3.1 | Regex-based `{{var}}` extraction + deduplication |
| 3.3 | Instance creation flow | 3.1, 3.2, 2.3 | Copy template steps → render variables → set `next_step_id` (transactional) |
| 3.4 | Instance list endpoint | 3.1 | `GET /api/instances` with next step + progress |
| 3.5 | Instance detail endpoint | 3.1 | `GET /api/instances/:id` with all steps |
| 3.6 | Step completion | 3.3 | `PATCH .../steps/:id` — toggle completion, advance `next_step_id` |
| 3.7 | Instance status lifecycle | 3.6 | Auto-set `COMPLETED` when all steps done, support `ABANDONED` |
| 3.8 | Unit + integration tests | 3.3–3.7 | Test instance creation, variable rendering, step completion |

### Phase 4: Backend — Todos
> Standalone todo CRUD *(parallel with Phase 3)*

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 4.1 | Todo entity + module | 1.7 | `TodoModule` with entity, service, controller |
| 4.2 | Todo CRUD endpoints | 4.1 | `GET/POST/PATCH/DELETE /api/todos` |
| 4.3 | Unit + integration tests | 4.2 | Test all todo endpoints |

### Phase 5: Backend — Today Dashboard
> Optimized dashboard query

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 5.1 | Dashboard endpoint | 3.4, 4.2 | `GET /api/dashboard` — returns in-progress runs (with next step) + incomplete todos |
| 5.2 | Tests | 5.1 | Test dashboard aggregation |

### Phase 6: Frontend — Shell & Routing
> App skeleton, navigation, shared components

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 6.1 | App routing | 1.2 | Routes: `/today`, `/runs`, `/templates`, `/todos` with default redirect to `/today` |
| 6.2 | Navigation component | 6.1 | Sidebar or top nav with 4 links |
| 6.3 | API service layer | 6.1 | Angular `HttpClient` services for templates, instances, todos |
| 6.4 | Shared UI components | 6.1 | Loading spinner, empty states, confirmation dialog |

### Phase 7: Frontend — Templates
> Template list, editor, step editor with markdown preview

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 7.1 | Template list page | 6.3 | List templates with step counts, create button |
| 7.2 | Template editor page | 7.1 | Edit name/description, display steps list |
| 7.3 | Step editor | 7.2 | Inline or modal editor with markdown preview (`ngx-markdown`) |
| 7.4 | Step drag-and-drop reorder | 7.2 | Angular CDK drag-drop, calls move endpoint |
| 7.5 | Template deletion | 7.1 | With confirmation dialog |

### Phase 8: Frontend — Runs
> Instance creation with dynamic variable form, execution screen

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 8.1 | Run list page | 6.3 | List instances with progress, status |
| 8.2 | "Start Run" flow | 8.1, 6.3 | Select template → extract variables → render dynamic form → create instance |
| 8.3 | Run execution screen | 8.1 | Step list with expand/collapse, progress bar, rendered markdown |
| 8.4 | Step completion UI | 8.3 | Complete/uncomplete toggle, auto-advance |
| 8.5 | Copy code block button | 8.3 | Clipboard copy for code blocks in rendered instructions |

### Phase 9: Frontend — Todos
> Todo list with inline add/complete/delete *(parallel with Phase 8)*

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 9.1 | Todo list page | 6.3 | List todos, inline add, checkbox toggle, delete |

### Phase 10: Frontend — Today Dashboard
> Guided view combining runs and todos

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 10.1 | Today dashboard page | 6.3, 8.3 | Next step per active run + incomplete todos |
| 10.2 | Inline step completion | 10.1 | Complete step from dashboard, auto-refresh |
| 10.3 | Quick-open run | 10.1 | Link to full run execution screen |

### Phase 11: Polish & Deployment
> Keyboard shortcuts, Docker, final testing

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 11.1 | Keyboard shortcuts | 8.3 | SPACE (complete), N/P (navigate), C (copy) |
| 11.2 | Docker Compose | 1.6 | Production build: nginx for frontend, Node for backend, shared SQLite volume |
| 11.3 | End-to-end smoke test | All | Manual walkthrough: create template → start run → complete → verify dashboard |
| 11.4 | README + setup docs | All | Dev setup, Docker instructions, architecture overview |

---

## Relevant Files (to create)

### Backend repo (`checklist-execution-system-api`)
- `src/template/` — Template entity, service, controller
- `src/template-step/` — TemplateStep entity, service, controller (including ordering logic)
- `src/instance/` — Instance entity, service, controller (including variable rendering)
- `src/instance-step/` — InstanceStep entity, service, controller
- `src/todo/` — Todo entity, service, controller
- `src/dashboard/` — Dashboard controller (aggregates runs + todos)
- `src/database/migrations/` — TypeORM migration files
- `sonar-project.properties` — SonarQube project config
- `.nvmrc` — Node version pin
- `Dockerfile`
- `.copilot/skills/nestjs-best-practices/SKILL.md` — Custom NestJS skill
- `.copilot/skills/testing-guidelines/SKILL.md` — Custom testing skill

### Frontend repo (`checklist-execution-system-ui`)
- `src/app/pages/today/` — Today dashboard component
- `src/app/pages/templates/` — Template list + editor components
- `src/app/pages/runs/` — Run list + execution components
- `src/app/pages/todos/` — Todo list component
- `src/app/services/` — API service layer
- `src/app/shared/` — Shared components (nav, dialogs, etc.)
- `sonar-project.properties` — SonarQube project config
- `.nvmrc` — Node version pin
- `Dockerfile`
- `.copilot/skills/angular-19-best-practices/SKILL.md` — Custom Angular 19 skill
- `.copilot/skills/ui-ux-guidelines/SKILL.md` — Custom UI/UX skill
- `.copilot/skills/testing-guidelines/SKILL.md` — Custom testing skill

---

## Verification Strategy

1. **Backend**: Each phase includes unit + integration tests (Jest + supertest). Run `npm test` to validate.
2. **Frontend**: Component tests with Angular TestBed. Run `ng test`.
3. **API contract**: Generate OpenAPI spec (NestJS Swagger plugin) and validate frontend calls against it.
4. **Code quality**: SonarQube scans via SonarLint (connected mode) for continuous feedback + on-demand full scans against local SonarQube instance.
5. **E2E smoke test**: Manual walkthrough of the full flow (Phase 11.3).
6. **Docker**: `docker compose up` and verify all functionality works in containerized environment.

---

## Decisions

- **Stack**: NestJS + Angular 19 + SQLite (TypeScript everywhere, Angular-like patterns on backend)
- **Auth**: None (single-user, local app)
- **Repos**: Separate repos for frontend/backend, shared VS Code workspace
- **Instance status**: `IN_PROGRESS`, `COMPLETED`, `ABANDONED`
- **Delete behavior**: CASCADE for child records; templates deletable even with instances (instances are self-contained)
- **Todos**: Show all incomplete on dashboard (no "today" flag for V1)
- **Step un-completion**: Supported (toggle, recalculate next_step_id)
- **Copilot skills**: One monolithic SKILL.md per framework (Angular, NestJS); can split later if too large
- **Node version**: Locked via `.nvmrc` in both repos (existing nvm setup)
- **SonarQube**: Reuse existing local Docker instance; configure SonarLint VS Code extension in connected mode
- **Devcontainers**: Deferred — not needed for solo/single-machine development

## Further Considerations

1. **UI framework choice**: Angular Material vs Tailwind CSS + headless components. Material is faster to scaffold but more opinionated. Tailwind gives full design control. Recommend **Tailwind + headless** for a cleaner, modern look.

2. **OpenAPI-first vs code-first**: Generate API spec from NestJS decorators (code-first) or define OpenAPI spec first and generate code? Recommend **code-first** with NestJS Swagger module — less overhead for a solo project.

3. **Monorepo infra**: A third repo or folder for Docker Compose + workspace file, or keep `docker-compose.yml` in the backend repo? Recommend **separate lightweight infra repo** for clarity.