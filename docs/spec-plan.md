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

### Phase 0: Project Setup
> Foundation — both repos, tooling, Docker scaffolding

| # | Task | Details |
|---|------|---------|
| 0.1 | Create backend repo | NestJS project via `nest new`, configure TypeORM + SQLite, add `.gitignore`, README |
| 0.2 | Create frontend repo | Angular 19 via `ng new`, configure routing, add Tailwind/Material, README |
| 0.3 | Lock Node version | Add `.nvmrc` file to both repos pinning the Node LTS version (use existing nvm setup) |
| 0.4 | VS Code workspace file | `.code-workspace` file referencing both repos, include recommended extensions (SonarLint, Angular Language Service, ESLint, Prettier) |
| 0.5 | SonarQube integration | Configure SonarLint VS Code extension to connect to existing local SonarQube instance. Add `sonar-project.properties` to both repos with project keys, sources, exclusions, and coverage paths. |
| 0.6 | Docker setup | `Dockerfile` for each repo + `docker-compose.yml` in a separate infra repo or root |
| 0.7 | Database schema design and migration plan | Finalize the five core tables and TypeORM migration approach. Generate the initial migration only after the first-pass entities are stable. |
| 0.8 | API design doc | Finalize OpenAPI/Swagger spec from the existing endpoint definitions |

### Phase 1: Copilot Agent Skill Preparation
> Set up AI-assisted development guardrails after the repos and workspace exist

Phase 1 should produce a deliberate, layered skill stack rather than a large collection of overlapping instructions:
- External skills provide framework defaults, modern patterns, and common troubleshooting guidance.
- Repo-local skills provide project-specific policy, constraints, and implementation standards for this app.
- Deferred and rejected skills should stay documented so the toolchain does not drift over time.

**1A — Install Recommended External Skills**

