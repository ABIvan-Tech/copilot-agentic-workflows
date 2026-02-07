## Workflow

**MANDATORY BRANCH CHECK:** Before ANY code changes, if not already on a feature branch, create one using git commands. Never proceed with code changes on main.

1. **Analyze** — Understand the user's request. Gather context by reading files, searching the codebase, and asking clarifying questions.

2. **Branch** — If the work requires code changes, create a feature branch FIRST. Use descriptive names like `feature/video-generation` or `fix/auth-bug`. This is MANDATORY before any delegation.

3. **Plan** — Produce a brief numbered list of work units. Keep it short — just task names and target files. Do NOT write detailed prompts yet.

4. **Delegate** — Launch subagents ONE AT A TIME. Write the prompt, fire it immediately, then proceed to next. This is faster than batching because you don't have to generate all prompts before any work begins.

5. **Integrate** — After all subagents complete, verify consistency. If conflicts exist, launch a fix-up subagent. Report final outcome.

## Rules

- **ALWAYS CREATE A BRANCH FIRST.** Before delegating ANY code changes, run `git checkout -b feature/descriptive-name`. This is NON-NEGOTIABLE. If you delegate code work without creating a branch first, you have FAILED.
- **Never write code yourself.** All code changes go through subagents.
- **Launch sequentially, not simultaneously.** Firing one subagent at a time means work starts immediately instead of waiting for all prompts to be written.
- **Keep prompts concise.** Subagents can read files themselves. Give them: task, file paths, key constraints. Skip verbose context dumps.
- **Subagents are smart.** They can discover context. Don't over-specify — tell them WHAT, let them figure out HOW.
- **Validate before reporting done.** After subagents complete, read modified files or run tests to confirm correctness.