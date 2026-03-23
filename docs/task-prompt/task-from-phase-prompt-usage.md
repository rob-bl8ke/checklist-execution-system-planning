## Task From Phase Plan: Usage

### What it does

The `Task From Phase Plan` prompt generates issue-ready markdown for a single phased task.

It now supports dynamic source selection:
- If you add a plan document to chat context, the prompt uses that plan first.
- If you add a supporting spec document to chat context, the prompt uses that spec first.
- If you do not provide either file, the prompt falls back to:
  - `docs/spec-plan.md`
  - `docs/full-spec.md`

### Recommended workflow

1. Open Copilot Chat.
2. If you want to use a non-default plan, add that plan file to chat context.
3. If needed, also add a supporting spec file to chat context.
4. Type `/`.
5. Search for `Task From Phase Plan`.
6. Select it.
7. Enter a phase/task reference such as:
   - `3.3`
   - `Phase 3 Task 3.3`
   - `2.5 Step reordering (move)`
   - `8.3 Run execution screen`

### Default behavior

If no files are added to chat context, the prompt uses:
- `docs/spec-plan.md` as the default plan
- `docs/full-spec.md` as the default supporting spec

### Example usage patterns

Use the default repo files:
- `/` → `Task From Phase Plan` → `3.3`

Use a different plan file in context:
1. Add your alternate plan markdown file to chat context.
2. Run `Task From Phase Plan`.
3. Enter `3.3`.

Use both a different plan and a different supporting spec:
1. Add both files to chat context.
2. Run `Task From Phase Plan`.
3. Enter `8.3 Run execution screen`.

### Other ways to run it

1. Run `Chat: Run Prompt...` from the command palette and choose `Task From Phase Plan`.
2. Open `.github/prompts/task-from-phase.prompt.md` and use the play button.

### Notes

- The prompt expects a single task reference, not a whole phase.
- If the task reference is ambiguous, the prompt should ask for clarification.
- If multiple plan-like files are in chat context, be explicit about which one should drive the result.