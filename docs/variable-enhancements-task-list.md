# Variable Enhancements — Phased Task List

## Phase 1: Backend — Custom Delimiters

1. Generate new TypeORM migration adding `variable_prefix TEXT NULL` and `variable_suffix TEXT NULL` columns to the `template` table
2. Add `variablePrefix: string | null` and `variableSuffix: string | null` fields to `Template` entity, mapped to the new columns
3. Add optional `variablePrefix` and `variableSuffix` to `CreateTemplateDto` with `@IsOptional()`, `@IsString()`, `@MaxLength(10)`, and a custom "both-or-neither" validator
4. Add identical delimiter fields + validator to `UpdateTemplateDto`
5. Add `escapeRegex(str: string): string` utility function (escapes regex metacharacters) — used by extraction and rendering
6. Refactor `VariableExtractionService.extract()` to accept optional `prefix` and `suffix` params; default to `{{`/`}}`; build regex dynamically using `escapeRegex`
7. Update `TemplatesService.create()` and `TemplatesService.update()` to persist the new delimiter fields
8. Update `InstancesService.create()` to read `template.variablePrefix`/`variableSuffix` and pass them to `renderInstructions()`
9. Update `InstancesService.renderInstructions()` to accept and use custom delimiters via dynamic regex; unmatched variable fallback reconstructs using the template's delimiters
10. Add tests to `variable-extraction.service.spec.ts` covering custom delimiters and unchanged default behavior
11. Add tests to `instances.service.spec.ts` for rendering with custom delimiters
12. Add integration tests to `instances.controller.spec.ts` — create template with `@{`/`}`, add step, start run, assert rendered output
13. Add tests to `templates.controller.spec.ts` for create/update/get with delimiter fields

## Phase 2: Frontend — Custom Delimiters

14. Add `variablePrefix: string | null` and `variableSuffix: string | null` to the `Template` interface in `api.models.ts`
15. Add optional `variablePrefix` and `variableSuffix` to `CreateTemplateDto` and `UpdateTemplateDto` interfaces in `api.models.ts`
16. Add `escapeRegex(str: string): string` utility to the frontend (can be a small shared utility alongside `start-run.component.ts` or a shared helper file)
17. Refactor `extractVariables()` in `start-run.component.ts` to accept optional `prefix` and `suffix` params; build regex dynamically using the frontend `escapeRegex`
18. Update `selectTemplate()` in `StartRunComponent` to read `template.variablePrefix`/`variableSuffix` and pass them to `extractVariables()`
19. Add a collapsible "Variable Delimiters" section to `template-editor.component.ts` with two text inputs (Prefix, Suffix); wire to create/update template API calls
20. Update `start-run.component.spec.ts`: add `extractVariables` tests with custom delimiters and a component test verifying form fields are built from a template with custom delimiters
21. Update template editor tests: verify delimiter inputs are rendered and values are sent with create/update calls

## Phase 3: Verification — Custom Delimiters

22. Run `npm test` in `checklist-execution-system-api` — all tests pass
23. Run `npm test` in `checklist-execution-system-ui` — all tests pass
24. Run migration against the existing database; confirm existing templates are unaffected (null delimiter columns)
25. Manual: create template with default delimiters, add `{{serviceName}}` step, start run — confirm backward compat
26. Manual: create template with `@{`/`}`, add `@{serviceName}` step, start run — confirm `serviceName` form field appears and renders correctly
27. Manual: step containing both literal `{{helmValue}}` and `@{myVar}` — confirm only `myVar` is extracted
28. Manual: edit existing template to add then remove custom delimiters — confirm behavior updates correctly

## Phase 4: Backend — Transform Engine

29. Create `src/instance/transform-registry.ts` — plain map of transform name → pure function `(value, ...args) => string`, with initial set: `upper`, `lower`, `capitalize`, `title_case`, `trim`, `remove_spaces`, `collapse_spaces`, `snake_case`, `kebab_case`, `camel_case`, `pascal_case`, `replace(s, r)`, `default(fallback)`, `truncate(n)`, `pad_left(n, ch)`, `pad_right(n, ch)`. Export `getTransform(name)`.
30. Create `src/instance/transform-pipeline.ts` — parser that splits inner placeholder content on `|` (respecting quoted strings), identifies the variable name (first segment) and transform chain. Export `evaluate(expression, variables)` which applies the chain left-to-right; leaves the full placeholder intact on unknown transform name or missing variable with no `default` in chain.
31. Update `InstancesService.renderInstructions()` to replace the direct `variables[key.trim()]` lookup with a call to `evaluate(capturedContent, variables)` from the pipeline
32. Update `VariableExtractionService.extract()` to strip pipe expressions — after capturing inner content, split on `|` and take only the first segment (trimmed) as the variable name
33. Unit test every built-in transform in `transform-registry.spec.ts` — happy path, edge cases (empty string, special chars, missing args)
34. Unit test the pipeline parser and evaluator in `transform-pipeline.spec.ts` — single transform, chained transforms, parameterized transforms, unknown transform (fail-safe), missing variable, quoted args containing commas or pipes
35. Update `variable-extraction.service.spec.ts` — verify `extract()` returns `["title"]` from `{{title | upper | remove_spaces}}`, not the full expression
36. Update `instances.service.spec.ts` — test rendering with single transform, chain, and parameterized transform
37. Update `instances.controller.spec.ts` — integration test: step `Deploy {{service | upper}} v{{version | replace(".", "_")}}`, start run, assert rendered output

## Phase 5: Frontend — Pipe-Aware Extraction

38. Update `extractVariables()` in `start-run.component.ts` — after matching placeholder content, extract base variable name by splitting on `|` and taking first segment: `match[1].split('|')[0].trim()`
39. Update `start-run.component.spec.ts` — add tests: `{{title | upper}}` → `["title"]`, `{{name | replace('_', '-') | lower}}` → `["name"]`, deduplication across same variable with different pipes, all existing tests still pass

## Phase 6: Verification — Transforms

40. Run `npm test` in both repos — all tests pass
41. Manual: plain `{{service}}` with no pipes still works
42. Manual: `{{service | upper}}` with `service=api` → renders `API`
43. Manual: `{{title | remove_spaces | lower}}` with `title=My Service` → renders `myservice`
44. Manual: `{{version | replace(".", "_")}}` with `version=1.2.3` → renders `1_2_3`
45. Manual: `{{optional | default("N/A")}}` with no value provided → renders `N/A`
46. Manual: `{{service | nonexistent}}` → placeholder left intact (no crash)
47. Manual: same variable used with and without pipes in same step (`{{service}}` and `{{service | upper}}`) — both resolve from one form field
48. Manual: transforms + custom delimiters — template with `@{`/`}`, step with `@{service | lower}` → works correctly
