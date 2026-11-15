---
description: Full feature workflow - branch, plan, build, review, test, docs, PR (stops for human approval at git steps)
argument-hint: <feature name>
---

Run the full feature workflow for: **$ARGUMENTS**. Follow AGENTS.md strictly.

1. **Branch:** check the current branch. If it's `main` or unrelated, **ask the dev** to create `feature/<kebab-name>` (offer the exact command). Wait for confirmation.
2. **Plan:** do what `/plan` does → `docs/plans/<name>.md`. Show a short summary and ask the dev to approve it before building.
3. **Build:** implement in small steps. After each logical step, propose a Conventional Commit message and **ask before committing**.
4. **Recheck** (run as sub-agents, in parallel where possible):
   - `code-reviewer`
   - `security-auditor` (mandatory if auth, booking, patient data, files, logs or dependencies were touched)
   - `test-engineer`
   - `ui-reviewer` if screens changed, `db-reviewer` if migrations changed
   Fix real findings and re-run until security says PASS.
5. **Docs:** run `docs-keeper` (CHANGELOG, PROGRESS, API/DB, pitch deck + video script if user-visible).
6. **Health:** run the `/doctor` checks.
7. **PR:** summarise what changed, the test results (real numbers) and the security verdict. **Ask the dev** before pushing or opening the PR, then use the PR template.

Never skip a step silently. If something can't run, say so.