| # | Task | Details |
|---|------|---------|
| 1A.1 | Install `web-design-guidelines` | `npx skills add vercel-labs/agent-skills --skill web-design-guidelines`. Audits UI code against Vercel's Web Interface Guidelines (191K installs, security PASS). Use for reviewing Angular component templates for accessibility, layout, and UX compliance. |
| 1A.2 | Install `tdd` | `npx skills add mattpocock/skills --skill tdd`. TDD with vertical slices and behavior-focused testing (6.9K installs, all security audits PASS). Framework-agnostic — works for both Jest/NestJS and Angular TestBed tests. |
| 1A.3 | Install `angular-component` | `npx skills add https://github.com/analogjs/angular-skills --skill angular-component`. Use as the primary Angular UI implementation baseline for standalone components, signal inputs/outputs, modern control flow, OnPush patterns, and accessibility-first templates. |
| 1A.4 | Install `angular-signals` | `npx skills add https://github.com/analogjs/angular-skills --skill angular-signals`. Use as the reactive state companion for component-local and service-level signal patterns, computed state, effects, and RxJS interop. |
| 1A.5 | Install `angular-testing` | `npx skills add https://github.com/analogjs/angular-skills --skill angular-testing`. Use as the Angular-specific testing companion for standalone components, signal-based state, HttpClient testing, and modern TestBed patterns. |
| 1A.6 | Defer `angular-routing` and `angular-http` | `analogjs/angular-skills:angular-routing` and `analogjs/angular-skills:angular-http` are strong fits, but can be installed when frontend implementation begins in Phases 6-10. Their patterns should still inform the custom Angular skill research. |
| 1A.7 | Skip `angular-forms` for V1 | `analogjs/angular-skills:angular-forms` is built around Angular Signal Forms, which are experimental in Angular v21+. This project is targeting Angular 19 and should standardize on typed reactive forms instead. |
| 1A.8 | Skip `angular-best-practices` | `sajeetharan/angular-agent-kit:angular-best-practices` has some useful performance rules, but it has very low adoption, a Gen Agent Trust Hub WARN result, and overlaps with the selected AnalogJS skills plus the planned custom Angular skill. |
| 1A.9 | Skip `angular-enterprise-ui` | `josegusnay/angular-enterprise-skills:angular-enterprise-ui` is too prescriptive for this app's direction: strict SCSS/BEM, atomic design categorization, and rigid UI-layer separation conflict with the planned Tailwind + headless component approach. |
| 1A.10 | Skip `angular-material-cdk-animations` for now | `7spade/black-tortoise:angular-material-cdk-animations` is useful when a project is explicitly Material/CDK-led, but this app is not committing to Angular Material as the primary UI system. Revisit only if the frontend becomes Material-heavy. |
| 1A.11 | Install `nestjs-best-practices` | `npx skills add https://github.com/kadajett/agent-nestjs-skills --skill nestjs-best-practices`. Use as the primary NestJS architectural guardrail. Strong coverage across feature modules, dependency injection, exception handling, security, performance, testing, database/ORM, API design, and deployment practices. |
| 1A.12 | Install `nestjs-expert` | `npx skills add https://github.com/sickn33/antigravity-awesome-skills --skill nestjs-expert`. Use as the practical troubleshooting companion for NestJS dependency injection failures, circular dependencies, request lifecycle issues, TypeORM integration errors, Jest/Supertest setup, and other real-world framework pitfalls. |
| 1A.13 | Evaluate `nestjs-expert` from `jeffallan/claude-skills` (DO NOT INSTALL) | Good scaffolding-oriented guidance for controllers, services, DTOs, Swagger decorators, and basic tests, but overlaps heavily with the two selected NestJS skills and adds less unique value for this project. |
| 1A.14 | Skip `developer-kit/nestjs` | `giuseppe-trisciuoglio/developer-kit:nestjs` is Drizzle ORM-centered. Since this project standardizes on TypeORM + SQLite, installing it would bias generation toward the wrong database patterns, migration workflow, and repository structure. |
| 1A.15 | Evaluate `ui-ux-pro-max` rules (DO NOT INSTALL) | `nextlevelbuilder/ui-ux-pro-max-skill` has excellent UX rules (99 guidelines, 10 priority categories) BUT: **fails Gen Agent Trust Hub security audit** and is React Native-focused. Extract useful rules (accessibility, forms, navigation, animation) into our custom UI/UX skill instead. |
| 1A.16 | Skip `frontend-design` | `anthropics/skills/frontend-design` is too creative/avant-garde for a developer productivity tool. We need consistency and usability, not "unforgettable" aesthetics. |
| 1A.17 | Skip `webapp-testing` | `anthropics/skills/webapp-testing` is Python Playwright E2E only — not relevant for Jest unit/integration testing. |

**Phase 1A Install Commands**

Run these once the frontend and backend repos exist:

```bash
# Shared foundation skills
npx skills add vercel-labs/agent-skills --skill web-design-guidelines
npx skills add mattpocock/skills --skill tdd

# Angular skills
npx skills add https://github.com/analogjs/angular-skills --skill angular-component
npx skills add https://github.com/analogjs/angular-skills --skill angular-signals
npx skills add https://github.com/analogjs/angular-skills --skill angular-testing

# NestJS skills
npx skills add https://github.com/kadajett/agent-nestjs-skills --skill nestjs-best-practices
npx skills add https://github.com/sickn33/antigravity-awesome-skills --skill nestjs-expert
```

Optional later, when frontend implementation reaches routing and data-access work in Phases 6-10:

```bash
npx skills add https://github.com/analogjs/angular-skills --skill angular-routing
npx skills add https://github.com/analogjs/angular-skills --skill angular-http
```

**1B — Create Custom Angular 19 Skill (single monolithic SKILL.md)**

