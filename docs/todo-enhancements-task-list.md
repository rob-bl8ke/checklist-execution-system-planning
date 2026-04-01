# Todo Enhancements — Phased Task List

This task list assumes [current-spec.md](current-spec.md) and [todo-enhancements-plan.md](todo-enhancements-plan.md) are already aligned on the intended end state.

**47 tasks across 7 phases.** Backend first (Phases 1–3), then frontend (Phases 4–7). Each phase groups related work and is independently verifiable before moving on.

---

### Phase 1: Backend — Todo Schema, Enum, and DTOs

1. Create `src/todo/enums/todo-priority.enum.ts` exporting `TodoPriority` with values `LOW`, `NORMAL`, `HIGH`, `CRITICAL`
2. Update `src/todo/todo.entity.ts` — add `dueDate` mapped to `due_date` (`text`, nullable) and `priority` (`text`, default `NORMAL`)
3. Generate a TypeORM migration adding `due_date TEXT NULL` and `priority TEXT NOT NULL DEFAULT 'NORMAL'` to the `todo` table
4. Ensure the migration backfills existing rows with `priority = 'NORMAL'` and preserves existing Todo data unchanged
5. Update `src/todo/dto/create-todo.dto.ts` — keep `title` required; add optional `description`, optional `dueDate` (`@IsDateString()`), and optional `priority` (`@IsEnum(TodoPriority)`)
6. Update `src/todo/dto/update-todo.dto.ts` — support optional `title`, `description`, `dueDate`, `priority`, and `completed`, including edit flows that clear `description` or `dueDate`

---

### Phase 2: Backend — Todo Service Ordering and Dashboard Alignment

7. Refactor `src/todo/todos.service.ts` so all Todo ordering rules are defined in one place rather than split across endpoints
8. Add a reusable Todo priority ranking helper in `src/todo/todos.service.ts` for `CRITICAL > HIGH > NORMAL > LOW`
9. Update `TodosService.create()` to default `description` to `null`, `dueDate` to `null`, `priority` to `NORMAL`, `completed` to `false`, and `completedAt` to `null`
10. Update `TodosService.update()` to edit `title`, markdown detail (`description`), `dueDate`, and `priority` without breaking the existing `completedAt` toggle semantics
11. Implement deterministic `findAll()` ordering in `src/todo/todos.service.ts`: incomplete first, dated incomplete Todos by `dueDate` ascending, undated incomplete Todos next, then priority, then `createdAt` descending, then completed Todos by `completedAt` descending
12. If SQLite or TypeORM ordering becomes awkward for the mixed null and priority rules, keep the repository query simple and apply one server-side sort in `TodosService` with tests pinning the exact behavior
13. Update `src/dashboard/dashboard.service.ts` so the Today dashboard returns only incomplete Todos while preserving the same execution ordering used by `GET /api/todos`
14. Remove any duplicated Todo ordering logic between `DashboardService` and `TodosService` by delegating to one shared implementation or helper

---

### Phase 3: Backend — Tests and Verification

15. Update `src/todo/todos.service.spec.ts` — cover create defaults, editing metadata fields, clearing `description`, clearing `dueDate`, priority changes, and completed timestamp behavior
16. Add ordering tests to `src/todo/todos.service.spec.ts` covering overdue vs future due dates, undated Todos, priority precedence, and completed-last behavior
17. Update `src/todo/todos.controller.spec.ts` — cover `POST /api/todos` and `PATCH /api/todos/:id` with `description`, `dueDate`, and `priority`
18. Add validation coverage to `src/todo/todos.controller.spec.ts` for invalid `priority`, invalid `dueDate`, and unknown request fields
19. Update `src/todo/todos.controller.spec.ts` `GET /api/todos` assertions to verify the new ordering contract
20. Update `src/dashboard/dashboard.service.spec.ts` so Today still excludes completed Todos while preserving the richer ordering for incomplete ones
21. Run `npm test` in `checklist-execution-system-api`
22. Run the new Todo migration against the development database and confirm existing Todo rows remain intact with `priority = NORMAL`
23. Manual API smoke test: create a critical overdue Todo, a normal undated Todo, and a completed Todo; verify `GET /api/todos` and `GET /api/dashboard` ordering and field shapes

