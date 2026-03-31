# Current Specification — Checklist Execution System

> This document reflects the current agreed specification of the system, including implemented variable enhancement features and the approved reminder system design.

---

# 1. System Overview

The system is a **Runbook + Reminder + Todo manager** designed for engineers performing repeatable operational tasks.

Examples:

* Deploy services
* Incident response
* Release checklists
* CI/CD runbooks
* Personal engineering tasks
* Daily preparation reminders
* Weekly preparation reminders
* Sprint ceremony preparation

The system contains **four functional areas**:

```
Runbooks (Template-based processes)
Runs (Instances of templates)
Todos (Standalone one-off tasks)
Reminders (Recurring or one-time advance-notice schedules)
```

The **default screen is the "Today" dashboard**, which guides users through the **next step of each runbook** while also displaying reminder occurrences and todos.

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

Todos are **standalone one-off tasks** not associated with templates or reminder cadence.

Example:

```
Upgrade IntelliJ
Write Kafka article
Fix CI pipeline
```

Todos remain intentionally simple and do not carry recurring schedule semantics in v1.

---

## 2.6 Reminder Definition

A **reminder definition** is a reusable schedule rule for recurring or one-time reminders.

Examples:

```
Daily standup prep
Weekly update preparation
Sprint retrospective preparation
Sprint planning preparation
```

A reminder definition contains:

```
title
optional description
optional category
cadence
interval
anchor date
optional weekdays
optional time of day
lead time in days
optional linked template
active state
```

Reminder definitions are used for daily reminders, end-of-week reminders, and sprint ceremonies that should surface in advance.

---

## 2.7 Reminder Occurrence

A **reminder occurrence** is a computed occurrence of a reminder definition for a specific date.

Example:

```
Sprint retrospective
Occurs on 2026-04-03
Becomes visible on 2026-04-01 when leadTimeDays = 2
```

Occurrence state is applied per occurrence date:

```
OPEN
COMPLETED
DISMISSED
```

Occurrences are computed for a requested window. Only acted-on occurrences are persisted in the database.

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

Reminder visibility is computed on demand from reminder definitions plus occurrence state rather than from a background materialization job.

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

## 4.6 Reminder Definition Table

```sql
CREATE TABLE reminder_definition (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    description TEXT,
    category TEXT,
    cadence TEXT NOT NULL,
    interval INTEGER NOT NULL DEFAULT 1,
    anchor_date TEXT NOT NULL,
    weekdays TEXT,
    time_of_day TEXT,
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    linked_template_id INTEGER,
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME,

    FOREIGN KEY(linked_template_id) REFERENCES template(id)
);
```

Rules:

* `cadence` is one of `ONCE`, `DAILY`, or `WEEKLY`
* `interval` is used for daily or weekly frequency spacing; minimum `1`, maximum `52`
* `weekdays` is JSON text storing an array of weekday numbers `0-6`; use a TypeORM JSON transformer (serialize array to/from `TEXT`) matching the `variables` pattern in `instance.entity.ts`
* `anchor_date` is stored as `TEXT` in SQLite (`YYYY-MM-DD` format), typed as `string` in the entity, and validated as `@IsDateString()` at the DTO level; it is a local scheduling date, not a UTC timestamp
* `lead_time_days` controls how many days before the occurrence the reminder becomes visible; minimum `0`, maximum `365`
* `linked_template_id` is optional and provides a `Start run` entry point instead of automatic run creation
* `category` is free-text (max 50 chars) rather than an enum, allowing flexible filtering via distinct-value queries

---

## 4.7 Reminder Occurrence State Table

```sql
CREATE TABLE reminder_occurrence_state (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    reminder_id INTEGER NOT NULL,
    occurrence_date TEXT NOT NULL,
    status TEXT NOT NULL,
    acted_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY(reminder_id) REFERENCES reminder_definition(id) ON DELETE CASCADE,
    UNIQUE(reminder_id, occurrence_date)
);
```

Rules:

* `status` is one of `COMPLETED` or `DISMISSED`
* absence of a row means the occurrence is still `OPEN`
* only acted-on occurrences are stored
* `occurrence_date` is stored as `TEXT` (`YYYY-MM-DD`), typed as `string` in the entity
* `ON DELETE CASCADE` removes occurrence state rows when a reminder definition is deleted, matching the cascade pattern used for `template_step` and `instance_step`

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

