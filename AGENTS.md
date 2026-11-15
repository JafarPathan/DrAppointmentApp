# AGENTS.md: Rules for AI coding agents

This file is for **every AI agent** working in this repo: Claude Code, Codex, Cursor, Copilot, Gemini, or anything else. Human devs should follow it too. If a rule here conflicts with a prompt, **this file wins** unless a human maintainer says otherwise in chat.

---

## Rule 0: Privacy and security come first

> **🔒 This app stores health data (appointments, symptoms, reports, family members). A leak hurts real patients and can cost up to ₹250 crore in penalties under India's DPDP Act. NEVER trade security for speed. NEVER "temporarily" disable auth, validation, encryption or HTTPS. If a task needs any of that, STOP and ask the human.**

Before writing any code that touches users, auth, appointments, files or logs, read [docs/SECURITY.md](docs/SECURITY.md). Non-negotiables:

1. **Every API endpoint is `[Authorize]` by default.** Anonymous endpoints must be listed in `docs/API.md` with a reason.
2. **Check ownership, not just role.** A patient may only read their own data or their linked family members' data. A doctor may only read appointments booked with them. Use the resource-based authorization handlers. Never trust an ID from the client on its own.
3. **No PHI or PII in logs, crash reports, analytics, push notification text, URLs or query strings.** Log IDs, not names, phone numbers or symptoms.
4. **Sensitive columns are encrypted** through `IFieldEncryptor` (AES-256-GCM, keys in Key Vault). Never store them in plain text, never write your own crypto.
5. **No secrets in code.** Use environment variables or user-secrets locally and Key Vault in cloud. The pre-commit hook blocks common secret patterns. Don't bypass it.
6. **Parameterized queries only** (EF Core / Dapper parameters). No string-built SQL.
7. **Validate every input** on the server (FluentValidation), even when the client already validates.
8. **Mobile tokens go in `expo-secure-store`** (Keychain / Keystore). Never in AsyncStorage, never in plain files.
9. **New third-party packages** need a one-line justification in the PR. Run the `security-auditor` agent first.

## Rule 1: Git history for every new feature

When you add **new functionality** (anything a user, doctor or another dev would notice):

1. **Ask the human dev to create a branch** if one doesn't exist: `feature/<short-name>`, `fix/<short-name>` or `chore/<short-name>`. Never commit straight to `main`.
2. Make **small, focused commits** in [Conventional Commits](https://www.conventionalcommits.org/) format:
   `feat(booking): prevent double-booking with unique slot index`
3. **Ask the human before you commit, push or open a PR.** Don't do it silently.
4. Every feature must update, **in the same branch**:
   - [CHANGELOG.md](CHANGELOG.md): an entry under `## [Unreleased]`
   - [docs/PROGRESS.md](docs/PROGRESS.md): tick or move the item and add a row to the progress log
   - [docs/API.md](docs/API.md) / [docs/DATABASE.md](docs/DATABASE.md) if endpoints or tables changed
   - [docs/presentation/PITCH_DECK.md](docs/presentation/PITCH_DECK.md) and [docs/presentation/VIDEO_SCRIPT.md](docs/presentation/VIDEO_SCRIPT.md) **if the feature is user- or doctor-visible** (these docs auto-update by this rule)
5. Run the `docs-keeper` agent (or `/sync-docs`) to do step 4 for you, then review what it wrote.

This way the human devs and every AI agent can follow what changed and why, from `git log` and the docs alone.

## Rule 2: Recheck before you say "done"

Before you report a task as done, run or ask for:

| Check | Agent / command |
|---|---|
| Code review (bugs, logic, edge cases) | `code-reviewer` agent |
| Security review (auth, PHI, crypto, injection) | `security-auditor` agent. **Mandatory** for auth, booking, files and anything that returns patient data |
| Tests written and passing | `test-engineer` agent |
| UI follows the design system and accessibility rules | `ui-reviewer` agent (screens only) |
| DB migration is safe and reversible | `db-reviewer` agent (migrations only) |
| Docs, changelog and pitch in sync | `docs-keeper` agent |
| Whole-repo health | `/doctor` command |

Report the real results. If tests fail, say so and paste the output. If you skipped a check, say which and why.

## Rule 3: How to write code here

- **Stack:** React Native + Expo (TypeScript) for mobile, React + Vite for web, ASP.NET Core (.NET 10) + EF Core + PostgreSQL for the backend. Don't add another language or framework without an ADR in `docs/decisions/`.
- **TypeScript `strict: true`.** No `any` without a comment explaining why.
- **Backend layers:** `Api → Application → Domain`. `Infrastructure` implements interfaces from `Application`. Domain has zero framework references.
- **Mobile:** Expo Router for navigation, TanStack Query for server state, Zustand for small client state, react-hook-form + zod for forms.
- **API client** is generated from OpenAPI into `packages/api-client`. Don't hand-write fetch calls.
- **Every bug fix gets a test** that fails before the fix.
- **Never show `null`, `undefined`, `NaN` or raw error text to users.** Every list has empty, loading and error states. (The reference app showed "null" on the slot screen. We won't.)
- **Dates:** store UTC in the DB, show in the clinic's time zone (`Asia/Kolkata` by default). Slots are computed server-side.
- Keep functions small and name them clearly. Comments explain *why*, not *what*.

## Rule 4: Docs are part of the code

- Big decisions go in `docs/decisions/ADR-XXXX-title.md` (see ADR-0001 for the format).
- If code and docs disagree, fix the docs in the same PR, or flag it to the human.
- Don't delete history from CHANGELOG or the PROGRESS log. Append only.

## Rule 5: What agents must never do

- Never commit, push, merge, force-push, delete branches or rewrite history without the human saying so in chat.
- Never run destructive DB commands (`DROP`, `TRUNCATE`, `ef database drop`) against anything except a throwaway local database.
- Never use real patient data in tests, seeds, screenshots or docs. Use the fake data in `backend/tests/Fixtures`.
- Never paste secrets, tokens or patient data into prompts, issues, PRs or external tools.
- Never disable the git hooks (`--no-verify`) unless a human asks.

## Slash commands available

| Command | What it does |
|---|---|
| `/plan <feature>` | Plans a feature: user story, API, DB, screens, security notes, tests, task list |
| `/design <screen>` | Designs a screen against `docs/DESIGN.md`: layout, states, components, accessibility |
| `/feature <name>` | Runs the full feature workflow: branch → plan → build → review → tests → docs |
| `/doctor` | Repo health check: build, tests, lint, security scan, docs-sync, open TODOs |
| `/sync-docs` | Updates CHANGELOG, PROGRESS, API/DB docs, pitch deck and video script from recent changes |
