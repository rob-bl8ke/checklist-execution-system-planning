## Plan: Recurring Reminders & Sprint Prep

Introduce a first-class reminder/schedule feature instead of extending todos. A reminder definition stores cadence, lead time, optional template link, and category; a separate per-occurrence state record handles completion or dismissal for each scheduled occurrence. This fits daily reminders, end-of-week prep reminders, and sprint ceremonies without forcing recurring behavior into one-off todos or auto-generating runs too early.

**Tightened Data Model**
- `reminder_definition`
  Fields:
  `id` INTEGER PK, `title` TEXT NOT NULL, `description` TEXT NULL, `category` TEXT NULL (free-text, max 50 chars), `cadence` TEXT NOT NULL (`ONCE` | `DAILY` | `WEEKLY`), `interval` INTEGER NOT NULL DEFAULT 1 (min 1, max 52), `anchor_date` TEXT NOT NULL (stored as `YYYY-MM-DD` text in SQLite, typed as `string` in entity, validated as `@IsDateString()` at DTO level — local scheduling date, not UTC timestamp), `weekdays` TEXT NULL (JSON array of weekday numbers 0-6; use TypeORM JSON transformer matching `instance.variables` pattern), `time_of_day` TEXT NULL (`HH:mm`), `lead_time_days` INTEGER NOT NULL DEFAULT 0 (min 0, max 365), `linked_template_id` INTEGER NULL FK -> `template.id`, `active` BOOLEAN NOT NULL DEFAULT TRUE, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, `updated_at` DATETIME NULL.
  Rules:
  use `TEXT` for schedule anchors and occurrences because SQLite has no native `DATE` type; the core requirement is day-based advance notice; keep `time_of_day` optional and separate. `ONCE` ignores `weekdays` and uses `anchor_date` as the occurrence date. `DAILY` ignores `weekdays`. `WEEKLY` requires at least one weekday and uses `interval` for weekly vs biweekly cadence (`1` = weekly, `2` = every other week). `lead_time_days` must be >= 0. `linked_template_id` is optional and there is no `linked_todo_id` in v1 because todos remain one-off tasks.
