How to use it in chat:
1. Open Copilot Chat.
2. Type `/`.
3. Search for `Task From Phase Plan`.
4. Select it.
5. Enter a phase/task reference such as:
   - `3.3`
   - `Phase 3 Task 3.3`
   - `2.5 Step reordering (move)`
   - `8.3 Run execution screen`

Other ways to run it:
1. Run `Chat: Run Prompt...` from the command palette and choose `Task From Phase Plan`.
2. Open task-from-phase.prompt.md and use the play button.

Example:
- `/Task From Phase Plan 3.3` if your chat UI supports inline prompt arguments
- Otherwise: `/` → `Task From Phase Plan` → enter `3.3`

Note:
- Your planning note in plan.md is still uncommitted and untouched.
- I did not create any companion instructions files or batch prompts yet.

Natural next steps:
1. Run the prompt once against `3.3` and check whether the issue format feels right.
2. If needed, I can tighten the prompt wording based on that first output.
3. I can add a second prompt for converting an entire phase into multiple issue drafts.

Made changes.