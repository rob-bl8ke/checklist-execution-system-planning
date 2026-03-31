## Plan: Reminders — Phased Task List

**72 tasks across 12 phases.** Backend first (Phases 1–8), then frontend (Phases 9–12). Each phase is independently verifiable before moving to the next.

---

### Phase 1: Backend — Database & Module Scaffold

1. Create `src/reminder/enums/reminder-cadence.enum.ts` exporting `ReminderCadence` with values `ONCE`, `DAILY`, `WEEKLY`
2. Create `src/reminder/enums/reminder-occurrence-status.enum.ts` exporting `ReminderOccurrenceStatus` with values `COMPLETED`, `DISMISSED`
3. Create `src/reminder/reminder-definition.entity.ts` — `@Entity('reminder_definition')` with: `id` (`@PrimaryGeneratedColumn`), `title` (text), `description` (text, nullable), `category` (text, nullable), `cadence` (`ReminderCadence`, text), `interval` (integer, default 1), `anchorDate` (text, `anchor_date`), `weekdays` (text nullable, JSON array transformer matching `instance.variables` pattern), `timeOfDay` (text nullable, `time_of_day`), `leadTimeDays` (integer, default 0, `lead_time_days`), `linkedTemplateId` (integer nullable, `linked_template_id`), `active` (boolean, default true), `createdAt` (`@CreateDateColumn`), `updatedAt` (`@UpdateDateColumn` nullable); `@OneToMany` → `ReminderOccurrenceState`
4. Create `src/reminder/reminder-occurrence-state.entity.ts` — `@Entity('reminder_occurrence_state')` with: `id`, `reminderId` (integer, `reminder_id`), `occurrenceDate` (text, `occurrence_date`), `status` (`ReminderOccurrenceStatus`, text), `actedAt` (`@CreateDateColumn`, `acted_at`); `@ManyToOne` → `ReminderDefinition` with `onDelete: 'CASCADE'`; `@Unique(['reminderId', 'occurrenceDate'])`
5. Generate TypeORM migration creating the `reminder_definition` table with all columns, FK to `template(id)`, and three indexes: on `active`, on `linked_template_id`, and composite on `(reminder_id, occurrence_date)`
6. Generate TypeORM migration creating the `reminder_occurrence_state` table with FK to `reminder_definition(id) ON DELETE CASCADE` and unique constraint on `(reminder_id, occurrence_date)`
7. Create `src/reminder/reminder.module.ts` — imports `TypeOrmModule.forFeature([ReminderDefinition, ReminderOccurrenceState])`, declares the controller, provides and exports the service
8. Register `ReminderDefinition` and `ReminderOccurrenceState` in the `entities` array in `AppModule`; add `ReminderModule` to `imports`

---

### Phase 2: Backend — Recurrence Engine

9. Create `src/reminder/recurrence.utils.ts` — export pure function `computeOccurrences(definition, from: string, to: string): string[]` returning `YYYY-MM-DD` dates in the window; handles `ONCE` (returns `[anchorDate]` if in range), `DAILY` (every `interval` days from `anchorDate`), `WEEKLY` (every `interval` weeks on each listed weekday, anchored from the week containing `anchorDate`); **short-circuits once the current candidate exceeds `to`** — never precomputes-then-filters
10. Export `prepStartDate(occurrenceDate: string, leadTimeDays: number): string` — returns `occurrenceDate` minus `leadTimeDays` calendar days as `YYYY-MM-DD`
11. Export `computeDerivedFields(occurrenceDate, leadTimeDays, today): { isInPrepWindow, isOverdue, daysUntilOccurrence }` — `isInPrepWindow` is `today >= prepStartDate(...)`, `isOverdue` is `today > occurrenceDate`, `daysUntilOccurrence` is signed days from today to occurrenceDate (negative if past)
12. Write `src/reminder/recurrence.utils.spec.ts` covering: `ONCE` in/out of range, `ONCE` with leadTimeDays making visible early, `DAILY interval=1` over multiple days, `DAILY interval=2` skip-day, `WEEKLY interval=1` single weekday, `WEEKLY interval=2` biweekly, `WEEKLY interval=1` multiple weekdays per week, `WEEKLY` anchor not on listed weekday (first occurrence on next matching day), short-circuit at window end, `isInPrepWindow` exactly on prepStartDate, `isInPrepWindow` one day before, `isOverdue` for past/present/future, `daysUntilOccurrence` signed values

---

### Phase 3: Backend — DTOs

