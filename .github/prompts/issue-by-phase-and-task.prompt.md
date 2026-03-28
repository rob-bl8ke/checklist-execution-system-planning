---
name: issue-by-phase-and-task
description: Generate and create GitHub issues from one or more phase/task references in a supplied plan file. Use when you have a phased task like 3.3, a range like 3.1-3.4, a list like 3.1, 3.3, 4.2, or a whole phase and want structured GitHub issues drafted, reviewed, and created directly from your plan.
argument-hint: Task or phase selection, for example, 3.3, Phase 3, 3.1-3.4, 3.1, 3.3, 4.2, or 8.3 Run execution screen. Add the plan file first, and add a supporting spec file when task clarification requires it.
---

Generate and create one GitHub issue per task from the project planning documents.

### Source selection:
- Prefer a plan document explicitly provided in chat context by the user.
- Prefer a supporting spec document explicitly provided in chat context by the user.
- If no plan document is provided, ask the user to supply it before drafting output.
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
3. Analyze the active plan document first so the selected item is properly understood before drafting output.
4. Use the active supporting spec document only to clarify requirements, behavior, data model details, API expectations, or UX expectations for that same selected item.
5. For each item in the normalized processing list, in order:
   a. Generate a complete GitHub issue draft using the structure defined below.
   b. Present the draft to the user for review.
   c. Ask for explicit approval: "Does this look correct? Approve to create this issue on GitHub, or let me know what to change."
   d. Do not create the issue until the user gives explicit approval. If the user requests changes, apply them and re-present the updated draft before asking again.
   e. Once approved, create the issue using the GitHub MCP server if it is available. If it is not available, fall back to the `gh` CLI following the GitHub CLI fallback notes and PowerShell command pattern below. If the target repository has not been established in the session, ask the user for it before creating. If a GitHub project has not been established in the session, ask the user whether issues should be added to a project and for the project details before creating.
   f. After the issue is successfully created, report the issue number and URL, then proceed to the next item.
6. Keep each generated item tightly scoped to one functional unit.
7. Preserve the original phase/task numbering from the plan.
8. Do not invent architecture, dependencies, deliverables, acceptance criteria, or diagrams that are not supported by the planning docs.
9. If the requested task reference, range, list, or phase is ambiguous, missing, or partially invalid, ask a short clarifying question instead of drafting the wrong output.

### Output rules:
- Output markdown only for each draft.
- Do not include implementation code.
- Keep the wording concrete and execution-focused.
- If GitHub issue numbers do not exist yet, use plan references in `## Dependencies/Blockers`, such as `2.3` or `Phase 2 Task 2.3`.
- Keep `## Notes and Risks` brief. Omit obvious filler.
- Process one item at a time: draft → verify → create → next.
- Only include a `## Visual Aids` section when a diagram would materially improve understanding of the selected item.

### GitHub issue creation:
- Prefer the GitHub MCP server to create issues when it is available.
- If the GitHub MCP server is not available, fall back to the `gh` CLI using the pattern described in the GitHub CLI fallback notes below.
- Set the issue title to the task title from the plan only, with no phase or task prefix: `{Title}`.
- Set the issue body to the full markdown content of the generated issue (all sections from `## Story` onward, excluding the `## Item` header block).
- After creation, report the issue number and URL before moving to the next item.
- If the user has indicated that issues should be added to a GitHub project, add each issue to the project after creation. Ask the user for the project number and owner if not already established.

### Repository routing:
- Ask the user which repository (or repositories) to target if not already established in the session.
- For multi-repo workspaces, ask the user which repo owns each category of task (e.g. frontend, backend, cross-cutting) before creating the first issue, so routing is consistent for the rest of the session.
- If the target repository for a specific task is ambiguous, ask a short clarifying question rather than guessing.

### GitHub CLI fallback notes:
- Use the `gh` CLI only when the GitHub MCP server is not available.
- When writing an issue body in PowerShell, do not pipe via stdin — it silently fails. Instead write the body to a temp file using `[System.IO.Path]::GetTempFileName() + ".md"` with `Set-Content -Encoding UTF8`, pass the path to `--body-file`, then delete the temp file.
- `gh issue create` may produce no stdout on success. Always verify with `gh issue list --repo {owner}/{repo} --limit 3` immediately after to confirm creation and obtain the issue number.
- After creation, always verify the issue body is correct before adding it to a project.

### PowerShell command pattern for gh CLI fallback:
```powershell
$body = @'
{full issue markdown body}
'@
$tmpFile = [System.IO.Path]::GetTempFileName() + ".md"
$body | Set-Content -Path $tmpFile -Encoding UTF8
gh issue create --repo {owner}/{repo} --title "{title}" --body-file $tmpFile 2>&1
Remove-Item $tmpFile

# Verify creation and get the issue number
gh issue list --repo {owner}/{repo} --limit 3 2>&1

# If adding to a project:
gh project item-add {project-number} --owner {project-owner} --url https://github.com/{owner}/{repo}/issues/{number} 2>&1
```

### Diagram rules:
- After drafting the issue body, decide whether a visual aid would materially reduce ambiguity for that generated item.
- If helpful, include one or more PlantUML diagrams using the most appropriate type: sequence, state, or activity.
- Cross-check the diagram syntax for correctness and clarity before outputting it.
- For each included diagram, provide:
	- a short heading that names the diagram
	- a fenced `plantuml` block containing the script
	- a short summary explaining what the diagram shows
- If no diagram adds meaningful clarity, omit the `## Visual Aids` section entirely.

Use this exact output structure for each generated GitHub issue draft:

~~~markdown
## Item
Phase / Task ID: {phase or task identifier}
Title: {phase or task title}

## Story
{Write as a user story grounded in the planning documents: "As a [role], I want [goal] so that [benefit]."}

## Short Description
{A concise paragraph summary of the work}

Linked to: [Phase {N} | Task {N.N}]

## Acceptance Criteria
- [ ] {Testable outcome}
- [ ] {Testable outcome}

## Dependencies/Blockers
- {Only include when supported and relevant. Use plan references like 2.3 or Phase 2 Task 2.3 if issue numbers do not exist yet.}

## Spec Foundation
- {Relevant requirement or decision from the planning docs}
- {Optional second supporting reference if needed}

## Inputs
- {Upstream task, spec section, or artifact}
- {Upstream task, spec section, or artifact}

## Scope Included
- {Required item}
- {Required item}

## Scope Excluded
- {Explicit non-goal}
- {Explicit non-goal}

## Deliverables
- {Code, config, doc, or test artifact}
- {Code, config, doc, or test artifact}

## Verification
- {Automated test, command, or validation step}
- {Manual check if needed}

## Notes and Risks
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
- When dependencies are listed in the phase table, carry them into `## Dependencies/Blockers`.
- When the plan is high-level, make the issue more actionable without broadening scope.
- When backend or frontend testing is relevant, reflect the project verification strategy from the active planning documents.
- Do not merge multiple adjacent tasks into one generated item.
- Omit the entire `## Dependencies/Blockers` section when there are no supported dependencies or blockers.
- Omit the entire `## Notes and Risks` section when there is nothing meaningful to add.
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
- Add a plan file to chat context, then run this prompt with `3.3`.
- Add a plan file to chat context, then run this prompt with `Phase 3` to expand into one generated item per task if that phase contains tasks.
- Add a plan file to chat context, then run this prompt with `3.1-3.4` to generate and create one issue per task in that range.
- Add both a plan file and supporting spec file to chat context, then run this prompt with `8.3 Run execution screen` when plan data alone is not enough to clarify the task.