- `reminder_occurrence_state`
  Fields:
  `id` INTEGER PK, `reminder_id` INTEGER NOT NULL FK -> `reminder_definition.id` ON DELETE CASCADE, `occurrence_date` TEXT NOT NULL (stored as `YYYY-MM-DD` text, typed as `string` in entity), `status` TEXT NOT NULL (`COMPLETED` | `DISMISSED`), `acted_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP.
  Rules:
  add a unique constraint on (`reminder_id`, `occurrence_date`). Absence of a row means the occurrence is still open. This keeps the database small and avoids persisting generated occurrences that were never acted on. `ON DELETE CASCADE` removes occurrence state rows when a reminder definition is deleted, matching the cascade pattern for `template_step` and `instance_step`.
- Indexes
  Add an index on `reminder_definition.active`, an index on `reminder_definition.linked_template_id`, and a composite index on `reminder_occurrence_state(reminder_id, occurrence_date)`.

- Controller routing note
  Declare static routes such as `/reminders/agenda` before parameter routes such as `/reminders/:id` in the Nest controller so `agenda` is not consumed as an `id` path segment.

**Tightened Endpoint Shapes**
1. `GET /api/reminders`
   Purpose: list reminder definitions for the management screen.
   Query params: optional `active=true|false`, optional `linkedTemplateId`, optional `category`.
   Response shape: reminder definitions plus derived summary fields `nextOccurrenceDate` and `nextPrepStartDate`.
2. `GET /api/reminders/{id}`
   Purpose: fetch one reminder definition for editing.
3. `POST /api/reminders`
   Purpose: create a reminder definition.
   Body shape: `title`, `description?`, `category?`, `cadence`, `interval?`, `anchorDate`, `weekdays?`, `timeOfDay?`, `leadTimeDays?`, `linkedTemplateId?`.
4. `PUT /api/reminders/{id}`
   Purpose: full update from the reminder editor form.
   Body shape: same as create plus optional `active`.
5. `DELETE /api/reminders/{id}`
   Purpose: remove a reminder definition and its occurrence state rows.
6. `GET /api/reminders/agenda?from=YYYY-MM-DD&to=YYYY-MM-DD`
   Purpose: return computed reminder occurrences for a date window.
   Response shape: occurrence projections containing `reminderId`, `title`, `description`, `category`, `occurrenceDate`, `prepStartDate`, `timeOfDay`, `status` (`OPEN` | `COMPLETED` | `DISMISSED`), `isInPrepWindow`, `isOverdue`, `daysUntilOccurrence`, `linkedTemplate` summary, and `canStartRun`. Derived fields (`isInPrepWindow`, `isOverdue`, `daysUntilOccurrence`) are computed on the backend based on the server's current date at response time.
7. `PATCH /api/reminders/{id}/occurrences/{occurrenceDate}`
   Purpose: record user action for one occurrence.
   Body shape: `status` where `status` is `COMPLETED`, `DISMISSED`, or `OPEN`.
   Semantics: `COMPLETED` or `DISMISSED` upserts an occurrence-state row; `OPEN` removes any existing occurrence-state row so the occurrence becomes visible again.
8. `GET /api/dashboard?upcomingDays=7`
   Purpose: preserve the Today endpoint while extending it for reminders.
   Response shape: existing `runs` and `todos`, plus `reminders: { dueNow: ReminderAgendaItem[], upcoming: ReminderAgendaItem[] }`.
   Semantics: `dueNow` means occurrences where `today >= prepStartDate` (prep window has started) and still open. `upcoming` means occurrences where `today < prepStartDate` and `occurrenceDate <= today + upcomingDays`. `prepStartDate` is always `occurrenceDate - leadTimeDays`. `upcomingDays` is optional (default 7, min 1, max 30) and controls only the reminder upcoming window; runs and todos remain unfiltered. When `upcomingDays` is `0`, the `upcoming` array is empty.

**Recommended DTO Shapes**
- `CreateReminderDto`
  `title: string`, `description?: string`, `category?: string`, `cadence: 'ONCE' | 'DAILY' | 'WEEKLY'`, `interval?: number`, `anchorDate: string`, `weekdays?: number[]`, `timeOfDay?: string`, `leadTimeDays?: number`, `linkedTemplateId?: number`
- `UpdateReminderDto`
  Same fields as create, plus `active?: boolean`
- `UpdateReminderOccurrenceDto`
  `status: 'COMPLETED' | 'DISMISSED' | 'OPEN'`

**Contract Decisions**
- Use explicit cadence fields instead of RRULE in v1. The supported cases are simple and explicit columns will be easier to validate, query, test, and evolve in this codebase.
- Keep reminder definitions separate from occurrence state. A recurring reminder is durable; user actions apply to one occurrence only.
- Do not persist generated future occurrences in v1. Compute them on demand from the definition and overlay occurrence-state rows.
- Use `PUT` for reminder definition updates because the editor will naturally submit the full form, and keep `PATCH` for occurrence actions because those are partial state transitions.
- Keep dashboard reminder payload occurrence-based rather than definition-based, because the Today screen needs actionable items, not raw schedule rules.


**Example Request And Response Shapes**
1. `POST /api/reminders`
   Example request:
   `title: 'Prepare weekly update', description: 'Collect notable changes before Friday wrap-up', category: 'WEEKLY_PREP', cadence: 'WEEKLY', interval: 1, anchorDate: '2026-03-31', weekdays: [5], leadTimeDays: 2`
   Example response:
   `id: 12, title: 'Prepare weekly update', description: 'Collect notable changes before Friday wrap-up', category: 'WEEKLY_PREP', cadence: 'WEEKLY', interval: 1, anchorDate: '2026-03-31', weekdays: [5], timeOfDay: null, leadTimeDays: 2, linkedTemplateId: null, active: true, nextOccurrenceDate: '2026-04-03', nextPrepStartDate: '2026-04-01', createdAt: '2026-03-31T18:00:00.000Z', updatedAt: null`
2. `POST /api/reminders`
   Sprint ceremony example request:
   `title: 'Sprint retrospective', category: 'SPRINT_RETRO', cadence: 'WEEKLY', interval: 2, anchorDate: '2026-04-03', weekdays: [5], leadTimeDays: 2, linkedTemplateId: 7`
   Example response:
   `id: 13, title: 'Sprint retrospective', description: null, category: 'SPRINT_RETRO', cadence: 'WEEKLY', interval: 2, anchorDate: '2026-04-03', weekdays: [5], timeOfDay: null, leadTimeDays: 2, linkedTemplateId: 7, active: true, nextOccurrenceDate: '2026-04-03', nextPrepStartDate: '2026-04-01', createdAt: '2026-03-31T18:05:00.000Z', updatedAt: null`
3. `GET /api/reminders`
   Example response item:
   `id: 13, title: 'Sprint retrospective', category: 'SPRINT_RETRO', cadence: 'WEEKLY', interval: 2, anchorDate: '2026-04-03', weekdays: [5], timeOfDay: null, leadTimeDays: 2, linkedTemplateId: 7, active: true, nextOccurrenceDate: '2026-04-03', nextPrepStartDate: '2026-04-01', lastCompletedOccurrenceDate: null`
4. `GET /api/reminders/agenda?from=2026-03-31&to=2026-04-10`
   Example response item:
   `reminderId: 13, title: 'Sprint retrospective', category: 'SPRINT_RETRO', occurrenceDate: '2026-04-03', prepStartDate: '2026-04-01', timeOfDay: null, status: 'OPEN', isInPrepWindow: false, isOverdue: false, daysUntilOccurrence: 3, linkedTemplate: { id: 7, name: 'Sprint Retro Template' }, canStartRun: true`
5. `PATCH /api/reminders/13/occurrences/2026-04-03`
   Example request:
   `status: 'COMPLETED'`
   Example response:
   `reminderId: 13, occurrenceDate: '2026-04-03', status: 'COMPLETED', actedAt: '2026-04-03T16:20:00.000Z'`
6. `GET /api/dashboard?upcomingDays=7`
   Example reminder payload:
   `reminders: { dueNow: [{ reminderId: 14, title: 'Daily standup prep', category: 'DAILY_PREP', occurrenceDate: '2026-03-31', prepStartDate: '2026-03-31', status: 'OPEN', linkedTemplate: null, canStartRun: false }], upcoming: [{ reminderId: 13, title: 'Sprint retrospective', category: 'SPRINT_RETRO', occurrenceDate: '2026-04-03', prepStartDate: '2026-04-01', status: 'OPEN', linkedTemplate: { id: 7, name: 'Sprint Retro Template' }, canStartRun: true }] }`

**Validation Rules And Edge Cases**
1. Common validation
   `title` required, trimmed, max 200 chars. `description` optional, max 4000 chars. `category` optional, free-text, max 50 chars; use distinct-value queries for filter dropdowns rather than a constrained enum. `leadTimeDays` integer >= 0 and <= 365. `interval` integer >= 1 and <= 52 (weekly * 52 = yearly practical limit). `timeOfDay`, when present, must be `HH:mm` 24-hour format.
2. `anchorDate`
   Required for every reminder. Stored as `TEXT` in SQLite (`YYYY-MM-DD` format), typed as `string` in the entity, validated as `@IsDateString()` at the DTO level. It is a local scheduling date rather than UTC timestamp. For weekly reminders it is the cadence anchor used to determine interval boundaries, not necessarily the only weekday that can occur.
3. `cadence = ONCE`
   `interval` must be `1`. `weekdays` must be absent or empty. The only valid occurrence is `anchorDate`. If the occurrence is in the past and open, the agenda should still return it when the requested window includes that past date; the dashboard may choose to omit stale past reminders after a retention threshold, but v1 can simply keep them visible if they are in the prep window and not acted on.
4. `cadence = DAILY`
   `interval` must be >= 1. `weekdays` must be absent or empty. Occurrences repeat every `interval` days starting from `anchorDate`. If `anchorDate` is in the future, nothing appears before the derived prep window of the first occurrence.
5. `cadence = WEEKLY`
   `interval` must be >= 1. `weekdays` is required, must contain unique values in the range `0-6`, and should be normalized into ascending order. Weekly means every `interval` weeks on the given weekdays, anchored from the week containing `anchorDate`. Biweekly is represented as `interval = 2`.
6. Template links
   If `linkedTemplateId` is provided, the API must verify that the template exists. `DELETE /api/templates/{id}` returns `409 Conflict` if any active reminder definitions reference the template via `linkedTemplateId`. The error message should identify the blocking reminders. This prevents silent reminder degradation.
7. Occurrence action validation
   `occurrenceDate` path param must be a valid `YYYY-MM-DD` date and must align with a real computed occurrence for that reminder. The handler must compute the occurrence schedule for the given reminder and verify the date is a valid occurrence. Reject actions against non-occurrence dates with `400`. Repeating the same `COMPLETED` or `DISMISSED` action should be idempotent.
8. Agenda window validation
   `from` and `to` are required for `/reminders/agenda`. Reject `from > to`. Cap the allowed window to 90 days so the recurrence engine cannot be asked for arbitrarily large ranges. The recurrence engine must short-circuit once it exceeds the window end date rather than precomputing all possible occurrences then filtering. For `/dashboard`, default `upcomingDays = 7`, minimum `1`, maximum `30`.
9. Duplicate semantics
   Two reminder definitions may share the same title and schedule; do not enforce cross-row uniqueness. The uniqueness boundary is occurrence state per reminder definition.
10. Time and timezone edge case
   Because v1 is day-oriented, agenda visibility should be based on local calendar dates, not UTC instants. If you later add external notifications, introduce an explicit timezone on reminder definitions before switching logic to timestamps.
11. Past-due behavior
   If an occurrence is open and `today > occurrenceDate`, return it as `OPEN` and mark it overdue in derived fields rather than silently dropping it. This avoids hidden missed reminders.
12. State restoration
   `PATCH ... status = OPEN` should delete the occurrence-state row, not write a third persisted state. That keeps storage minimal and preserves `OPEN` as the default derived state.

**Frontend Payload Shapes**
1. Today dashboard reminder item
   `ReminderAgendaItem`: `reminderId`, `title`, `description`, `category`, `occurrenceDate`, `prepStartDate`, `timeOfDay`, `status`, `isInPrepWindow`, `isOverdue`, `daysUntilOccurrence`, `linkedTemplate`, `canStartRun`
   Derived fields (`isInPrepWindow`, `isOverdue`, `daysUntilOccurrence`) are computed on the backend based on the server's current date at response time.
   Use this in [checklist-execution-system-ui/src/app/pages/today/today.component.ts](checklist-execution-system-ui/src/app/pages/today/today.component.ts) so the UI does not need to recompute date semantics.
2. Today dashboard response
   Keep existing `runs` and `todos` and add:
   `reminders: { dueNow: ReminderAgendaItem[], upcoming: ReminderAgendaItem[] }`
   `dueNow` should already exclude completed or dismissed occurrences. `upcoming` should include only future occurrences where `today < prepStartDate` and `occurrenceDate <= today + upcomingDays`. When `upcomingDays` is `0`, the `upcoming` array is empty.
3. Reminder management list item
   `ReminderListItem`: `id`, `title`, `description`, `category`, `cadence`, `interval`, `anchorDate`, `weekdays`, `timeOfDay`, `leadTimeDays`, `linkedTemplateId`, `linkedTemplateName`, `active`, `nextOccurrenceDate`, `nextPrepStartDate`, `lastCompletedOccurrenceDate`
   This supports the top-level Reminders page table or card list without an extra per-row lookup.
4. Reminder detail payload
   The edit screen can reuse the full reminder definition response from `GET /api/reminders/{id}` with no separate detail-only contract. Include `linkedTemplateId` and, if useful for display, a lightweight `linkedTemplateName`.
5. Reminder page route and nav integration
   Add a new top-level route beside Today, Runs, Templates, and Todos in [checklist-execution-system-ui/src/app/app.routes.ts](checklist-execution-system-ui/src/app/app.routes.ts). Add a matching nav entry in [checklist-execution-system-ui/src/app/components/nav/nav.component.ts](checklist-execution-system-ui/src/app/components/nav/nav.component.ts). Keep reminders separate from Todos in the information architecture.
6. List/detail loading model
   The Reminders page should load reminder definitions for management. The Today page should load reminder occurrences via the dashboard payload. Do not force the management page to reconstruct occurrences from raw cadence fields unless the user is explicitly previewing a schedule window.

**Exact Nest Shapes**
1. Entity design
   Create one entity for reminder definitions and one entity for occurrence state. Follow the same TypeORM style as [checklist-execution-system-api/src/template/template.entity.ts](checklist-execution-system-api/src/template/template.entity.ts) and [checklist-execution-system-api/src/todo/todo.entity.ts](checklist-execution-system-api/src/todo/todo.entity.ts): `@Entity`, `@PrimaryGeneratedColumn`, snake_case column names where needed, and `@CreateDateColumn` or `@UpdateDateColumn` for audit fields.
2. Reminder definition entity fields
   `id: number`
   `title: string`
   `description: string | null`
   `category: string | null`
   `cadence: ReminderCadence`
   `interval: number`
   `anchorDate: string` (stored as TEXT `YYYY-MM-DD` in SQLite, not a Date object)
   `weekdays: string | null` as stored JSON text in SQLite; use a TypeORM JSON transformer (serialize array to/from TEXT) matching the `instance.variables` pattern
   `timeOfDay: string | null`
   `leadTimeDays: number`
   `linkedTemplateId: number | null`
   `active: boolean`
   `createdAt: Date`
   `updatedAt: Date | null`
   Include a `ManyToOne` to `Template` for the optional template link if the codebase already favors relation loading there; otherwise keep `linkedTemplateId` as a plain column and resolve template summaries in the service layer for simpler queries.
3. Reminder occurrence state entity fields
   `id: number`
   `reminderId: number`
   `occurrenceDate: string` (stored as TEXT `YYYY-MM-DD` in SQLite)
   `status: ReminderOccurrenceStatus`
   `actedAt: Date`
   Include a `ManyToOne` back to the reminder definition.
4. Enums
   `ReminderCadence` should be `ONCE`, `DAILY`, `WEEKLY`.
   `ReminderOccurrenceStatus` should be `COMPLETED`, `DISMISSED`.
   Keep `OPEN` derived only and do not persist it.
5. DTO classes
   `CreateReminderDto`
   Fields: `title`, `description?`, `category?`, `cadence`, `interval?`, `anchorDate`, `weekdays?`, `timeOfDay?`, `leadTimeDays?`, `linkedTemplateId?`.
   Validation: `@IsString`, `@IsNotEmpty`, `@MaxLength`, `@IsEnum`, `@IsInt`, `@Min`, `@Max`, `@IsDateString`, `@IsOptional`, `@IsArray`, `@ArrayUnique` as appropriate.
   `UpdateReminderDto`
   Use `PartialType(CreateReminderDto)` if Swagger tooling and current Nest setup support it cleanly; otherwise mirror the existing explicit optional-field style used in the API DTOs.
   Add `active?: boolean`.
   `UpdateReminderOccurrenceDto`
   Field: `status` validated against `COMPLETED`, `DISMISSED`, `OPEN`.
6. Controller shape
   Add a dedicated reminders controller using the same conventions as [checklist-execution-system-api/src/todo/todos.controller.ts](checklist-execution-system-api/src/todo/todos.controller.ts) and [checklist-execution-system-api/src/instance/instances.controller.ts](checklist-execution-system-api/src/instance/instances.controller.ts): `GET`, `POST`, `PUT`, `DELETE`, and `PATCH` with `ParseIntPipe` for ids. Declare `GET agenda` before `GET :id`.
7. Service split
   Keep CRUD logic for reminder definitions in one service. Implement recurrence projection logic in a dedicated pure utility module (`src/reminder/recurrence.utils.ts`) as exported functions with no `@Injectable()` decorator and no database access. This makes recurrence computation independently unit-testable without TestBed or mocks. The service calls these utils and overlays occurrence state from the database. The dashboard service should depend on the reminder service for occurrence projections rather than duplicating recurrence logic.
8. API response models
   Return plain JSON objects rather than exposing entity internals directly where derived fields are involved. Definition list responses should add `nextOccurrenceDate`, `nextPrepStartDate`, `lastCompletedOccurrenceDate`, `createdAt`, and `updatedAt`. The `lastCompletedOccurrenceDate` field should be computed with a single query using `MAX(occurrence_date) WHERE status = 'COMPLETED' GROUP BY reminder_id` to avoid N+1 lookups. Agenda responses should return occurrence projection items with derived booleans already computed on the backend based on the server's current date at response time.

**Reminder UX And Form Behavior**
1. Top-level page purpose
   The new Reminders page is for managing recurring definitions. Today remains the execution surface that shows actionable occurrences.
2. Page layout
   Use a two-part layout on the Reminders page: a compact filter or summary row at the top and a list of reminder cards or rows below. Each item should show title, cadence summary, lead time, next occurrence, optional linked template, and active/inactive status.
3. Primary actions
   The list page should support `Create reminder`, `Edit`, `Deactivate/Activate`, and `Delete`. Do not put occurrence-level completion actions on the management page; those belong on Today or an agenda preview.
4. Creation flow
   Clicking `Create reminder` opens a form page or modal with ceremony presets at the top. Presets should prefill title, category, cadence, interval, default weekdays, and suggested lead time, but still allow editing before save.
5. Form sections
   Section 1: Basics with title, description, and category.
   Section 2: Schedule with cadence selector, interval, anchor date, weekday picker when cadence is weekly, and optional time of day.
   Section 3: Visibility with lead time days and a plain-language summary such as `Shown 2 days before occurrence`.
   Section 4: Runbook link with optional template picker.
   Section 5: Preview with the next few computed occurrences so users can verify the schedule before saving.
6. Conditional form behavior
   If cadence is `ONCE`, hide interval and weekday controls.
   If cadence is `DAILY`, show interval and hide weekdays.
   If cadence is `WEEKLY`, show interval and weekday multi-select.
   If a template is linked, show helper text that Today will offer a `Start run` action rather than auto-creating a run.
7. Validation UX
   Validate immediately for impossible combinations such as weekly cadence with no weekdays, negative lead time, or invalid time format. Show schedule-specific errors adjacent to the relevant control instead of a single generic form error.
8. Preview behavior
   The reminder editor should preview the next 3 to 5 occurrences locally from the current form state or via a lightweight preview endpoint. Recommended v1 approach: local preview in the UI only if it can reuse the same pure recurrence logic semantics cleanly; otherwise add a non-persistent server-side preview endpoint later. Do not block v1 on preview if it complicates implementation.
9. Today page behavior
   Add a reminders section before active runs or directly above todos in [checklist-execution-system-ui/src/app/pages/today/today.component.ts](checklist-execution-system-ui/src/app/pages/today/today.component.ts). Split it into `Ready Now` and `Coming Up` sections. Each item should show title, due date, lead-time framing like `in 2 days`, and actions: `Done`, `Dismiss`, and `Start run` when linked to a template.
10. Navigation behavior
   The nav should gain a Reminders entry. If notification-style emphasis is added later, badge counts should attach to Today rather than the Reminders page, because Today is the action surface.

**Spec Delta**
1. System overview
   Expand the product scope from three functional areas to four: Runbooks, Runs, Todos, and Reminders.
2. Core concepts
   Add `Reminder Definition` and `Reminder Occurrence` sections. Define reminder definitions as recurring schedules with optional template links and occurrence state as per-date completion or dismissal.
3. Database design
   Add the two reminder tables and note that occurrence rows are only persisted when acted on.
4. API specification
   Add a new `Reminders` section with the reminder CRUD, agenda endpoint, and occurrence action endpoint. Update the `Dashboard` response contract to include reminders.
5. UX design
   Add a Reminders page and update the Today dashboard section to show reminder occurrences in prep windows and upcoming occurrences.
6. Final system summary
   Increase the count of functional areas and endpoint inventory accordingly. Update to 9 database tables, ~32 REST endpoints, and 6 UI pages. Note that reminders are app-delivered in v1 and that linked templates surface a `Start run` action rather than scheduled instance creation.
7. Future enhancements
   Move external notifications, timezone-aware delivery, calendar sync, and a possible Sprint aggregate into future enhancements instead of leaving them implicit.

8. Current spec rewrite scope
   The spec update should directly replace Section 1, extend Section 2 with reminder concepts, add Section 4.6 and 4.7 for reminder tables, add Section 6.6 for reminders, update Section 7.1 and 7.2 plus add a dedicated reminders screen section, update Section 12 counts and summary language, and expand Section 13 future enhancements with external notifications, timezone support, calendar sync, and an optional Sprint aggregate.


**Steps**
1. Confirm v1 scope and semantics: recurring reminder definitions, advance notice in days, manual sprint schedule setup, and optional template links are included. External notifications, calendar sync, auto-created runs, and a dedicated Sprint aggregate are excluded from the first implementation. This blocks schema and API design.
2. Add a reminder domain to the API. Create a new module/entity/controller/service for reminder definitions and a second table for per-occurrence state. Reminder fields should cover title, description, category, recurrence type, anchor date, optional weekday config, optional time, lead time days, optional templateId, and active flag. Occurrence state should capture reminderId, occurrenceDate, status, completedAt, and dismissedAt. Depends on step 1.
3. Implement recurrence calculation as pure utility functions in `src/reminder/recurrence.utils.ts` (no `@Injectable()` decorator, no database access). It should compute occurrences for a requested date window and determine visibility using occurrenceDate minus leadTimeDays. Support once, daily, weekly, biweekly, and weekly day-of-week variants. Keep the logic deterministic and unit-testable without TestBed or mocks. Depends on step 2.
4. Extend dashboard and reminder APIs. Update the dashboard payload to return active reminders that are already in their prep window and upcoming reminders for the next 7-14 days. Add reminder CRUD plus occurrence actions such as complete or dismiss for a specific occurrence date. Do not auto-create instances; instead include linked template metadata so the UI can offer Start run when appropriate. Depends on steps 2 and 3.
5. Add migrations and application wiring. Register the new entities and module in AppModule, add database migrations, and index reminder activity queries and occurrence state lookups. Preserve backward compatibility for existing templates, runs, and todos. Parallel with step 3 after entity shapes stabilize; blocks verification.
6. Add frontend reminder models and API services. Extend api models, augment dashboard fetching, and add a dedicated reminders API service. Keep recurring reminder types separate from Todo so recurring occurrence semantics stay isolated from one-off task semantics. Depends on step 4.
7. Add Today dashboard reminder UX. Add reminder sections above or alongside active runs and todos, split between items already in their prep window and upcoming items. Provide actions to mark the current occurrence done and to open or start a linked runbook when present. Depends on step 6.
8. Add reminder management UI as a new top-level page. Do not overload the existing Todos screen. The reminder editor should cover cadence, lead time days, anchor date, weekday/time, category, and optional template link. Parallel with step 7 after step 6.
9. Add sprint ceremony presets on top of the reminder model. Provide presets for refinement, retro, and planning that prefill cadence and naming, but store them as normal reminders. This keeps v1 simple while leaving room for a future Sprint aggregate if you later want sprint start/end metadata and generated ceremonies. Depends on step 8.
10. Verify end to end. Add backend unit tests for recurrence and prep-window visibility, dashboard tests for occurrence state behavior, and UI/manual checks for daily reminders, weekly reminders with lead time, and biweekly sprint ceremonies linked to templates. Depends on steps 4, 7, and 8.

**Relevant files**
- c:\Code\rob-bl8ke\checklist-execution-system-api\src\app.module.ts — register the new reminder entities/module alongside Template, Instance, and Todo modules.
- c:\Code\rob-bl8ke\checklist-execution-system-api\src\dashboard\dashboard.service.ts — extend getToday() so the dashboard returns active and upcoming reminders in addition to runs and todos.
- c:\Code\rob-bl8ke\checklist-execution-system-api\src\dashboard\dashboard.controller.ts — add `@Query('upcomingDays')` parameter with `DefaultValuePipe` and validation; keep the dashboard contract aligned with the expanded payload.
- c:\Code\rob-bl8ke\checklist-execution-system-api\src\template\templates.service.ts — update `remove()` to check for linked reminder definitions and throw `ConflictException` if any active reminders reference the template.
- c:\Code\rob-bl8ke\checklist-execution-system-api\src\todo\todo.entity.ts — preserve as the one-off task model; use as a boundary reference rather than extending it with recurring occurrence semantics.
- c:\Code\rob-bl8ke\checklist-execution-system-api\src\reminder\reminder-definition.entity.ts — reminder definition entity with schedule columns, weekdays JSON transformer, anchorDate as string.
- c:\Code\rob-bl8ke\checklist-execution-system-api\src\reminder\reminder-occurrence-state.entity.ts — per-occurrence state entity with CASCADE delete on reminder FK.
- c:\Code\rob-bl8ke\checklist-execution-system-api\src\reminder\reminders.controller.ts — reminder CRUD, agenda, and occurrence state endpoints; declare `GET agenda` before `GET :id`.
- c:\Code\rob-bl8ke\checklist-execution-system-api\src\reminder\reminders.service.ts — reminder CRUD, occurrence state, derived field computation; uses recurrence utils.
- c:\Code\rob-bl8ke\checklist-execution-system-api\src\reminder\recurrence.utils.ts — pure functions for computing occurrence dates (no DI); independently unit-testable.
- c:\Code\rob-bl8ke\checklist-execution-system-api\src\reminder\dto\ — CreateReminderDto, UpdateReminderDto, UpdateReminderOccurrenceDto.
- c:\Code\rob-bl8ke\checklist-execution-system-api\src\reminder\enums\ — ReminderCadence, ReminderOccurrenceStatus enums.
- c:\Code\rob-bl8ke\checklist-execution-system-ui\src\app\models\api.models.ts — add reminder definition types, occurrence state, and expanded dashboard response models.
- c:\Code\rob-bl8ke\checklist-execution-system-ui\src\app\services\reminders-api.service.ts — HTTP client for reminder CRUD, agenda, and occurrence actions.
- c:\Code\rob-bl8ke\checklist-execution-system-ui\src\app\pages\today\today.component.ts — add reminder sections and actions to the dashboard entrypoint.
- c:\Code\rob-bl8ke\checklist-execution-system-ui\src\app\pages\reminders\ — reminder management list and editor flows.
- c:\Code\rob-bl8ke\checklist-execution-system-ui\src\app\pages\todos\todo-list\todo-list.component.ts — reference as the current one-off todo interaction pattern; do not turn it into the recurring reminder editor.
- c:\Code\rob-bl8ke\checklist-execution-system-planning\docs\current-spec.md — update concepts, API, and UX sections once the approach is approved.

**Verification**
1. Add backend recurrence tests proving leadTimeDays makes reminders appear before the scheduled occurrence for once, daily, weekly, and biweekly schedules.
2. Add backend dashboard tests proving completion or dismissal hides only the current occurrence and not the entire recurring reminder definition.
3. Manually create a daily reminder and confirm it appears on Today each day without generating duplicate todos.
4. Manually create a weekly reminder with leadTimeDays set to 2 and confirm it becomes visible two days early and remains visible until completed or the occurrence passes.
5. Manually create a sprint retro preset linked to a template and confirm Today shows it early and allows starting a run without auto-generating one.
6. Run regression checks to confirm existing Today runs and todos behave exactly as before when no reminders exist.

**Decisions**
- Use a first-class reminder domain instead of extending Todo because recurring items need schedule metadata and per-occurrence state, while todos are currently one-off tasks.
- Deliver inside the app first, but shape API/service boundaries so email, Slack, or webhook delivery can be added later without changing the reminder model.
- Keep schedule definition manual in v1, including sprint ceremonies as presets, instead of introducing calendar sync.
- Do not auto-create instances in v1 because many runbook runs need human timing or variable input; linked templates should produce an explicit Start run action instead.
- Compute reminder visibility on demand and persist only per-occurrence state, avoiding a background scheduler while delivery remains dashboard-based.

**Further Considerations**
1. If you later want sprint start/end tracking, add a Sprint aggregate that generates reminder definitions for planning, refinement, and retro rather than replacing the reminder engine.
2. If exact ceremony times become important, add optional time-of-day and timezone settings before introducing external notifications.
