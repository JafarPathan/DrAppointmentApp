# CLAUDE.md

All rules for AI agents live in **[AGENTS.md](AGENTS.md)**. Read it first and follow it. It is mandatory.

Quick reminders:

- **🔒 Privacy and security first.** Read [docs/SECURITY.md](docs/SECURITY.md) before touching auth, patient data, files or logs.
- **New feature → ask the dev to create a branch.** Conventional Commits. Ask before any commit or push.
- **Every feature updates** CHANGELOG.md, docs/PROGRESS.md and, if user-visible, the pitch deck and video script. Use the `docs-keeper` agent or `/sync-docs`.
- **Before saying "done":** `code-reviewer` + `security-auditor` + `test-engineer` agents. Report real results.
- Sub-agents are in `.claude/agents/`. Slash commands are in `.claude/commands/`.
- **Debate first, change later (AGENTS.md Rule 0.5):** no design/architecture/security/API/dependency change starts until its DEBATE.md thread is `🟢 CONSENSUS` from **all** agents (or a human decision on an escalated thread). Stamps must be detailed and researched, with sources listed.
- **Cross-agent collaboration:** Check [DEBATE.md](DEBATE.md) for active debates, architecture critiques, and consensus stamps with Antigravity / Gemini and other agents.
