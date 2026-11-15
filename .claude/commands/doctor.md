---
description: Repo health check (build, tests, lint, security scan, dependency audit, docs sync, TODOs)
---

Run a full health check of the DrBookingApp repo and report a scorecard. Run what exists and skip what isn't scaffolded yet, saying so for each one.

1. **Git:** current branch (warn if on `main` with changes), uncommitted files, whether hooks are enabled (`git config core.hooksPath` should be `.githooks`).
2. **Backend** (if `backend/` exists): `dotnet build`, `dotnet test`, `dotnet list package --vulnerable --include-transitive`, `dotnet format --verify-no-changes`.
3. **Mobile / web** (if `apps/*` exist): `npm ci` if needed, then `npm run lint`, `npm run typecheck`, `npm test`, `npm audit --omit=dev`.
4. **Secrets:** grep tracked files for likely secrets (private keys, `AccountKey=`, `Password=`, `sk_live`, `AIza`, JWTs). Report file:line, but **never print the secret value**.
5. **Privacy smells:** grep for `console.log(`, `Console.WriteLine(`, logging calls that include `phone`, `name`, `dob`, `purpose`, `notes`; `[AllowAnonymous]` not listed in docs/API.md; `AsyncStorage` holding tokens; `http://` URLs.
6. **Docs sync:** compare code vs docs. Endpoints in code but missing from docs/API.md, entities missing from docs/DATABASE.md, `[Unreleased]` empty while the branch has feat/fix commits, PROGRESS.md "Last updated" older than the latest commit.
7. **TODO/FIXME/HACK** count, with any that mention security listed.

Output a table: Check | Status (✅/⚠️/❌/⏭️ skipped) | Details. Then list the top 3 things to fix first. Don't fix anything automatically; offer to.
