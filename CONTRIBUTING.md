# Contributing

Welcome! This guide is for human devs on the team. AI agents follow [AGENTS.md](AGENTS.md), which has the same rules plus a few more.

> **🔒 Privacy first. Read [docs/SECURITY.md](docs/SECURITY.md) before your first PR.**

## 1. Setup

1. Install: Node LTS, **pnpm** (via `corepack enable`), .NET 10 SDK, Docker Desktop (for local Postgres), Git, Android Studio (emulator), Xcode (Mac only, for iOS).
2. Clone the repo, then turn on the git hooks:
   ```bash
   git config core.hooksPath .githooks
   ```
3. Install and build everything (ADR-0005):
   ```bash
   pnpm install
   ```
   ```bash
   pnpm turbo run build
   ```
   Frontend-only devs don't need the .NET SDK to get API types: `backend/openapi/v1.json` is committed, and `pnpm --filter api-client generate` regenerates the client from it.
4. Local secrets go in `dotnet user-secrets` (backend) and `.env.local` (apps). Both are git-ignored. Ask the lead for **dev-only** keys. Never use production keys locally.

## 2. Branches

| Prefix | Use for | Example |
|---|---|---|
| `feature/` | New functionality | `feature/live-queue` |
| `fix/` | Bug fix | `fix/slot-timezone` |
| `security/` | Security fix (keep the PR description vague until merged) | `security/token-rotation` |
| `chore/` | Tooling, deps, docs | `chore/update-expo` |

`main` is always releasable. No direct pushes; everything goes through a PR with one review.

## 3. Commits: Conventional Commits

```
<type>(<scope>): <what, in present tense>

feat(booking): hold slot for 5 minutes during checkout
fix(queue): stop queue position going negative after cancel
security(auth): rotate refresh token on every use
docs(pitch): add live-queue slide
```

Types: `feat`, `fix`, `security`, `perf`, `refactor`, `test`, `docs`, `chore`.
Scopes: `auth`, `booking`, `queue`, `doctor`, `patient`, `search`, `notify`, `admin`, `mobile`, `web`, `api`, `db`, `infra`, `pitch`.

## 4. Pull request checklist

The PR template has the full list. The short version:

- [ ] Tests added/updated and passing
- [ ] `security-auditor` run (required for auth, booking, patient data and files)
- [ ] CHANGELOG.md + docs/PROGRESS.md updated
- [ ] API/DB docs updated if contracts changed
- [ ] Pitch deck / video script updated if the feature is user-visible
- [ ] No PHI in logs, no secrets, no `console.log` of user data
- [ ] Screenshots for UI changes (fake data only)

## 5. Working with AI agents

- Let the agent do the work, but **you own the commit**. Read the diff.
- Use `/feature <name>` to run the full flow, or `/plan` → build → `/doctor`.
- If an agent wants to skip a check "to save time", say no.

## 6. Definition of done

Code merged + tests green + security reviewed + docs synced + works on **both** an Android emulator and an iOS simulator/device.
