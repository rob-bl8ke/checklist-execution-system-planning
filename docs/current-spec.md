# Current Specification — Checklist Execution System

> This document reflects the implemented state of the system, including all variable enhancement features (custom delimiters + pipe transforms) built after the initial spec.

---

# 1. System Overview

The system is a **Runbook + Todo manager** designed for engineers performing repeatable operational tasks.

Examples:

* Deploy services
* Incident response
* Release checklists
* CI/CD runbooks
* Personal engineering tasks

The system contains **three functional areas**:

```
Runbooks (Template-based processes)
Runs (Instances of templates)
Todos (Standalone tasks)
```

The **default screen is the "Today" dashboard**, which guides users through the **next step of each runbook** while also displaying todos.

---

# 2. Core Concepts

## 2.1 Template

A **template** defines a reusable runbook.

Example:

```
Deployment Process
1 Pull latest develop
2 Run tests
3 Build artifact
4 Deploy staging
5 Smoke tests
6 Deploy production
```

Templates contain **ordered steps**.

Templates may optionally configure **custom variable delimiters** (`variablePrefix` and `variableSuffix`) to avoid conflicts with external tooling (Helm, Terraform, Mustache, ARM templates, etc.). Defaults to `{{` and `}}`.

---

## 2.2 Template Step

Each template step contains:

```
title
instructions (markdown)
position (for ordering)
```

Instructions may contain **variable placeholders** using the template's configured delimiters:

```
kubectl apply -f {{manifestFile}}
```

Placeholders support **pipe transforms** for inline formatting:

```
kubectl apply -f {{manifestFile | lower}}
docker build -t {{serviceName | kebab_case}}:{{version | replace(".", "_")}} .
```

---

## 2.3 Instance (Run)

An instance represents **a single execution of a template**.

Example:

```
Deploy credit-domain v1.5
```

An instance contains:

```
variables (JSON)
copied steps
execution state
pointer to next step
```

---

## 2.4 Instance Step

Instance steps are **copied from template steps** when the run begins.

Each step stores:

```
rendered instructions
completion status
timestamp
notes
```

Instance steps are **immutable with respect to template changes**. Variable substitution and pipe transforms are applied at run creation time; rendered instructions are stored as-is.

---

## 2.5 Todo

Standalone tasks not associated with templates.

Example:

```
Upgrade IntelliJ
Write Kafka article
Fix CI pipeline
```

---

# 3. System Architecture

Stack:

Frontend

```
Angular SPA
```

Backend

```
NestJS REST API (TypeScript)
```

Database

```
SQLite (TypeORM + migrations)
```

Deployment

```
Docker Compose or local dev
```

---

# 4. Database Design

## 4.1 Template Table

```sql
CREATE TABLE template (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    description TEXT,
    variable_prefix TEXT NULL,
    variable_suffix TEXT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME
);
```

`variable_prefix` and `variable_suffix` are both nullable. When null, the system defaults to `{{` and `}}`. Both columns must be set together or both left null.

---

## 4.2 Template Step Table

Uses **gap ordering** for drag-and-drop.

```sql
CREATE TABLE template_step (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    template_id INTEGER NOT NULL,
    position INTEGER NOT NULL,
    title TEXT NOT NULL,
    instructions TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY(template_id) REFERENCES template(id)
);
```

Example positions:

```
100
200
300
400
```

---

## 4.3 Instance Table

Stores run metadata and variables.

```sql
CREATE TABLE instance (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    template_id INTEGER NOT NULL,
    name TEXT NOT NULL,

    variables TEXT,

    status TEXT DEFAULT 'IN_PROGRESS',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

    next_step_id INTEGER,

    FOREIGN KEY(template_id) REFERENCES template(id)
);
```

Example variables:

```
{
  "serviceName": "credit-domain",
  "version": "1.5.2"
}
```

---

## 4.4 Instance Step Table

Stores the **snapshot of rendered instructions** (with all variables substituted and transforms applied).

```sql
CREATE TABLE instance_step (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    instance_id INTEGER NOT NULL,

    step_order INTEGER NOT NULL,
    title TEXT NOT NULL,

    instructions_template TEXT,
    rendered_instructions TEXT,

    completed BOOLEAN DEFAULT FALSE,
    completed_at DATETIME,
    notes TEXT,

    FOREIGN KEY(instance_id) REFERENCES instance(id)
);
```

---

## 4.5 Todo Table

```sql
CREATE TABLE todo (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    description TEXT,

    completed BOOLEAN DEFAULT FALSE,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    completed_at DATETIME
);
```

---

# 5. Template Variable System

## 5.1 Placeholders

Templates may contain variable placeholders. The delimiter characters are configurable per template. By default:

```
{{variableName}}
```

With custom delimiters (e.g. `@{` / `}`):

```
@{variableName}
```

