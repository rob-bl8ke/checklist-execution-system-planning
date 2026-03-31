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

---

## 2.2 Template Step

Each template step contains:

```
title
instructions (markdown)
position (for ordering)
```

Instructions may contain **variables**:

```
kubectl apply -f {{manifestFile}}
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

Instance steps are **immutable with respect to template changes**.

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

Recommended stack (but implementation-agnostic):

Frontend

```
Angular SPA
```

Backend

```
Spring Boot REST API
```

Database

```
SQLite
```

Deployment

```
Single container or local run
```

---

# 4. Database Design

## 4.1 Template Table

```sql
CREATE TABLE template (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    description TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME
);
```

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

Stores the **snapshot of rendered instructions**.

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

Templates may contain placeholders:

```
{{variableName}}
```

Example:

```
docker build -t {{serviceName}}:{{version}} .
```

## Instance Creation Flow

1. Load template steps
2. Extract placeholders using regex

```
{{(.*?)}} 
```

3. Deduplicate variable names
4. Generate dynamic form
5. Store values in `instance.variables`
6. Render instructions
7. Store rendered instructions in `instance_step.rendered_instructions`

---

# 6. API Specification

Base path:

```
/api
```

---

# 6.1 Templates

### List Templates

```
GET /api/templates
```

Response

```
[
  {
    "id":1,
    "name":"Deployment Process",
    "stepCount":6
  }
]
```

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

```
{
  "name":"Deployment Process"
}
```

---

### Update Template

```
PUT /api/templates/{id}
```

---

### Delete Template

```
DELETE /api/templates/{id}
```

---

# 6.2 Template Steps

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

```
{
  "beforeStepId":12,
  "afterStepId":13
}
```

Backend recalculates `position`.

---

# 6.3 Instances

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

```
{
  "templateId":1,
  "name":"Deploy credit-domain v1.5",
  "variables":{
     "serviceName":"credit-domain",
     "version":"1.5.2"
  }
}
```

---

### Get Instance

```
GET /api/instances/{id}
```

Returns steps.

---

# 6.4 Step Completion

```
PATCH /api/instances/{instanceId}/steps/{stepId}
```

Body

```
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

# 6.5 Todos

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

# 7.2 Today Dashboard (Guided System)

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

# 7.3 Runbook Execution Screen

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

# 7.4 Template List

```
Templates

Deployment Process
Sprint Ticket Lifecycle
Incident Runbook

[Create Template]
```

---

# 7.5 Template Editor

```
Template Name
Description

Steps

☰ Pull latest develop
☰ Run tests
☰ Build artifact

[Add Step]
```

Click step to edit.

---

# 7.6 Step Editor

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

# 7.7 Todo Screen

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

# 11. Future Enhancements

Not included in V1 but supported by architecture.

Possible additions:

```
template versioning
step dependencies
automation hooks
scheduled runbooks
variable validation
runbook sharing
attachments
metrics and history
```

---

# 12. Final System Summary

The system consists of:

```
5 database tables
11 REST endpoints
4 UI pages
Guided dashboard workflow
Template-based runbooks with variables
Standalone todos
```

Design goals achieved:

```
fast execution
minimal complexity
strong extensibility
excellent developer usability
```
