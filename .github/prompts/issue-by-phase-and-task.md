---
name: issue-by-phase-and-task
description: Generate Jira-ready issue content and a task specification from one or more phase/task references in a selected spec plan. Use when you have a phased task like 3.3, a range like 3.1-3.4, a list like 3.1, 3.3, 4.2, or a whole phase and want structured issue-ready markdown.
argument-hint: Task or phase selection, for example, 3.3, Phase 3, 3.1-3.4, 3.1, 3.3, 4.2, or 8.3 Run execution screen. Add the spec plan first, and add a supporting spec file when task clarification requires it.
---

Generate one or more issue-ready task packages from the project planning documents.

### Source selection:
- Prefer a spec plan document explicitly provided in chat context by the user.
- Prefer a supporting spec document explicitly provided in chat context by the user.
- If no spec plan document is provided, ask the user to supply it before drafting output.
- If a supporting spec document is needed to clarify requirements, behavior, data model details, API expectations, or UX expectations and none has been provided, ask the user to supply it before drafting output.

### Selection rules:
- Accept a single task reference, a contiguous task range, a comma-separated task list, or a whole phase.
- Normalize the requested selection into a numerically ordered processing list.
- If the user selects a phase that contains tasks, expand the phase into separate generated items for each task in that phase.
- If the user selects a phase that does not contain tasks, treat the phase itself as the generated item.
- If any requested phase or task cannot be resolved unambiguously from the active plan, ask a short clarifying question instead of guessing.

### Your job:
1. Determine the active plan document using the source selection rules above.
2. Parse the requested selection and resolve every referenced phase/task from the active plan document.
3. Analyze the active spec plan first so the selected item is properly understood before drafting output.
4. Use the active supporting spec document only to clarify requirements, behavior, data model details, API expectations, or UX expectations for that same selected item.
5. Generate one complete output package per selected item in the same numeric order as the normalized processing list.
6. Keep each generated item tightly scoped to one functional unit.
7. Preserve the original phase/task numbering from the plan.
8. Do not invent architecture, dependencies, deliverables, acceptance criteria, or diagrams that are not supported by the planning docs.
9. If the requested task reference, range, list, or phase is ambiguous, missing, or partially invalid, ask a short clarifying question instead of drafting the wrong output.

### Output rules:
- Output markdown only.
- Do not include implementation code.
- Keep the wording concrete and execution-focused.
- If Jira issue numbers do not exist yet, use plan references in `Blocked by`, such as `2.3` or `Phase 2 Task 2.3`.
- Keep `Notes / Risks` brief. Omit obvious filler.
- For multiple generated items, keep the output in numeric order and clearly separate each item.
- For each generated item, always output `# Jira Issue` first and `# Task Specification` second.
- Only include a diagram section when a sequence, state, or activity diagram would materially improve understanding of the selected item.

### Jira Issue section requirements:
- Write the `# Jira Issue` section using the structure and field intent from [define-tasks/from-spec](./define-tasks/from-spec.md).
- Use this exact field order inside `# Jira Issue`:
	- `### Task Title`
	- `#### 1. Story (Business Language)`
	- `#### 2. Short Description`
	- `#### 3. Estimation Points`
	- `#### 4. Acceptance Criteria`
	- `#### 5. Dependencies/Blockers` only when there are real dependencies or blockers supported by the planning documents
- Keep the Jira Issue section suitable for pasting directly into a Jira ticket.

### Task Specification section requirements:
- Write the `# Task Specification` section using the exact structure already defined below.
- Keep the task wording aligned with the active plan document when possible.
- Do not merge multiple adjacent tasks into one generated item.

### Diagram rules:
- After drafting `# Jira Issue` and `# Task Specification`, decide whether a visual aid would materially reduce ambiguity for that generated item.
- If helpful, include one or more PlantUML diagrams using the most appropriate type: sequence, state, or activity.
- Cross-check the diagram syntax for correctness and clarity before outputting it.
- For each included diagram, provide:
	- a short heading that names the diagram
	- a fenced `plantuml` block containing the script
	- a short summary explaining what the diagram shows
- If no diagram adds meaningful clarity, omit the diagram section entirely.

Use this exact output structure for each generated item:

~~~markdown
## Item
Phase / Task ID: {phase or task identifier}
Title: {phase or task title}

# Jira Issue
### Task Title

#### 1. Story (Business Language)
{Write as a user story grounded in the planning documents}

#### 2. Short Description
{A concise paragraph summary of the work}

#### 3. Estimation Points
1. {Concrete factor that affects effort}
2. {Concrete factor that affects effort}

#### 4. Acceptance Criteria
- [ ] {Testable outcome}
- [ ] {Testable outcome}

#### 5. Dependencies/Blockers
- {Only include when supported and relevant}

# Task Specification
## Title:
Phase / Task ID:

Blocked by:

#### Goal
{2-4 sentences describing the outcome and why it exists}

#### Spec Foundation
- {Relevant requirement or decision from the planning docs}
- {Optional second supporting reference if needed}

#### Inputs
- {Upstream task, spec section, or artifact}
- {Upstream task, spec section, or artifact}

#### Scope Included
- {Required item}
- {Required item}
- {Required item}

#### Scope Excluded
- {Explicit non-goal}
- {Explicit non-goal}

#### Deliverables
- {Code, config, doc, or test artifact}
- {Code, config, doc, or test artifact}

#### Acceptance Criteria
- {Observable done condition}
- {Observable done condition}
- {Observable done condition}

#### Verification
- {Automated test, command, or validation step}
- {Manual check if needed}

#### Notes / Risks
- {Optional edge case, ambiguity, or follow-up concern}

## Visual Aids
### {Diagram name}
```plantuml
{PlantUML script only when a diagram is materially useful}
```
Summary: {Short explanation of the diagram}
~~~

#### Additional guidance:
- Prefer the task wording already present in the active plan document when possible.
- When dependencies are listed in the phase table, carry them into `Blocked by`.
- When the plan is high-level, make the issue more actionable without broadening scope.
- When backend or frontend testing is relevant, reflect the project verification strategy from the active planning documents.
- Do not merge multiple adjacent tasks into one generated item.
- Omit the entire `#### 5. Dependencies/Blockers` subsection from `# Jira Issue` when there are no supported dependencies or blockers.
- Omit the entire `## Visual Aids` section when no diagram is needed.

#### Example inputs:
- `3.3`
- `Phase 3`
- `Phase 3 Task 3.3`
- `3.1-3.4`
- `3.1, 3.3, 4.2`
- `2.5 Step reordering (move)`
- `8.3 Run execution screen`

#### Example usage:
- Add a spec plan file to chat context, then run this prompt with `3.3`.
- Add a spec plan file to chat context, then run this prompt with `Phase 3` to expand into one generated item per task if that phase contains tasks.
- Add a spec plan file to chat context, then run this prompt with `3.1-3.4` to generate one output package per task in that range.
- Add both a spec plan file and supporting spec file to chat context, then run this prompt with `8.3 Run execution screen` when plan data alone is not enough to clarify the task.