Custom delimiters allow templates to coexist with external tooling that already uses `{{...}}` (Helm charts, Mustache templates, Terraform, ARM templates, etc.).

---

## 5.2 Pipe Transforms

Placeholders support **pipe-based transforms** applied left-to-right at render time:

```
{{title | upper | remove_spaces}}
{{version | replace(".", "_")}}
{{optional | default("N/A")}}
```

Syntax: `{{variableName | transform1 | transform2(arg1, arg2)}}`

- The variable name is the first segment (before the first `|`)
- Transforms are applied sequentially; each receives the output of the previous
- An unknown transform name leaves the full original placeholder intact (fail-safe)
- A missing variable with no `default` transform in the chain leaves the placeholder intact

### Built-in Transforms

| Category | Transforms |
|---|---|
| Case | `upper`, `lower`, `capitalize` (first letter), `title_case` (each word) |
| Whitespace | `trim`, `remove_spaces`, `collapse_spaces` |
| Naming | `snake_case`, `kebab_case`, `camel_case`, `pascal_case` |
| Substitution | `replace(search, replacement)` — all occurrences |
| Fallback | `default(fallbackValue)` — used when variable is empty or missing |
| Truncation | `truncate(maxLength)`, `pad_left(length, char)`, `pad_right(length, char)` |

---

## 5.3 Instance Creation Flow

1. Load template (including `variablePrefix`, `variableSuffix`)
2. Build regex dynamically from escaped delimiters
3. Scan step instructions to extract placeholder expressions
4. Strip pipe chains — extract only the **base variable name** for the form (e.g. `title | upper` → `title`)
5. Deduplicate variable names
6. Generate dynamic form from variable names
7. Store submitted values in `instance.variables`
8. Render instructions: for each placeholder, evaluate the expression — resolve the variable value, apply any pipe transforms in order
9. Store rendered output in `instance_step.rendered_instructions`

---

## 5.4 Delimiter Validation Rules

- Both `variablePrefix` and `variableSuffix` must be provided together, or both omitted
- Neither may be empty when provided
- Maximum 10 characters each
- Prefix and suffix must not be identical to each other
- All regex-special characters in delimiters are escaped before building the extraction/rendering regex

---

# 6. API Specification

Base path:

```
/api
```

---

## 6.1 Templates

### List Templates

```
GET /api/templates
```

Response

```json
[
  {
    "id": 1,
    "name": "Deployment Process",
    "description": "Standard deploy runbook",
    "variablePrefix": null,
    "variableSuffix": null,
    "stepCount": 6
  }
]
```

`variablePrefix` and `variableSuffix` are always returned (null when using defaults).

---

### Get Template

```
GET /api/templates/{id}
```

---

### Create Template

```
POST /api/templates
```

Body

```json
{
  "name": "Helm Deploy",
  "description": "Deploy using Helm (avoid {{ }} collision)",
  "variablePrefix": "@{",
  "variableSuffix": "}"
}
```

`variablePrefix` and `variableSuffix` are optional. Omit or set to `null` for defaults.

---

### Update Template

```
PUT /api/templates/{id}
```

Body fields are the same as Create. To revert to default delimiters, send `variablePrefix: null, variableSuffix: null`.

---

### Delete Template

```
DELETE /api/templates/{id}
```

---

## 6.2 Template Steps

### Add Step

```
POST /api/templates/{id}/steps
```

---

### Update Step

```
PUT /api/templates/{templateId}/steps/{stepId}
```

---

### Delete Step

```
DELETE /api/templates/{templateId}/steps/{stepId}
```

---

### Move Step

```
PATCH /api/templates/{templateId}/steps/{stepId}/move
```

Body

```json
{
  "beforeStepId": 12,
  "afterStepId": 13
}
```

Backend recalculates `position`.

---

## 6.3 Instances

### List Runs

```
GET /api/instances
```

Response includes:

```
next step
progress
```

---

### Create Instance

```
POST /api/instances
```

Body

```json
{
  "templateId": 1,
  "name": "Deploy credit-domain v1.5",
  "variables": {
    "serviceName": "credit-domain",
    "version": "1.5.2"
  }
}
```

Variable names in the body must match the **base variable names** extracted from the template steps (pipe transform suffixes are not part of the variable name).

---

### Get Instance

```
GET /api/instances/{id}
```

Returns steps with `renderedInstructions` already substituted and transforms already applied.

---

## 6.4 Step Completion

```
PATCH /api/instances/{instanceId}/steps/{stepId}
```

Body

```json
{
  "completed": true
}
```

Updates:

```
instance_step.completed
instance.next_step_id
```

---

## 6.5 Todos

### List Todos

```
GET /api/todos
```

---

### Create Todo

```
POST /api/todos
```

---

### Update Todo

```
PATCH /api/todos/{id}
```