| # | Task | Details |
|---|------|---------|
| 1B.1 | Research Angular 19 best practices | Gather official Angular guidelines and compare them against the installed external Angular skills (`angular-component`, `angular-signals`, `angular-testing`) so the custom skill only captures project-specific policy, Angular 19 constraints, and any deliberate deviations from those external defaults. |
| 1B.2 | Create `angular-19-best-practices` SKILL.md | Create a repo-local Angular policy skill that references the installed external Angular skills as the default baseline and only owns project-specific guidance: frontend folder structure, Angular 19 constraints, typed reactive forms over Signal Forms, route layout for `/today`, `/runs`, `/templates`, and `/todos`, `ngx-markdown` integration, Tailwind + headless component conventions, Angular CDK drag-drop usage, API service structure, dashboard/run/todo UI patterns, and app-specific accessibility and testing expectations. Install into frontend repo `.copilot/skills/`. |

**1C — Create Custom NestJS Skill (single monolithic SKILL.md)**

| # | Task | Details |
|---|------|---------|
| 1C.1 | Research NestJS best practices | Gather official NestJS patterns and compare them against the installed external NestJS skills (`nestjs-best-practices`, `nestjs-expert`) so the custom skill focuses on project-specific architecture, data modeling, and workflow rules instead of restating generic NestJS guidance. |
| 1C.2 | Create `nestjs-best-practices` SKILL.md | Create a repo-local NestJS policy skill that references the installed external NestJS skills as the default baseline and only owns project-specific guidance: feature module boundaries for templates, template steps, instances, instance steps, todos, and dashboard; TypeORM + SQLite conventions; migration workflow; transactional instance creation; gap-based step ordering and rebalancing; `next_step_id` handling; instance status lifecycle; delete semantics; NestJS error response expectations; Swagger/code-first API conventions; and backend testing expectations for Jest, Supertest, and test-database setup. Install into backend repo `.copilot/skills/`. |

**1D — Create Custom UI/UX Skill (Project-Specific)**

| # | Task | Details |
|---|------|---------|
| 1D.1 | Extract rules from ui-ux-pro-max | Cherry-pick only the web-relevant, security-safe rules from ui-ux-pro-max's Quick Reference and reconcile them with `web-design-guidelines` so the custom UI/UX skill captures project-specific UX policy instead of duplicating generic accessibility and layout advice. Keep Accessibility (CRITICAL), Layout & Responsive (HIGH), Forms & Feedback (MEDIUM), Navigation Patterns (HIGH), Typography & Color (MEDIUM), and Animation (MEDIUM); strip React Native-specific guidance. |
| 1D.2 | Create `ui-ux-guidelines` SKILL.md | Create a repo-local UI policy skill that uses `web-design-guidelines` as the baseline and only owns project-specific frontend guidance: Tailwind + headless component conventions, design tokens, responsive shell/navigation patterns, markdown rendering and code block presentation, drag-drop affordances for step reordering, form UX for dynamic run variables and todos, progress and empty-state patterns, confirmation and destructive-action patterns, reduced-motion guidance, and an accessibility checklist tuned for a keyboard-friendly engineer productivity tool. Install into frontend repo `.copilot/skills/`. |

**1E — Create Custom Testing Skill (Supplement TDD Skill)**

| # | Task | Details |
|---|------|---------|
| 1E.1 | Create `testing-guidelines` SKILL.md | Create a repo-local testing policy skill that supplements `tdd`, `angular-testing`, and the installed NestJS skills rather than duplicating them. It should own project-specific testing guidance: test pyramid for this app, vertical-slice priorities by phase, Jest/Supertest setup for NestJS, SQLite test-database strategy, Angular standalone component and service test conventions, API contract verification, test data builders/factories, naming and file placement conventions, minimum coverage expectations, and clear rules for what to test aggressively vs. what to leave to lower-level framework coverage. Install into both repos' `.copilot/skills/`. |

**Phase 1 Custom Skill Outlines**

These outlines define what each repo-local `SKILL.md` should contain so authoring can begin without another architecture pass.

**Angular 19 Skill Outline**