---

### Phase 4: Frontend — Todo Models and Shared Editor Component

24. Update `src/app/models/api.models.ts` — add `TodoPriority`, extend `Todo` with `dueDate` and `priority`, and extend `CreateTodoDto` / `UpdateTodoDto` with the richer Todo fields
25. Keep `src/app/services/todos-api.service.ts` structurally unchanged, but ensure the updated DTO and response types are used consistently
26. Create `src/app/pages/todos/todo-editor/todo-editor.component.ts` as a shared modal for both create and edit flows
27. Build the Todo editor UI using the same modal structure as `src/app/pages/templates/step-editor/step-editor.component.ts`
28. Add Todo editor form fields for `title`, markdown detail, `dueDate`, and `priority`, plus save and cancel actions
29. Add live markdown preview to the Todo editor using `ngx-markdown`, matching the existing step editor preview approach
30. Implement create and edit modes in the Todo editor so the same component can either call `createTodo()` or `updateTodo()` based on whether a Todo is provided

---

### Phase 5: Frontend — Todos Page UX

31. Update `src/app/pages/todos/todo-list/todo-list.component.ts` to replace the inline title-only add form with a modal-driven create flow
32. Add edit actions to existing Todo rows and wire them to the shared Todo editor modal
33. Add expand/collapse state per Todo in `todo-list.component.ts`, reusing the expand pattern from `src/app/pages/runs/run-execution/run-execution.component.ts`
34. Render markdown detail for expanded Todos using `ngx-markdown`; collapse cleanly when no detail exists
35. Show due date, overdue state, priority badge, and completion metadata in each Todo row
36. Keep completed Todos visible at the bottom of the page and preserve API order rather than re-sorting differently in the client
37. Add delete confirmation using `src/app/components/confirm-dialog/confirm-dialog.component.ts`
38. Keep quick complete and reopen behavior on the Todos page, ensuring UI state stays consistent after edit, toggle, and delete actions

---

### Phase 6: Frontend — Today Dashboard Todo Alignment

39. Update `src/app/pages/today/today.component.ts` so dashboard Todos show compact priority cues, due date, and overdue state while remaining quick-complete only
40. Keep Today’s Todo section read-only apart from completion toggles: no markdown expansion, no edit modal, no delete actions
41. Ensure Today uses the dashboard response order directly after filtering to incomplete items, without introducing a second conflicting client-side ordering rule

---

### Phase 7: Frontend — Tests and Verification

42. Create `src/app/pages/todos/todo-editor/todo-editor.component.spec.ts` covering create mode, edit mode, validation, and markdown preview
43. Update `src/app/pages/todos/todo-list/todo-list.component.spec.ts` — cover modal open and close, create flow, edit flow, expand/collapse behavior, overdue rendering, and delete confirmation
44. Update `src/app/pages/today/today.component.spec.ts` — cover compact Todo rendering, due-date and priority display, and quick-complete behavior
45. Run `npm test` in `checklist-execution-system-ui`
46. Manual UI smoke test on the Todos page: create a Todo with markdown links, edit it, clear its due date, change its priority, expand it, complete it, reopen it, and delete it
47. Manual UI smoke test on Today: verify overdue and high-priority Todos stand out, appear in API order, and can be marked complete without exposing edit or markdown expansion controls

---

47 tasks, 7 phases. Phases 1–3 are backend-focused and should complete before frontend integration starts. Within the frontend, Phase 4 should land before the page work in Phases 5–6, while Phase 7 can be written alongside Phases 5–6 rather than strictly after them.