---

### Delete Todo

```
DELETE /api/todos/{id}
```

---

# 7. UX Design

## 7.1 Navigation

```
Today
Runs
Templates
Todos
```

Default page:

```
Today
```

---

## 7.2 Today Dashboard (Guided System)

Displays:

```
Next step of each runbook
+
Todos
```

Example:

```
Today
--------------------------------

📋 Deploy credit-domain v1.5
Step 3 of 7

Build artifact

mvn clean package

[Copy] [Complete] [Open]

--------------------------------

☐ Upgrade IntelliJ
☐ Write Kafka article
```

---

## 7.3 Runbook Execution Screen

Layout:

```
Runbook Header
Progress bar

Step list
One step expanded
```

Example:

```
✔ Pull latest develop
✔ Run tests
▶ Build artifact
Deploy staging
Smoke tests
```

Expanded step:

```
Build artifact

mvn clean package

[Copy]
[Mark Complete]
```

---

## 7.4 Template List

```
Templates

Deployment Process
Sprint Ticket Lifecycle
Incident Runbook

[Create Template]
```

---

## 7.5 Template Editor

```
Template Name
Description

▼ Variable Delimiters (Advanced)
  Prefix: [ @{  ]   Suffix: [ }  ]
  (Leave blank to use defaults: {{ and }})

Steps

☰ Pull latest develop
☰ Run tests
☰ Build artifact

[Add Step]
```

The "Variable Delimiters" section is collapsible. Empty inputs mean default `{{`/`}}`.

---

## 7.6 Step Editor

```
Title

Markdown Instructions

Live Preview
```

Supports:

```
headers
lists
links
code blocks
```

---

## 7.7 Todo Screen

```
Master Todo

[Add Todo...]

[ ] Upgrade IntelliJ
[ ] Write Kafka article
[✔] Fix CI pipeline
```

---

# 8. Markdown Rendering

Markdown stored in step instructions.

Use frontend library:

```
ngx-markdown
```

Code blocks render with syntax highlighting.

Each code block includes:

```
[Copy]
```

---

# 9. Keyboard Shortcuts

Optional but recommended.

```
SPACE → mark step complete
N → next step
P → previous step
C → copy command
```

---

# 10. Performance Considerations

Dashboard optimized via:

```
instance.next_step_id
```

Query:

```sql
SELECT i.id, i.name, s.title, s.rendered_instructions
FROM instance i
LEFT JOIN instance_step s
ON s.id = i.next_step_id;
```

---

# 11. Key Implementation Files

### Backend (`checklist-execution-system-api`)

| File | Purpose |
|---|---|
| `src/template/template.entity.ts` | `variablePrefix`, `variableSuffix` columns |
| `src/template/dto/create-template.dto.ts` | Optional delimiter fields + both-or-neither validation |
| `src/template/dto/update-template.dto.ts` | Same as create DTO |
| `src/template/templates.service.ts` | Persists and returns delimiter fields |
| `src/instance/variable-extraction.service.ts` | Dynamic regex from delimiters; strips pipe expressions |
| `src/instance/instances.service.ts` | Reads template delimiters; calls transform pipeline |
| `src/instance/transform-registry.ts` | Registry of built-in transform functions |
| `src/instance/transform-pipeline.ts` | Parses `varName \| fn1 \| fn2(arg)` and evaluates chain |
| `src/common/escape-regex.ts` | Escapes regex metacharacters in delimiter strings |
| `src/database/migrations/` | Includes delimiter column migration |

### Frontend (`checklist-execution-system-ui`)

| File | Purpose |
|---|---|
| `src/app/models/api.models.ts` | `variablePrefix`/`variableSuffix` on Template and DTOs |
| `src/app/pages/templates/template-editor/` | Collapsible delimiter configuration UI |
| `src/app/pages/runs/start-run/start-run.component.ts` | `extractVariables()` with dynamic delimiters and pipe stripping |

---

# 12. Final System Summary

The system consists of:

```
5 database tables (template has 2 additional columns vs initial spec)
11 REST endpoints
4 UI pages
Guided dashboard workflow
Template-based runbooks with variables
  - Configurable delimiters per template (default: {{ / }})
  - Pipe transforms applied at render time (upper, lower, replace, default, etc.)
Standalone todos
```

Design goals achieved:

```
fast execution
minimal complexity
strong extensibility
excellent developer usability
backward compatible (existing templates with default delimiters unchanged)
```

---

# 13. Future Enhancements

Not included but supported by architecture:

```
template versioning
step dependencies
automation hooks
scheduled runbooks
variable validation rules
runbook sharing
attachments
metrics and history
user-defined transforms (Tier 3 — sandboxed JS via isolated-vm)
step editor hint showing effective delimiter pattern
transform discoverability (help tooltip / docs link in step editor)
```
