# Plan: Variable Enhancements — Custom Delimiters + Pipe Transforms

## TL;DR
Two incremental enhancements to the template variable system:
1. **Custom delimiters** (Phases A–C): Configurable `variablePrefix`/`variableSuffix` per template so users can avoid collisions with `{{...}}` used by external systems (Helm, Terraform, Mustache, ARM templates, etc.). Defaults to `{{`/`}}`.
2. **Pipe transforms** (Phases D–F): Built-in string transforms using pipe syntax — `{{title | upper | remove_spaces}}` — so rendered instructions can apply formatting without the user manually transforming values. Builds on the delimiter infrastructure.

## Decisions
- **Scope:** Per template (all steps in a template share the same delimiters)
- **Default:** `{{` and `}}` — existing templates work unchanged with no migration of existing data
- **Storage:** Two nullable text columns on `template` table; null means "use default"
- **Validation:** Both prefix and suffix must be provided together (or both null). Non-empty, max ~10 chars. Must not be identical to each other.
- **Transform syntax:** Pipe-based — `{{varName | fn1 | fn2(arg)}}`. Left-to-right evaluation. Parsed by splitting inner content on `|` after isolating the variable name.
- **Transform scope:** Built-in functions only (Tier 2). No user-defined JavaScript (Tier 3 deferred).
- **Transform location:** All transform logic lives on the backend in `renderInstructions()`. The frontend only extracts *variable names* (strips pipes) for the form — it never evaluates transforms.
- **Transform ordering:** Transforms depend on custom delimiters being implemented first (shares the dynamic regex infrastructure).

---

## Steps

### Phase A: Backend — Custom delimiters

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

### Phase B: Frontend — Custom delimiters *[parallel with Phase A tests]*

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

### Phase C: Verification — Delimiters

**C1. Automated tests** — Run `npm test` in both repos; all existing + new tests pass
**C2. Migration check** — Run migration on existing database, verify existing templates unaffected
**C3. Manual walkthrough:**
  1. Create template with default delimiters → add step with `{{serviceName}}` → start run → verify variable form and rendering (backward compat)
  2. Create template with custom delimiters `@{` / `}` → add step with `@{serviceName}` → start run → verify form extracts `serviceName` and rendering substitutes correctly
  3. Create template with custom delimiters → add step containing literal `{{helmValue}}` alongside `@{myVar}` → start run → verify only `myVar` is extracted, `{{helmValue}}` passes through untouched
  4. Edit existing template to add/change/remove custom delimiters → verify behavior updates

### Phase D: Backend — Transform engine *[depends on Phases A–C]*

**D1. Transform registry** *[no dependencies]*
- Create `src/instance/transform-registry.ts` — a plain map of `string → (value: string, ...args: string[]) => string`
- Built-in transforms (initial set):
  - **Case:** `upper`, `lower`, `capitalize` (first letter), `title_case` (each word)
  - **Whitespace:** `trim`, `remove_spaces`, `collapse_spaces`
  - **Naming conventions:** `snake_case`, `kebab_case`, `camel_case`, `pascal_case`
  - **Substitution:** `replace(search, replacement)` — literal string replace (all occurrences)
  - **Fallback:** `default(fallbackValue)` — if variable is empty/missing, use fallback
  - **Truncation:** `truncate(maxLength)`, `pad_left(length, char)`, `pad_right(length, char)`
- Each function is a pure `(value, ...args) => string` with no side effects
- Export a lookup function: `getTransform(name: string): TransformFn | undefined`

**D2. Transform pipeline — parser + evaluator** *[depends on D1]*
- Create `src/instance/transform-pipeline.ts`
- **Parser:** Given the inner content of a placeholder (e.g. `title | upper | replace("_", "-")`):
  1. Split on `|` (respecting quoted strings containing `|` — use a simple state machine or regex)
  2. First segment = variable name (trimmed)
  3. Remaining segments = transform calls, each parsed into `{ name: string, args: string[] }`
  4. Argument parsing: `replace("_", "-")` → `{ name: "replace", args: ["_", "-"] }`
