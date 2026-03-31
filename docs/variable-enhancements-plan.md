# Plan: Custom Variable Delimiters

## TL;DR
Add configurable `variablePrefix` and `variableSuffix` fields to the Template entity so users can define their own placeholder pattern (e.g. `@{`/`}` instead of `{{`/`}}`). This prevents collisions when step instructions contain literal `{{...}}` for external systems (Helm, Terraform, Mustache, ARM templates, etc.). Defaults to `{{`/`}}` for backward compatibility.

## Decisions
- **Scope:** Per template (all steps in a template share the same delimiters)
- **Default:** `{{` and `}}` — existing templates work unchanged with no migration of existing data
- **Storage:** Two nullable text columns on `template` table; null means "use default"
- **Validation:** Both prefix and suffix must be provided together (or both null). Non-empty, max ~10 chars. Must not be identical to each other.

---

## Steps

### Phase A: Backend changes

**A1. Migration — add columns to `template` table** *[no dependencies]*
- New migration adding `variable_prefix TEXT NULL` and `variable_suffix TEXT NULL` to the `template` table
- No data migration needed — existing rows stay null (which means default `{{`/`}}`)
- File: `src/database/migrations/<timestamp>-AddVariableDelimiters.ts`

**A2. Template entity — add fields** *[depends on A1]*
- Add `variablePrefix: string | null` and `variableSuffix: string | null` to `Template` entity (`src/template/template.entity.ts`)
- Map to `variable_prefix` / `variable_suffix` columns

**A3. Template DTOs — accept delimiters** *[depends on A2]*
- Add optional `variablePrefix?: string` and `variableSuffix?: string` to `CreateTemplateDto` and `UpdateTemplateDto`
- Add `class-validator` decorators: `@IsOptional()`, `@IsString()`, `@MaxLength(10)`, and custom validation that both must be provided together or both omitted
- Files: `src/template/dto/create-template.dto.ts`, `src/template/dto/update-template.dto.ts`

**A4. VariableExtractionService — parameterize delimiters** *[no dependencies]*
- Change `extract(steps)` signature to `extract(steps, prefix?: string, suffix?: string)`
- Default params to `'{{'` and `'}}'`
- Build regex dynamically from escaped prefix/suffix instead of hardcoded `/{{(.*?)}}/g`
- Use `RegExp` constructor with `escapeRegex()` utility: `new RegExp(escapeRegex(prefix) + '(.*?)' + escapeRegex(suffix), 'g')`
- File: `src/instance/variable-extraction.service.ts`