- Purpose and scope: explain that the skill supplements `angular-component`, `angular-signals`, and `angular-testing` and only governs Angular 19 project policy for this app
- When to use: creating pages, shared components, API services, route configuration, typed forms, markdown rendering, and drag-drop checklist UI
- Project structure: `pages/`, `services/`, `shared/`, route ownership, and file placement expectations
- Component conventions: standalone components, signal inputs/outputs, OnPush by default, control flow syntax, and when to split container vs. presentational responsibilities
- State conventions: where to use signals, where to keep RxJS, how to expose read-only state from services, and how to model transient UI state
- Forms conventions: typed reactive forms for dynamic run variables and todo editing, validation display rules, and when not to introduce experimental Signal Forms
- Routing conventions: page route map, lazy loading expectations, route param handling, and navigation patterns for `/today`, `/runs`, `/templates`, and `/todos`
- Data-access conventions: `HttpClient` service layer boundaries, DTO typing from backend contracts, loading/error state patterns, and refresh behavior after mutations
- Markdown and checklist conventions: `ngx-markdown` usage, code block rendering, copy-button integration, and safe presentation of operational instructions
- CDK and interaction conventions: drag-drop rules for step reordering, keyboard support, focus management, and empty/loading/error state patterns
- Testing conventions: what Angular tests must prove, preferred TestBed patterns, and what should be covered by the shared testing skill instead
- Anti-patterns: overusing RxJS for local state, mixing business logic into presentational components, adopting Angular 20+ only APIs without verification, and styling drift away from project UI rules
- Definition of done: checklist for accessibility, responsiveness, state handling, route behavior, and test coverage

**NestJS Skill Outline**

- Purpose and scope: explain that the skill supplements `nestjs-best-practices` and `nestjs-expert` and only governs project-specific backend policy
- When to use: creating modules, entities, DTOs, services, controllers, migrations, and transactional workflows
- Module map: `template`, `template-step`, `instance`, `instance-step`, `todo`, and `dashboard`, plus how responsibilities are divided across controller/service/entity layers
- Data model conventions: TypeORM entity style, naming, timestamps, relations, enums, and SQLite-specific considerations
- DTO and validation conventions: request DTO boundaries, `class-validator` usage, response shaping expectations, and when to separate create/update/move DTOs
- Service-layer rules: transaction boundaries, repository usage, business-rule ownership, and where lifecycle recalculation logic must live
- Project-specific workflow rules: template step ordering, gap rebalance strategy, transactional instance creation, variable extraction/rendering, `next_step_id` updates, completion toggling, and status transitions
- Delete and lifecycle semantics: cascade rules, template deletion behavior with historical instances, and how completed/abandoned instances are represented
- API conventions: route layout, code-first Swagger expectations, standard NestJS error responses, and dashboard aggregation shape
- Testing conventions: unit vs. integration split, SQLite test-database usage, Supertest expectations, and the minimum coverage required for workflow-heavy services
- Anti-patterns: leaking ORM entities directly to clients without intent, pushing workflow logic into controllers, using ad hoc SQL when TypeORM patterns suffice, and bypassing transactions for multi-step mutations
- Definition of done: checklist for validation, transactions, error handling, migration readiness, and test coverage

**UI/UX Skill Outline**

- Purpose and scope: explain that the skill supplements `web-design-guidelines` and only governs this app's visual system and productivity-tool interaction model
- Design principles: calm, dense-but-readable productivity UI, strong information hierarchy, minimal friction, and explicit action feedback
- Visual system: color tokens, spacing scale, typography rules, surface hierarchy, icon usage, and treatment of success/warning/destructive states
- Shell and navigation: responsive app shell, primary nav behavior, active-state treatment, and layout expectations across desktop and smaller screens
- Page patterns: Today dashboard cards, run execution flow, template editor layout, and todo list behavior
- Forms and feedback: inline validation, confirmation rules, destructive actions, optimistic vs. confirmed updates, and toast/banner usage
- Markdown presentation: readable instruction blocks, code fences, copy affordances, callout styling, and overflow handling
- Checklist and drag-drop behavior: clear affordances, non-ornamental motion, reorder feedback, and accessible alternatives to pointer-only interactions
- Accessibility rules: keyboard-first operation, focus order, visible focus, contrast, reduced motion, semantic structure, and screen-reader expectations
- Motion rules: where motion is useful, where it is noise, and how `prefers-reduced-motion` must be respected
- Anti-patterns: generic marketing-style UI, over-animated interactions, cramped dense layouts, low-contrast tokens, and destructive actions without confirmation
- Definition of done: checklist for readability, responsiveness, a11y, motion restraint, and task-flow clarity

