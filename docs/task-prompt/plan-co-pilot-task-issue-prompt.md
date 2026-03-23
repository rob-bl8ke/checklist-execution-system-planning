## Plan: Copilot Task Issue Prompt

Create a workspace-scoped Copilot prompt file that converts a selected phased task from the project planning documents into issue-ready markdown. The prompt should live in `.github/prompts/`, be discoverable from chat, and guide Copilot to use the planning docs as source-of-truth while producing the simplified task template agreed in this discussion.

**Steps**
1. Create a workspace prompt file in `.github/prompts/` rather than a user-profile prompt, because the workflow is project-specific and should travel with the planning repository.
2. Use a `.prompt.md` file, not an instructions file or skill, because this is a single focused task: generate one GitHub issue body from a referenced phase/task in the plan.
3. Configure frontmatter with a clear `name`, a strong `description` containing trigger phrases like `task`, `issue`, `phase`, and `full-spec-plan`, an `argument-hint` that tells the user to provide a phase/task reference, and `agent: "plan"` so the prompt stays in planning/drafting mode rather than attempting implementation.
4. In the prompt body, instruct Copilot to use `docs/full-spec-plan.md` as the primary source and `docs/full-spec.md` as the supporting requirements source. The prompt should explicitly say to locate the referenced phased task, inherit its dependencies and details, and map the result into the simplified issue template.
5. Standardize the generated output format around the simplified sections: Title, Phase / Task ID, Labels, Blocked by, Goal, Spec Foundation, Inputs, Scope Included, Scope Excluded, Deliverables, Acceptance Criteria, Verification, and Notes / Risks.
6. Require the prompt to keep the generated issue focused on a single functional unit, preserve existing phase numbering, and avoid inventing architecture, scope, or dependencies that are not supported by the planning docs.
7. Add prompt instructions for sensible defaults where the plan is underspecified: infer labels from phase/domain, keep `Blocked by` as plan references when GitHub issue numbers do not yet exist, and leave Notes / Risks concise.
8. Include a short example invocation in the prompt body, such as `Phase 3, Task 3.3` or `2.5 Step reordering (move)`, so the usage pattern is obvious when opened directly in the editor or invoked from chat.
9. Add a fallback behavior for ambiguous references: if the requested phase/task cannot be uniquely identified, the prompt should ask for clarification rather than drafting the wrong issue.
10. Validate the prompt design by mentally testing it against at least one backend task, one frontend task, and one infrastructure task from `docs/full-spec-plan.md` to ensure the template stays concise but sufficient.
11. After approval, the next execution step would be to create the actual prompt file and optionally add one or two example prompt invocations to the planning docs or repository README.

**Relevant files**
- `c:\Code\rob-bl8ke\checklist-execution-system-planning\docs\full-spec-plan.md` — primary source for phased tasks, dependencies, design decisions, and delivery expectations.
- `c:\Code\rob-bl8ke\checklist-execution-system-planning\docs\full-spec.md` — supporting source for system requirements, API expectations, UX behavior, and data model semantics.
- `c:\Code\rob-bl8ke\checklist-execution-system-planning\.github\prompts\` — target location for the new workspace prompt file.

**Verification**
1. Confirm the file type and location match VS Code prompt conventions: `.github/prompts/*.prompt.md`.
2. Confirm the frontmatter includes a useful `description` and `argument-hint`, since those drive discoverability and usability in chat.
3. Test the prompt design against sample inputs like `3.3 Instance creation flow`, `8.3 Run execution screen`, and `1.6 Docker setup` to ensure the output stays consistent across backend, frontend, and infra tasks.
4. Check that the prompt does not require extra repo files or hidden assumptions beyond the two planning docs.
5. Confirm the generated markdown matches the simplified issue template and uses plan references instead of fabricated GitHub issue IDs when IDs do not yet exist.

**Decisions**
- Use a workspace prompt, not a user-profile prompt.
- Use a `.prompt.md` file, not instructions or a skill.
- Set the prompt to planning-oriented behavior that produces issue-ready markdown only.
- Keep the output aligned to the simplified issue template defined in this conversation.
- Treat `docs/full-spec-plan.md` as the primary task source and `docs/full-spec.md` as supporting context.

**Further Considerations**
1. Decide whether to create one general-purpose prompt for all tasks or a second prompt later for batching multiple tasks into a milestone or issue set. Recommendation: start with one single-task prompt only.
2. Decide whether the prompt should emit plain markdown only or include optional YAML/frontmatter for downstream automation. Recommendation: plain markdown only for now.
3. Decide whether to also add a companion `.instructions.md` file later if repeated issue-writing style rules start cluttering the prompt. Recommendation: defer until prompt usage proves that need.