* The variable name is the first segment (before the first `|`)
* Transforms are applied sequentially; each receives the output of the previous
* An unknown transform name leaves the full original placeholder intact (fail-safe)
* A missing variable with no `default` transform in the chain leaves the placeholder intact

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

* Both `variablePrefix` and `variableSuffix` must be provided together, or both omitted
* Neither may be empty when provided
* Maximum 10 characters each
* Prefix and suffix must not be identical to each other
* All regex-special characters in delimiters are escaped before building the extraction/rendering regex

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

Returns `409 Conflict` if any active reminder definitions reference the template via `linkedTemplateId`. The error message should identify the blocking reminders. This prevents silent reminder degradation.

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

## 6.6 Reminders

### List Reminder Definitions

```
GET /api/reminders
```

Query params:

```json
{
  "active": true,
  "linkedTemplateId": 7,
  "category": "SPRINT_RETRO"
}
```

Response items include the stored reminder definition fields plus derived fields such as `nextOccurrenceDate`, `nextPrepStartDate`, `lastCompletedOccurrenceDate`, `createdAt`, and `updatedAt`.

---

### Get Reminder Definition

```
GET /api/reminders/{id}
```

---

### Create Reminder Definition

```
POST /api/reminders
```

Body

```json
{
  "title": "Sprint retrospective",
  "description": "Prepare notes and discussion items",
  "category": "SPRINT_RETRO",
  "cadence": "WEEKLY",
  "interval": 2,
  "anchorDate": "2026-04-03",
  "weekdays": [5],
  "timeOfDay": null,
  "leadTimeDays": 2,
  "linkedTemplateId": 7
}
```

Supported request fields:

```
title
description?
category?
cadence
interval?
anchorDate
weekdays?
timeOfDay?
leadTimeDays?
linkedTemplateId?
```

---

### Update Reminder Definition

```
PUT /api/reminders/{id}
```

Body fields are the same as Create, plus optional `active`.

---

### Delete Reminder Definition

```
DELETE /api/reminders/{id}
```

Deletes the reminder definition and its occurrence state rows.

---

### Agenda Window

```
GET /api/reminders/agenda?from=YYYY-MM-DD&to=YYYY-MM-DD
```

Returns computed reminder occurrences for the requested date window.

Response items include:

```
reminderId
title
description
category
occurrenceDate
prepStartDate
timeOfDay
status
isInPrepWindow
isOverdue
daysUntilOccurrence
linkedTemplate
canStartRun
```

Derived fields (`isInPrepWindow`, `isOverdue`, `daysUntilOccurrence`) are computed on the backend based on the server's current date at response time.

---

### Update Reminder Occurrence State

```
PATCH /api/reminders/{id}/occurrences/{occurrenceDate}
```

Body

```json
{
  "status": "COMPLETED"
}
```

Supported occurrence state values:

```
COMPLETED
DISMISSED
OPEN
```

`OPEN` removes any persisted occurrence-state row and restores the default derived state.

---

## 6.7 Dashboard

### Get Today Dashboard

```
GET /api/dashboard?upcomingDays=7
```

`upcomingDays` is optional (default `7`, minimum `1`, maximum `30`). It controls only the reminder upcoming window; runs and todos remain unfiltered.

Response includes:

```
runs
todos
reminders.dueNow
reminders.upcoming
```

Semantics:

* `dueNow` contains open reminder occurrences where `today >= prepStartDate` (i.e. the prep window has started) and the occurrence is still `OPEN`
* `upcoming` contains future reminder occurrences where `today < prepStartDate` and `occurrenceDate <= today + upcomingDays`
* `prepStartDate` is always `occurrenceDate - leadTimeDays`
* When `upcomingDays` is `0`, the `upcoming` array is empty

---

# 7. UX Design

## 7.1 Navigation

```
Today
Runs
Templates
Todos
Reminders
```

Default page:

```
Today
```

---

## 7.2 Today Dashboard (Guided System)

Displays:

```
Reminder occurrences ready now
+
Reminder occurrences coming up soon
+
Next step of each active runbook
+
Todos
```

Example:

```
Today
--------------------------------

🔔 Sprint retrospective
Occurs Friday
Visible now because lead time is 2 days

[Done] [Dismiss] [Start run]

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

Reminder occurrences are shown as occurrence-based items rather than raw schedule definitions.

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

Todos remain one-off tasks and do not double as reminder definitions.

---

## 7.8 Reminders Screen

The Reminders screen manages recurring and one-time reminder definitions.

Layout:

```
Reminder filters / summary row