- **Evaluator:** `evaluate(expression: string, variables: Record<string, string>): string`
  1. Parse expression into variable name + transform chain
  2. Look up variable value from `variables` map
  3. Apply each transform in order (left-to-right), passing the result of the previous as input
  4. If a transform name is unknown, leave the full original placeholder intact (fail safe, don't crash)
  5. If the variable name is not found and no `default` transform is in the chain, leave placeholder intact

**D3. Update `renderInstructions()` to use the pipeline** *[depends on D2, A5]*
- In `InstancesService.renderInstructions()`, the regex `replace` callback currently does `variables[key.trim()]`
- Change it to call `evaluate(capturedContent, variables)` from the transform pipeline
- This naturally handles both plain variables (`{{title}}` → just a lookup) and piped expressions (`{{title | upper}}` → lookup + transform)
- File: `src/instance/instances.service.ts`

**D4. Update `VariableExtractionService` to strip pipes** *[depends on A4]*
- The `extract()` method captures inner content like `title | upper | replace("_", "-")`
- After capture, split on `|` and take only the first segment (trimmed) as the variable name
- This ensures the form prompt still shows `title`, not `title | upper | replace("_", "-")`
- File: `src/instance/variable-extraction.service.ts`

**D5. Backend tests** *[depends on D1–D4]*
- `transform-registry.spec.ts`: Unit test every built-in transform — `upper("hello")` → `"HELLO"`, `replace("a-b-c", "-", "_")` → `"a_b_c"`, `default("", "N/A")` → `"N/A"`, edge cases (empty string, special chars)
- `transform-pipeline.spec.ts`: Test parsing — `"title | upper"` → `{ varName: "title", transforms: [{ name: "upper", args: [] }] }`; test evaluation — chained transforms, unknown transforms, missing variables, quoted args with commas/pipes
- `variable-extraction.service.spec.ts`: Verify `extract()` returns `["title"]` from `{{title | upper | remove_spaces}}`, not the full expression
- `instances.service.spec.ts`: Test rendering with transforms — `{{service | upper}}` with `{ service: "api" }` → `"API"`
- `instances.controller.spec.ts`: Integration test — create template with step `Deploy {{service | upper}} v{{version | replace(".", "_")}}`, start run, verify rendered output

### Phase E: Frontend — Pipe-aware extraction *[depends on D4, parallel with D5]*

**E1. Update `extractVariables()` to strip pipes** *[no dependencies]*
- After matching placeholder content, split on `|` and take only the first segment (trimmed) as the variable name
- `extractVariables("Deploy {{service | upper}}")` → `["service"]` (not `["service | upper"]`)
- This is a small change: after capture group, do `match[1].split('|')[0].trim()`
- File: `src/app/pages/runs/start-run/start-run.component.ts`

**E2. Frontend tests** *[depends on E1]*
- `start-run.component.spec.ts`: Add tests verifying pipe expressions are stripped:
  - `extractVariables("{{title | upper}}")` → `["title"]`
  - `extractVariables("{{name | replace('_', '-') | lower}}")` → `["name"]`
  - `extractVariables("{{a | upper}} and {{a | lower}}")` → `["a"]` (deduplication still works)
  - Existing tests remain passing (backward compat)

### Phase F: Verification — Transforms *[depends on D5, E2]*

**F1. Automated tests** — Run `npm test` in both repos; all existing + new tests pass
**F2. Manual walkthrough:**
  1. Plain variable (no pipes) still works — backward compat confirmed
  2. Single transform: step with `{{service | upper}}`, provide `service=api` → renders `API`
  3. Chained transforms: `{{title | remove_spaces | lower}}`, provide `title=My Service` → renders `myservice`
  4. Parameterized transform: `{{version | replace(".", "_")}}`, provide `version=1.2.3` → renders `1_2_3`
  5. Default fallback: `{{optional | default("N/A")}}` with no value provided → renders `N/A`
  6. Unknown transform: `{{service | nonexistent}}` → leaves placeholder intact (no crash)
  7. Mixed in same step: `{{service}}` and `{{service | upper}}` both resolve from the same form field
  8. Transforms + custom delimiters: template with `@{` / `}` → step with `@{service | lower}` → works correctly

---

## Relevant files

### Backend (`checklist-execution-system-api`)
- `src/template/template.entity.ts` — add `variablePrefix`, `variableSuffix` columns
- `src/template/dto/create-template.dto.ts` — add optional delimiter fields + validation
- `src/template/dto/update-template.dto.ts` — add optional delimiter fields + validation
- `src/template/templates.service.ts` — pass through new fields on create/update
- `src/template/templates.controller.ts` — no logic changes, but verify DTO passthrough
- `src/instance/variable-extraction.service.ts` — parameterize `extract()` with prefix/suffix, build regex dynamically; update to strip pipe expressions and return only variable names
- `src/instance/instances.service.ts` — update `create()` and `renderInstructions()` to use template delimiters and evaluate pipe transforms
- `src/instance/transform-registry.ts` *(new)* — registry of built-in transform functions with name→implementation map
- `src/instance/transform-pipeline.ts` *(new)* — parses `varName | fn1 | fn2(arg)` expressions and evaluates the chain
- `src/database/migrations/` — new migration for `variable_prefix`/`variable_suffix` columns
- `src/instance/variable-extraction.service.spec.ts` — tests for custom delimiters and pipe-aware extraction
- `src/instance/instances.service.spec.ts` — test rendering with custom delimiters and transforms
- `src/instance/instances.controller.spec.ts` — integration tests for both features
- `src/instance/transform-registry.spec.ts` *(new)* — unit tests for each built-in transform
- `src/instance/transform-pipeline.spec.ts` *(new)* — unit tests for pipe expression parsing and evaluation

### Frontend (`checklist-execution-system-ui`)
- `src/app/models/api.models.ts` — add delimiter fields to `Template`, `CreateTemplateDto`, `UpdateTemplateDto`
- `src/app/pages/templates/template-editor/template-editor.component.ts` — add delimiter config UI
- `src/app/pages/runs/start-run/start-run.component.ts` — parameterize `extractVariables()`, strip pipes to extract base variable names, use template delimiters
- `src/app/pages/runs/start-run/start-run.component.spec.ts` — tests for custom delimiter extraction and pipe-aware extraction

---

## Further Considerations
1. **Regex-unsafe delimiters** — Delimiters like `(` or `$` are valid regex metacharacters. The `escapeRegex()` utility must properly escape all special characters. Recommend a small utility function shared/tested independently.
2. **Step editor hint** — Consider showing the effective delimiter pattern in the step editor as a UX hint (e.g., "Variables use @{ and } in this template"). This is a nice-to-have polish item that could be deferred.
3. **Transform discoverability** — Consider a help tooltip or documentation link in the step editor listing available transforms and their syntax. Could be deferred to a polish pass.
4. **Tier 3 user-defined transforms** — If built-in transforms prove insufficient, the pipe syntax naturally extends via a `custom:functionName` pipe. This would require a sandboxed JS execution environment (`isolated-vm`), a function editor UI, and per-template function storage. Deliberately deferred.