**A5. InstancesService — use template delimiters** *[depends on A2, A4]*
- In `create()`, read `template.variablePrefix` and `template.variableSuffix` from the loaded template
- Pass delimiters to `VariableExtractionService.extract()` (if the service is used during creation — currently it isn't directly, but `renderInstructions` uses the same regex)
- Update `renderInstructions()` to accept prefix/suffix params and build regex dynamically
- When rendering fallback for unmatched variables, reconstruct using the template's delimiters (e.g., `@{key}` not `{{key}}`)
- File: `src/instance/instances.service.ts`

**A6. Template controller/service — pass through delimiter fields** *[depends on A3]*
- Ensure create/update template endpoints accept and persist the new fields
- Return `variablePrefix` and `variableSuffix` in GET responses (even when null)
- Files: `src/template/templates.service.ts`, `src/template/templates.controller.ts`

**A7. Backend tests** *[depends on A4, A5, A6]*
- `variable-extraction.service.spec.ts`: Add tests for custom delimiters (`@{`/`}`, `<%`/`%>`, `${`/`}`, etc.) and verify default behavior unchanged
- `instances.service.spec.ts`: Add test creating instance from template with custom delimiters, verify `renderedInstructions` uses the right pattern
- `instances.controller.spec.ts`: Integration test — create template with custom delimiters, add step with `@{service}`, start run with variables, assert rendered output
- `templates.controller.spec.ts`: Test creating/updating template with delimiter fields, verify persistence and retrieval

### Phase B: Frontend changes *[parallel with Phase A tests]*

**B1. API models — add delimiter fields** *[no dependencies]*
- Add `variablePrefix: string | null` and `variableSuffix: string | null` to `Template` interface
- Add optional `variablePrefix?: string` and `variableSuffix?: string` to `CreateTemplateDto` and `UpdateTemplateDto`
- File: `src/app/models/api.models.ts`

**B2. Template editor — delimiter configuration UI** *[depends on B1]*
- Add an expandable/collapsible "Advanced Settings" or "Variable Delimiters" section to the template editor
- Two small text inputs: "Prefix" (placeholder: `{{`) and "Suffix" (placeholder: `}}`)
- Show current effective delimiters; empty means default `{{`/`}}`
- Wire to create/update template API calls
- File: `src/app/pages/templates/template-editor/template-editor.component.ts`

**B3. extractVariables() — parameterize delimiters** *[no dependencies]*
- Change `extractVariables(text: string)` to `extractVariables(text: string, prefix?: string, suffix?: string)`
- Default to `'{{'` and `'}}'`
- Build regex dynamically (same approach as backend: escape special chars, use RegExp constructor)
- File: `src/app/pages/runs/start-run/start-run.component.ts`

**B4. Start-run component — use template delimiters** *[depends on B1, B3]*
- After selecting a template, read `template.variablePrefix`/`variableSuffix`
- Pass them to `extractVariables()` when building the dynamic form
- File: `src/app/pages/runs/start-run/start-run.component.ts`

**B5. Frontend tests** *[depends on B3, B4]*
- `start-run.component.spec.ts`: Add `extractVariables` tests with custom delimiters; add component test selecting a template with custom delimiters and verifying form controls are built correctly
- Template editor tests: verify delimiter inputs appear and are sent with create/update calls

### Phase C: Verification

**C1. Automated tests** — Run `npm test` in both repos; all existing + new tests pass
**C2. Migration check** — Run migration on existing database, verify existing templates unaffected
**C3. Manual walkthrough:**
  1. Create template with default delimiters → add step with `{{serviceName}}` → start run → verify variable form and rendering (backward compat)
  2. Create template with custom delimiters `@{` / `}` → add step with `@{serviceName}` → start run → verify form extracts `serviceName` and rendering substitutes correctly
  3. Create template with custom delimiters → add step containing literal `{{helmValue}}` alongside `@{myVar}` → start run → verify only `myVar` is extracted, `{{helmValue}}` passes through untouched
  4. Edit existing template to add/change/remove custom delimiters → verify behavior updates

---

## Relevant files

### Backend (`checklist-execution-system-api`)
- `src/template/template.entity.ts` — add `variablePrefix`, `variableSuffix` columns
- `src/template/dto/create-template.dto.ts` — add optional delimiter fields + validation
- `src/template/dto/update-template.dto.ts` — add optional delimiter fields + validation
- `src/template/templates.service.ts` — pass through new fields on create/update
- `src/template/templates.controller.ts` — no logic changes, but verify DTO passthrough
- `src/instance/variable-extraction.service.ts` — parameterize `extract()` with prefix/suffix, build regex dynamically
- `src/instance/instances.service.ts` — update `create()` and `renderInstructions()` to use template delimiters
- `src/database/migrations/` — new migration for `variable_prefix`/`variable_suffix` columns
- `src/instance/variable-extraction.service.spec.ts` — tests for custom delimiters
- `src/instance/instances.service.spec.ts` — test rendering with custom delimiters
- `src/instance/instances.controller.spec.ts` — integration test with custom delimiters

### Frontend (`checklist-execution-system-ui`)
- `src/app/models/api.models.ts` — add delimiter fields to `Template`, `CreateTemplateDto`, `UpdateTemplateDto`
- `src/app/pages/templates/template-editor/template-editor.component.ts` — add delimiter config UI
- `src/app/pages/runs/start-run/start-run.component.ts` — parameterize `extractVariables()`, use template delimiters
- `src/app/pages/runs/start-run/start-run.component.spec.ts` — tests for custom delimiter extraction

---

## Further Considerations
1. **Regex-unsafe delimiters** — Delimiters like `(` or `$` are valid regex metacharacters. The `escapeRegex()` utility must properly escape all special characters. Recommend a small utility function shared/tested independently.
2. **Step editor hint** — Consider showing the effective delimiter pattern in the step editor as a UX hint (e.g., "Variables use @{ and } in this template"). This is a nice-to-have polish item that could be deferred.
