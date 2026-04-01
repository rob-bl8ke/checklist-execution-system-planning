
**Plan**
1. Update the spec in checklist-execution-system-planning/docs/current-spec.md so Todos are defined with markdown detail, date-only due dates, priority, overdue highlighting, completed-last behavior, and editing through the existing PATCH route.
2. Extend the backend Todo model in checklist-execution-system-api/src/todo/todo.entity.ts with due date and priority, while reusing the existing description field as the markdown detail field instead of adding a second long-text column.
3. Update validation in checklist-execution-system-api/src/todo/dto/create-todo.dto.ts and checklist-execution-system-api/src/todo/dto/update-todo.dto.ts to accept optional markdown detail, date-only dueDate, and priority values Low, Normal, High, Critical, including support for clearing optional fields during edit.
4. Add a new Todo migration using the same style as checklist-execution-system-api/src/database/migrations/1774900000000-AddVariableDelimiters.ts. Store dueDate as local YYYY-MM-DD text and priority as text with NORMAL as the default for existing rows.
5. Move Todo ordering into checklist-execution-system-api/src/todo/todos.service.ts so the API returns incomplete todos first, then due date ascending with undated items after dated ones, then priority Critical to Low, then newest first, with completed todos visible at the bottom ordered by completedAt descending.
6. Align the Today feed in checklist-execution-system-api/src/dashboard/dashboard.service.ts with the same ordering model while still excluding completed todos from the dashboard.
7. Extend shared frontend contracts in checklist-execution-system-ui/src/app/models/api.models.ts. The service surface in checklist-execution-system-ui/src/app/services/todos-api.service.ts can stay structurally the same because the existing CRUD methods already fit the new model.
8. Replace the current inline-only add flow in checklist-execution-system-ui/src/app/pages/todos/todo-list/todo-list.component.ts with a shared create/edit modal so users can set title, markdown detail, due date, and priority at creation time and later edit the same fields. Reuse the modal and live markdown preview patterns from checklist-execution-system-ui/src/app/pages/templates/step-editor/step-editor.component.ts.
9. Rework the Todos list UI in checklist-execution-system-ui/src/app/pages/todos/todo-list/todo-list.component.ts to add expand/collapse for rendered markdown detail, overdue highlighting, due date display, priority badges, completed-last grouping, and delete confirmation via checklist-execution-system-ui/src/app/components/confirm-dialog/confirm-dialog.component.ts. Reuse the expand/collapse pattern from checklist-execution-system-ui/src/app/pages/runs/run-execution/run-execution.component.ts.
10. Keep checklist-execution-system-ui/src/app/pages/today/today.component.ts as a quick-action surface only, but update it to show due date and overdue state, and optionally a compact priority badge. Editing and markdown expansion stay on the Todos page only.
11. Expand backend tests in checklist-execution-system-api/src/todo/todos.service.spec.ts, checklist-execution-system-api/src/todo/todos.controller.spec.ts, and checklist-execution-system-api/src/dashboard/dashboard.service.spec.ts, then expand UI tests in checklist-execution-system-ui/src/app/pages/todos/todo-list/todo-list.component.spec.ts for modal create/edit, ordering, overdue styling, expand/collapse, and delete confirmation.

**Relevant files**
- checklist-execution-system-planning/docs/current-spec.md — update Todo table, Todo endpoint payloads, Today/Todos UX, and editing semantics.
- checklist-execution-system-api/src/todo/todo.entity.ts — add due date and priority fields while keeping description as markdown detail.
- checklist-execution-system-api/src/todo/dto/create-todo.dto.ts — validate title plus optional detail, dueDate, and priority.
- checklist-execution-system-api/src/todo/dto/update-todo.dto.ts — support partial updates and clearing optional fields.
- checklist-execution-system-api/src/todo/todos.service.ts — centralize create/update mapping and business ordering.
- checklist-execution-system-api/src/todo/todos.controller.spec.ts — extend integration coverage for the richer payloads and ordering.
- checklist-execution-system-api/src/todo/todos.service.spec.ts — extend unit coverage for ordering and update semantics.
- checklist-execution-system-api/src/dashboard/dashboard.service.ts — keep Today’s todo feed aligned with the new ordering and fields.
- checklist-execution-system-api/src/dashboard/dashboard.service.spec.ts — verify dashboard todo filtering and ordering.
- checklist-execution-system-api/src/database/migrations/1774900000000-AddVariableDelimiters.ts — reference the current migration style when adding the new Todo migration.
- checklist-execution-system-ui/src/app/models/api.models.ts — extend Todo request and response interfaces.
- checklist-execution-system-ui/src/app/pages/todos/todo-list/todo-list.component.ts — introduce modal create/edit, ordering, expand/collapse, and visual state handling.
- checklist-execution-system-ui/src/app/pages/todos/todo-list/todo-list.component.spec.ts — extend UI coverage for the new interactions.
- checklist-execution-system-ui/src/app/pages/today/today.component.ts — surface due date and priority cues in the dashboard todo section.
- checklist-execution-system-ui/src/app/pages/templates/step-editor/step-editor.component.ts — reuse modal form and live markdown preview patterns.
- checklist-execution-system-ui/src/app/pages/runs/run-execution/run-execution.component.ts — reuse expand/collapse and markdown display patterns.
- checklist-execution-system-ui/src/app/components/confirm-dialog/confirm-dialog.component.ts — reuse for delete confirmation.

**Decisions**
1. Reuse the current description field as the markdown detail field.
2. Use date-only due dates, not date+time.
3. Sort Todos by due date first, then priority, then newest, with completed items still visible at the bottom.
4. Use a modal dialog for editing, and use the same form for create and edit.
5. Include Today dashboard ordering and overdue cues, but exclude editing or markdown expansion from Today.
6. Included scope: richer Todo data model, modal create/edit, expandable markdown detail on the Todos page, overdue and priority ordering, Today dashboard alignment, tests, and spec updates.
7. Excluded scope: reminder-style Todo notifications, inline editing inside list rows, dedicated Todo detail pages, and markdown expansion or edit controls on Today.

**Verification**
1. API tests should cover create, edit, clear-field behavior, invalid date or priority validation, and completed-last ordering.
2. Dashboard tests should verify incomplete-only filtering still holds while the new ordering is preserved.
3. UI tests should cover modal open and save flows, expand/collapse rendering of markdown detail, overdue styling, and delete confirmation.
4. Manual smoke test should create an overdue critical todo with markdown links, an undated normal todo, and a completed todo to confirm ordering, rendering, highlighting, edit persistence, and Today visibility.