**Testing Skill Outline**

- Purpose and scope: explain that the skill supplements `tdd`, `angular-testing`, and the external NestJS skills and only governs project-specific test strategy
- Testing philosophy: vertical slices first, protect workflow logic over boilerplate, and favor stable high-signal tests over large fragile suites
- Phase-by-phase priorities: which backend and frontend behaviors must be tested in Phases 2 through 10 and which can wait until smoke testing
- Backend test strategy: service unit tests, controller/integration tests, SQLite-backed flows, Supertest coverage, and transaction-heavy workflow validation
- Frontend test strategy: standalone component tests, service tests, interaction tests for runs/templates/todos, and when to avoid over-testing pure framework behavior
- Contract and integration checks: API contract alignment, DTO assumptions, dashboard aggregation shape, and markdown/rendering expectations that matter to users
- Test data patterns: builders/factories, fixture organization, deterministic IDs/dates, and reusable sample templates/runs/todos
- Naming and placement conventions: test file naming, colocated vs. integration test placement, and folder structure for factories/utilities
- Coverage rules: baseline expectations for critical services and components, and criteria for increasing coverage in workflow-heavy areas
- Anti-patterns: brittle snapshot-heavy UI tests, full end-to-end coverage for every branch, over-mocked workflow tests, and tests that duplicate Angular or NestJS internals
- Definition of done: checklist for behavior coverage, fixture clarity, deterministic execution, and maintainability

### Phase 2: Backend — Templates & Steps
> Core CRUD for templates and their steps

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 2.1 | Template entity + module | 0.1, 0.7 | `TemplateModule` with entity, service, controller |
| 2.2 | Template CRUD endpoints | 2.1 | `GET/POST/PUT/DELETE /api/templates` |
| 2.3 | Template Step entity + module | 2.1 | `TemplateStepModule` with gap ordering logic |
| 2.4 | Template Step CRUD endpoints | 2.3 | `POST/PUT/DELETE /api/templates/:id/steps` |
| 2.5 | Step reordering (move) | 2.4 | `PATCH .../steps/:id/move` with gap recalculation + rebalancing |
| 2.6 | Unit + integration tests | 2.2, 2.4, 2.5 | Test all template/step endpoints |

### Phase 3: Backend — Instances & Runs
> Instance creation, variable rendering, step completion

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 3.1 | Instance entity + module | 0.7 | `InstanceModule` with entity, service, controller |
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
| 4.1 | Todo entity + module | 0.7 | `TodoModule` with entity, service, controller |
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
| 6.1 | App routing | 0.2 | Routes: `/today`, `/runs`, `/templates`, `/todos` with default redirect to `/today` |
| 6.2 | Navigation component | 6.1 | Sidebar or top nav with 4 links |
| 6.3 | API service layer | 6.1 | Angular `HttpClient` services for templates, instances, todos |
| 6.4 | Shared UI components | 6.1 | Loading spinner, empty states, confirmation dialog |

### Phase 7: Frontend — Templates
> Template list, editor, step editor with markdown preview

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 7.1 | Template list page | 6.3, 2.2 | List templates with step counts, create button |
| 7.2 | Template editor page | 7.1, 2.4 | Edit name/description, display steps list |
| 7.3 | Step editor | 7.2, 2.4 | Inline or modal editor with markdown preview (`ngx-markdown`) |
| 7.4 | Step drag-and-drop reorder | 7.2, 2.5 | Angular CDK drag-drop, calls move endpoint |
| 7.5 | Template deletion | 7.1 | With confirmation dialog |