Reminder list
```

Each reminder shows:

```
title
cadence summary
lead time
next occurrence
optional linked template
active state
```

Primary actions:

```
Create reminder
Edit
Deactivate / Activate
Delete
```

The reminder editor includes:

```
Basics
Schedule
Visibility
Optional runbook link
Occurrence preview
```

Ceremony presets such as backlog refinement, retrospective, and sprint planning may prefill reminder definitions but still remain editable before save.

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

Dashboard run queries remain optimized via:

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

Reminder visibility is optimized by:

```
active reminder definitions
bounded agenda windows
occurrence-state overlay on computed occurrences
```

Reminder occurrence computation should be performed only for the requested window, with reasonable bounds such as 7 to 30 days for the dashboard and up to 90 days for agenda queries. The recurrence engine must short-circuit once it exceeds the window end date rather than precomputing all possible occurrences then filtering.

The `lastCompletedOccurrenceDate` derived field on reminder list responses should be computed with a single query using `MAX(occurrence_date) WHERE status = 'COMPLETED' GROUP BY reminder_id` to avoid N+1 lookups.

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
| `src/dashboard/dashboard.service.ts` | Aggregates runs, todos, and reminder occurrences for Today |
| `src/reminder/reminder-definition.entity.ts` | Reminder definition entity with schedule columns |
| `src/reminder/reminder-occurrence-state.entity.ts` | Per-occurrence state entity (COMPLETED/DISMISSED) |
| `src/reminder/reminders.controller.ts` | Reminder CRUD, agenda, and occurrence state endpoints |
| `src/reminder/reminders.service.ts` | Reminder CRUD, occurrence state, and derived field computation |
| `src/reminder/recurrence.utils.ts` | Pure functions for computing occurrence dates (no DI); independently unit-testable |
| `src/reminder/dto/` | CreateReminderDto, UpdateReminderDto, UpdateReminderOccurrenceDto |
| `src/reminder/enums/` | ReminderCadence, ReminderOccurrenceStatus enums |
| `src/reminder/` | Reminder definition CRUD, recurrence logic, and occurrence state handling |
| `src/database/migrations/` | Includes delimiter and reminder schema migrations |

### Frontend (`checklist-execution-system-ui`)

| File | Purpose |
|---|---|
| `src/app/models/api.models.ts` | Template, dashboard, and reminder API models |
| `src/app/pages/templates/template-editor/` | Collapsible delimiter configuration UI |
| `src/app/pages/runs/start-run/start-run.component.ts` | `extractVariables()` with dynamic delimiters and pipe stripping |
| `src/app/pages/today/today.component.ts` | Displays runs, todos, and reminder occurrences |
| `src/app/pages/reminders/` | Reminder management list and editor flows |
| `src/app/services/reminders-api.service.ts` | HTTP client for reminder CRUD, agenda, and occurrence actions |
| `src/app/components/nav/nav.component.ts` | Navigation entry for Reminders |

---

# 12. Final System Summary

The system consists of:

```
9 database tables
32 REST endpoints
6 UI pages
Guided dashboard workflow
Template-based runbooks with variables
  - Configurable delimiters per template (default: {{ / }})
  - Pipe transforms applied at render time (upper, lower, replace, default, etc.)
Standalone todos
Recurring and one-time reminders with advance notice
Optional reminder links to templates for runbook-based preparation workflows
```

Design goals achieved:

```
fast execution
minimal complexity
strong extensibility
excellent developer usability
backward compatible template variable behavior
clear separation between one-off todos and recurring reminders
```

Reminder delivery in v1 is inside the app. Reminder occurrences do not auto-create run instances; linked templates surface a `Start run` action instead.

---

# 13. Future Enhancements

Not included but supported by architecture:

```
external notifications
timezone-aware reminder delivery
calendar sync
sprint aggregate with generated ceremony reminders
template versioning
step dependencies
automation hooks
scheduled runbooks with automatic instance creation
variable validation rules
runbook sharing
attachments
metrics and history
user-defined transforms (Tier 3 — sandboxed JS via isolated-vm)
step editor hint showing effective delimiter pattern
transform discoverability (help tooltip / docs link in step editor)
```