13. Create `src/reminder/dto/create-reminder.dto.ts` — `title` (`@IsString @IsNotEmpty @MaxLength(200)`), `description?` (`@IsOptional @IsString @MaxLength(4000)`), `category?` (`@IsOptional @IsString @MaxLength(50)`), `cadence` (`@IsEnum(ReminderCadence)`), `interval?` (`@IsOptional @IsInt @Min(1) @Max(52)`), `anchorDate` (`@IsDateString`), `weekdays?` (`@IsOptional @IsArray`, each element `@IsInt @Min(0) @Max(6)`, `@ArrayUnique`), `timeOfDay?` (`@IsOptional @Matches(/^\d{2}:\d{2}$/)`), `leadTimeDays?` (`@IsOptional @IsInt @Min(0) @Max(365)`), `linkedTemplateId?` (`@IsOptional @IsInt @IsPositive`)
14. Create `src/reminder/dto/update-reminder.dto.ts` — all same fields as create but all `@IsOptional`, plus `active?` (`@IsOptional @IsBoolean`)
15. Create `src/reminder/dto/update-reminder-occurrence.dto.ts` — `status`: `@IsIn(['COMPLETED', 'DISMISSED', 'OPEN'])`

---

### Phase 4: Backend — Service

16. Create `src/reminder/reminders.service.ts` — inject `@InjectRepository(ReminderDefinition)`, `@InjectRepository(ReminderOccurrenceState)`, and `@InjectRepository(Template)` (for linkedTemplateId validation)
17. Implement `findAll(filters)` — apply optional WHERE clauses for `active`, `linkedTemplateId`, `category`; compute `lastCompletedOccurrenceDate` with a single `MAX(occurrence_date) WHERE status = 'COMPLETED' GROUP BY reminder_id` query; compute `nextOccurrenceDate`/`nextPrepStartDate` using `computeOccurrences` with a 90-day forward window; return merged objects including `createdAt` and `updatedAt`
18. Implement `findOne(id)` — throws `NotFoundException` if missing; returns full definition with derived fields
19. Implement `create(dto)` — validates `linkedTemplateId` exists via `existsBy({ id })`, throws `BadRequestException` if not; saves and returns response with derived fields
20. Implement `update(id, dto)` — throws `NotFoundException` if missing; re-validates `linkedTemplateId` if it changes; saves all DTO fields
21. Implement `remove(id)` — throws `NotFoundException` if missing; deletes definition; cascade removes occurrence state rows
22. Implement `getAgenda(from, to)` — load all active definitions; call `computeOccurrences` per definition; batch-load all occurrence-state rows for those reminders in the window in one query; build full `ReminderAgendaItem` per occurrence using `computeDerivedFields` and state overlay; return flat array sorted by `occurrenceDate`
23. Implement `updateOccurrenceState(id, occurrenceDate, dto)` — verify reminder exists; call `computeOccurrences(definition, occurrenceDate, occurrenceDate)` to confirm the date is a valid occurrence, throw `BadRequestException` if empty; if `status = OPEN`, delete state row and return `{ reminderId, occurrenceDate, status: 'OPEN' }`; otherwise upsert state row

---

### Phase 5: Backend — Controller & Template Guard

24. Create `src/reminder/reminders.controller.ts` — `@Controller('reminders')`; routes declared in order to prevent `agenda` being consumed as `:id`: `GET /` (query: active, linkedTemplateId, category), `GET /agenda` (query: from, to — validate `from <= to` and window ≤ 90 days, else `BadRequestException`), `GET /:id` (`ParseIntPipe`), `POST /` (`HttpCode(201)`), `PUT /:id`, `DELETE /:id` (`HttpCode(204)`), `PATCH /:id/occurrences/:occurrenceDate`
25. Update templates.service.ts `remove()` — inject `@InjectRepository(ReminderDefinition)` (add `TypeOrmModule.forFeature([ReminderDefinition])` to `TemplatesModule`); before deleting, call `reminderRepo.existsBy({ linkedTemplateId: id })`; throw `ConflictException` with descriptive message if any reminders reference the template

---

### Phase 6: Backend — Dashboard Extension

26. Update dashboard.controller.ts — add `@Query('upcomingDays', new DefaultValuePipe(7), ParseIntPipe) upcomingDays: number` to `getToday()`; pass `upcomingDays` to `dashboardService.getToday(upcomingDays)`
27. Update dashboard.service.ts — add `upcomingDays: number` param to `getToday()`; inject `RemindersService`; call `remindersService.getAgenda(today, today + upcomingDays)`; split results into `dueNow` (`isInPrepWindow && status OPEN`) and `upcoming` (`!isInPrepWindow && status OPEN && occurrenceDate within horizon`); add `reminders: { dueNow, upcoming }` to `DashboardResponse`
28. Update dashboard.module.ts — import `ReminderModule` to make `RemindersService` injectable in `DashboardService`