### Phase 8: Frontend — Runs
> Instance creation with dynamic variable form, execution screen

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 8.1 | Run list page | 6.3, 3.4 | List instances with progress, status |
| 8.2 | "Start Run" flow | 8.1, 6.3, 2.2, 3.3 | Select template → extract variables → render dynamic form → create instance |
| 8.3 | Run execution screen | 8.1, 3.5 | Step list with expand/collapse, progress bar, rendered markdown |
| 8.4 | Step completion UI | 8.3, 3.6 | Complete/uncomplete toggle, auto-advance |
| 8.5 | Copy code block button | 8.3 | Clipboard copy for code blocks in rendered instructions |

### Phase 9: Frontend — Todos
> Todo list with inline add/complete/delete *(parallel with Phase 8)*

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 9.1 | Todo list page | 6.3, 4.2 | List todos, inline add, checkbox toggle, delete |

### Phase 10: Frontend — Today Dashboard
> Guided view combining runs and todos

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 10.1 | Today dashboard page | 6.3, 5.1, 8.3, 9.1 | Next step per active run + incomplete todos |
| 10.2 | Inline step completion | 10.1, 3.6 | Complete step from dashboard, auto-refresh |
| 10.3 | Quick-open run | 10.1 | Link to full run execution screen |

### Phase 11: Polish & Deployment
> Keyboard shortcuts, Docker, final testing

| # | Task | Depends On | Details |
|---|------|-----------|---------|
| 11.1 | Keyboard shortcuts | 8.3 | SPACE (complete), N/P (navigate), C (copy) |
| 11.2 | Docker Compose | 0.6 | Production build: nginx for frontend, Node for backend, shared SQLite volume |
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
- **Angular external skills**: Install `analogjs/angular-skills:angular-component`, `analogjs/angular-skills:angular-signals`, and `analogjs/angular-skills:angular-testing` as the initial Angular skill set; defer `angular-routing` and `angular-http` until frontend implementation begins; do not install `angular-forms`, `angular-best-practices`, `angular-enterprise-ui`, or `angular-material-cdk-animations` for V1
- **NestJS external skills**: Install `kadajett/agent-nestjs-skills:nestjs-best-practices` as the primary architectural baseline and `sickn33/antigravity-awesome-skills:nestjs-expert` as the troubleshooting/debugging companion; do not install the `jeffallan/claude-skills:nestjs-expert` or `giuseppe-trisciuoglio/developer-kit:nestjs` skills for V1
- **UI/UX skill strategy**: Use `web-design-guidelines` as the external baseline for general accessibility, layout, and interaction quality, then keep the custom `ui-ux-guidelines` skill focused on Tailwind + headless patterns, markdown/checklist/runbook-specific UI behavior, and keyboard-first productivity workflows
- **Testing skill strategy**: Use `tdd` as the cross-stack workflow baseline, `angular-testing` for Angular-specific test implementation patterns, and the custom `testing-guidelines` skill for project-specific test scope, fixtures, SQLite test-database setup, and phase-by-phase coverage priorities
- **Phase ordering**: Project setup/bootstrap comes before Copilot skill authoring so repo-local skills and workspace guidance target the real project structure
- **Node version**: Locked via `.nvmrc` in both repos (existing nvm setup)
- **SonarQube**: Reuse existing local Docker instance; configure SonarLint VS Code extension in connected mode
- **Devcontainers**: Deferred — not needed for solo/single-machine development

## Further Considerations

1. **UI framework choice**: Angular Material vs Tailwind CSS + headless components. Material is faster to scaffold but more opinionated. Tailwind gives full design control. Recommend **Tailwind + headless** for a cleaner, modern look.

2. **OpenAPI-first vs code-first**: Generate API spec from NestJS decorators (code-first) or define OpenAPI spec first and generate code? Recommend **code-first** with NestJS Swagger module — less overhead for a solo project.

3. **Monorepo infra**: A third repo or folder for Docker Compose + workspace file, or keep `docker-compose.yml` in the backend repo? Recommend **separate lightweight infra repo** for clarity.