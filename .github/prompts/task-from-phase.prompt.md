---
name: "Task From Phase Plan"
description: "Generate a GitHub issue task from a phase or task reference in a selected spec plan. Use when you have a phased task like 3.3 or Phase 8 Task 8.3 and want issue-ready markdown."
argument-hint: "Task reference, for example: 3.3 or 8.3 Run execution screen. Optionally add the plan/spec files to chat context first."
agent: "Plan"
---

Generate one issue-ready task from the project planning documents.

Source selection:
- Prefer a spec plan document explicitly provided in chat context by the user.
- Prefer a supporting spec document explicitly provided in chat context by the user.
- If no plan document is provided, fall back to [spec-plan](../../docs/spec-plan.md).
- If no supporting spec document is provided, fall back to [full-spec](../../docs/full-spec.md).

Your job:
1. Determine the active plan document using the source selection rules above.
2. Find the referenced phase/task in the active plan document.
3. Use the active supporting spec document only to clarify requirements, behavior, data model details, API expectations, or UX expectations for that same task.
4. Produce a single GitHub-issue-style task in the exact structure below.
5. Keep the task tightly scoped to one functional unit.
6. Preserve the original phase/task numbering from the plan.
7. Do not invent architecture, dependencies, deliverables, or acceptance criteria that are not supported by the planning docs.
8. If the requested task reference is ambiguous or missing, ask a short clarifying question instead of drafting the wrong issue.

Output rules:
- Output markdown only.
- Do not include implementation code.
- Keep the wording concrete and execution-focused.
- If GitHub issue numbers do not exist yet, use plan references in `Blocked by`, such as `2.3` or `Phase 2 Task 2.3`.
- Infer sensible labels from the task domain and phase, for example:
  - `backend-templates`
  - `backend-instances`
  - `backend-todos`
  - `frontend-shell`
  - `frontend-templates`
  - `frontend-runs`
  - `frontend-todos`
  - `frontend-dashboard`
  - `infrastructure`
  - `phase-3`
- Keep `Notes / Risks` brief. Omit obvious filler.

Use this exact output structure:

Title:
Phase / Task ID:

Labels:
Blocked by:

Goal
{2-4 sentences describing the outcome and why it exists}

Spec Foundation
- {Relevant requirement or decision from the planning docs}
- {Optional second supporting reference if needed}

Inputs
- {Upstream task, spec section, or artifact}
- {Upstream task, spec section, or artifact}

Scope Included
- {Required item}
- {Required item}
- {Required item}

Scope Excluded
- {Explicit non-goal}
- {Explicit non-goal}

Deliverables
- {Code, config, doc, or test artifact}
- {Code, config, doc, or test artifact}

Acceptance Criteria
- {Observable done condition}
- {Observable done condition}
- {Observable done condition}

Verification
- {Automated test, command, or validation step}
- {Manual check if needed}

Notes / Risks
- {Optional edge case, ambiguity, or follow-up concern}

Additional guidance:
- Prefer the task wording already present in the active plan document when possible.
- When dependencies are listed in the phase table, carry them into `Blocked by`.
- When the plan is high-level, make the issue more actionable without broadening scope.
- When backend or frontend testing is relevant, reflect the project verification strategy from the active planning documents.
- Do not merge multiple adjacent tasks into one issue.

Example inputs:
- `3.3`
- `Phase 3 Task 3.3`
- `2.5 Step reordering (move)`
- `8.3 Run execution screen`

Example usage:
- Add a plan file to chat context, then run this prompt with `3.3`.
- Add both a plan file and supporting spec file to chat context, then run this prompt with `8.3 Run execution screen`.