---

### Phase 7: Backend — Tests

29. Write `src/reminder/reminders.service.spec.ts` — mock both repos; test: `findAll` returns derived fields, `findAll` with each filter, `create` with valid `linkedTemplateId`, `create` with nonexistent `linkedTemplateId` (400), `update` not-found and valid, `remove` not-found and valid, `getAgenda` correct `isInPrepWindow`/`isOverdue` placement, `updateOccurrenceState` valid date, non-occurrence date (400), OPEN deletes row, repeated COMPLETED is idempotent
30. Write `src/reminder/reminders.controller.spec.ts` — test all 7 routes, agenda window rejections (from > to, > 90 days), 404 paths for GET/PUT/DELETE, 409 not applicable here
31. Update dashboard.service.spec.ts — test `dueNow` contains occurrences in prep window, `upcoming` outside prep window within horizon, `upcoming` empty when `upcomingDays=0`, completed/dismissed excluded from both, runs/todos unaffected
32. Update templates.controller.spec.ts — add test that DELETE returns 409 when a `ReminderDefinition` row references the template

---

### Phase 8: Backend Verification

33. Run `npm test` in `checklist-execution-system-api` — all tests pass
34. Run `npm run migration:run` — both tables created; existing template/instance/todo data unaffected
35. Manual: `POST /api/reminders` DAILY with `leadTimeDays: 1` — confirm 201 with `nextOccurrenceDate` and `nextPrepStartDate`
36. Manual: `POST /api/reminders` WEEKLY biweekly (interval=2, weekdays=[5], anchorDate=2026-04-03) — confirm occurrences land on correct Fridays
37. Manual: `GET /api/reminders/agenda?from=2026-03-31&to=2026-04-10` — verify `isInPrepWindow`, `isOverdue`, `daysUntilOccurrence` match today's date
38. Manual: `PATCH /api/reminders/1/occurrences/2026-04-03` with `COMPLETED` — confirm 200; repeat — confirm idempotent
39. Manual: `PATCH /api/reminders/1/occurrences/2099-01-01` (non-occurrence date) — confirm 400
40. Manual: `PATCH .../occurrences/2026-04-03` with `OPEN` — confirm state row deleted; re-GET shows OPEN
41. Manual: `GET /api/dashboard?upcomingDays=7` — confirm `reminders.dueNow` and `reminders.upcoming` present; runs and todos unaffected
42. Manual: `GET /api/dashboard` (no param) — default `upcomingDays=7` behavior confirmed
43. Manual: `DELETE /api/templates/:id` for reminder-linked template — confirm 409

---

### Phase 9: Frontend — Models & API Service

44. Add `ReminderCadence` (`'ONCE' | 'DAILY' | 'WEEKLY'`) and `ReminderOccurrenceStatus` (`'COMPLETED' | 'DISMISSED' | 'OPEN'`) types to api.models.ts
45. Add `ReminderDefinition` interface to `api.models.ts` — all entity fields, `weekdays: number[] | null`, `nextOccurrenceDate: string | null`, `nextPrepStartDate: string | null`, `lastCompletedOccurrenceDate: string | null`, `createdAt: string`, `updatedAt: string | null`
46. Add `ReminderAgendaItem` interface to `api.models.ts` — `reminderId`, `title`, `description`, `category`, `occurrenceDate`, `prepStartDate`, `timeOfDay`, `status: ReminderOccurrenceStatus`, `isInPrepWindow`, `isOverdue`, `daysUntilOccurrence`, `linkedTemplate: { id: number; name: string } | null`, `canStartRun: boolean`
47. Update `DashboardResponse` interface in `api.models.ts` — add `reminders: { dueNow: ReminderAgendaItem[]; upcoming: ReminderAgendaItem[] }`
48. Add `CreateReminderDto`, `UpdateReminderDto`, `UpdateReminderOccurrenceDto` interfaces to `api.models.ts`
49. Create `src/app/services/reminders-api.service.ts` — `@Injectable({ providedIn: 'root' })`; `inject(HttpClient)`; `apiUrl = environment.apiUrl`; methods: `getReminders(filters?)`, `getReminder(id)`, `createReminder(dto)`, `updateReminder(id, dto)`, `deleteReminder(id)`, `getAgenda(from, to)`, `updateOccurrenceState(reminderId, occurrenceDate, dto)` — follow exact pattern of `todos-api.service.ts`
50. Update `dashboard-api.service.ts` `getDashboard()` — accept optional `upcomingDays?: number`; append `?upcomingDays=N` to URL when provided

