---
name: "Task From Phase Plan"
description: "Generate a GitHub issue task from a phase or task reference in the full-spec-plan. Use when you have a phased task like 3.3 or Phase 8 Task 8.3 and want issue-ready markdown."
argument-hint: "Phase/task reference, for example: 3.3, Phase 3 Task 3.3, or 8.3 Run execution screen"
agent: "Plan"
---

Generate one issue-ready task from the project planning documents.

Primary source:
- [full-spec-plan](../../docs/full-spec-plan.md)

Supporting source:
- [full-spec](../../docs/full-spec.md)

Your job:
1. Find the referenced phase/task in [full-spec-plan](../../docs/full-spec-plan.md).
2. Use [full-spec](../../docs/full-spec.md) only to clarify requirements, behavior, data model details, API expectations, or UX expectations for that same task.
3. Produce a single GitHub-issue-style task in the exact structure below.
4. Keep the task tightly scoped to one functional unit.
5. Preserve the original phase/task numbering from the plan.
6. Do not invent architecture, dependencies, deliverables, or acceptance criteria that are not supported by the planning docs.
7. If the requested task reference is ambiguous or missing, ask a short clarifying question instead of drafting the wrong issue.

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
- Prefer the task wording already present in [full-spec-plan](../../docs/full-spec-plan.md) when possible.
- When dependencies are listed in the phase table, carry them into `Blocked by`.
- When the plan is high-level, make the issue more actionable without broadening scope.
- When backend or frontend testing is relevant, reflect the project verification strategy from the planning docs.
- Do not merge multiple adjacent tasks into one issue.

Example inputs:
- `3.3`
- `Phase 3 Task 3.3`
- `2.5 Step reordering (move)`
- `8.3 Run execution screen`