---

### Phase 10: Frontend — Today Page

51. Update today.component.ts — add `reminders` signal typed as `{ dueNow: ReminderAgendaItem[]; upcoming: ReminderAgendaItem[] }` initialized to `{ dueNow: [], upcoming: [] }`; populate from dashboard response in `loadDashboard()`
52. Add `togglingOccurrenceKey` signal (`string | null`) for per-item loading state — key is `` `${reminderId}-${occurrenceDate}` ``
53. Add `markOccurrenceDone(item)` — sets key, calls `RemindersApiService.updateOccurrenceState(COMPLETED)`, then reloads dashboard; clears key on complete or error
54. Add `dismissOccurrence(item)` — same pattern with `DISMISSED`
55. Update template — add "Ready Now" section above runs; each item shows title, occurrence date, `today`/`in N days`/`overdue` framing, [Done] and [Dismiss] buttons (disabled while `togglingOccurrenceKey` matches), and [Start run] button only when `canStartRun` is true (navigates to `/runs/new?templateId=...`)
56. Add "Coming Up" section between "Ready Now" and runs — each item read-only showing title and `in N days` framing; no action buttons
57. Suppress "Ready Now" section heading entirely when both `dueNow` and `upcoming` are empty

---

### Phase 11: Frontend — Reminders Management Page

58. Create `src/app/pages/reminders/reminders.component.ts` — `OnPush`; signals: `reminders`, `loading`, `error`; fetches `getReminders()` on init; each card shows title, cadence summary, lead time, next occurrence, linked template name, active badge; actions: Edit (navigate to editor), Deactivate/Activate (PUT with toggled `active` then refresh), Delete (call `deleteReminder` then refresh)
59. Create `src/app/pages/reminders/reminder-editor/reminder-editor.component.ts` — handles create (`/reminders/new`) and edit (`/reminders/:id`) modes; form signals for each field; conditional fields based on cadence (ONCE hides interval and weekdays, DAILY shows interval, WEEKLY shows interval and weekday checkboxes 0–6); preview section computes next 3–5 occurrence dates locally using a `computeOccurrences` pure function matching backend logic; on save calls `createReminder` or `updateReminder` then navigates to `/reminders`
60. Add four ceremony preset buttons to the editor — "Daily Standup Prep" (DAILY interval=1 leadTimeDays=0), "Weekly Update" (WEEKLY interval=1 weekdays=[4] leadTimeDays=1), "Sprint Ceremony" (WEEKLY interval=2 weekdays=[5] leadTimeDays=2), "Backlog Refinement" (WEEKLY interval=2 weekdays=[2] leadTimeDays=1); clicking calls `applyPreset(preset)` which updates all relevant field signals; fields remain fully editable afterward
61. Add lazy-loaded routes to app.routes.ts — `'reminders'` → `RemindersComponent`, `'reminders/new'` → `ReminderEditorComponent`, `'reminders/:id'` → `ReminderEditorComponent`; follow existing lazy-load pattern
62. Add "Reminders" nav entry to nav.component.ts with `routerLink="/reminders"` between Todos and any trailing entries

---

### Phase 12: Frontend Verification

63. Run `npm test` in `checklist-execution-system-ui` — all existing tests pass
64. Manual: navigate to `/reminders` — empty state shown; no console errors
65. Manual: click "Daily Standup Prep" preset — fields populate; save — appears in list with correct cadence summary and next occurrence
66. Manual: create WEEKLY reminder (Fridays, interval=1, leadTimeDays=2) — editor preview shows 3–5 correct Fridays; list shows `nextOccurrenceDate` and `nextPrepStartDate`
67. Manual: create Sprint Retro linked to template — confirm Today shows it with [Start run]; clicking navigates to `/runs/new?templateId=...`
68. Manual: click [Done] on a "Ready Now" item — disappears from Today; stays gone on reload; definition still active on Reminders page
69. Manual: click [Dismiss] — same disappearance behavior
70. Manual: deactivate a reminder — Today no longer shows it in either section
71. Manual: attempt to delete a template linked to a reminder from the Templates page — 409 error surfaces in UI
72. Manual: `GET /api/dashboard` with no reminders defined — `reminders.dueNow: []` and `upcoming: []`; runs and todos render identically to pre-feature behavior

---

72 tasks, 12 phases. Phases 1–2 and 2–3 are sequential within the backend. Phases 9–12 are frontend-only and can start after Phase 8 completes. Within backend phases, Phase 6 (dashboard extension) depends on Phase 4 (service) being done. Phase 7 (tests) can be written alongside Phases 4–6 rather than strictly after.