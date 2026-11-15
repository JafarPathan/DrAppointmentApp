# Changelog

All notable changes to this project are recorded here. Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versions: [Semantic Versioning](https://semver.org/).

> **Rule:** every feature or fix branch adds an entry under `[Unreleased]` before merge (see [AGENTS.md](AGENTS.md) Rule 1). Append only. Never rewrite past entries.

## [Unreleased]

### Added
- Project documentation: README, PRD, architecture, security & privacy, database, API, design system, workflows, research notes.
- AI agent rules (`AGENTS.md`, `CLAUDE.md`) with privacy-first and git-history rules.
- Claude Code sub-agents: code-reviewer, security-auditor, test-engineer, ui-reviewer, db-reviewer, docs-keeper.
- Slash commands: `/plan`, `/design`, `/feature`, `/doctor`, `/sync-docs`.
- Git hooks: secret scan + docs-sync warning on commit.
- Doctor pitch deck and AI-video script.

<!-- Template for new entries:
### Added | Changed | Fixed | Security | Removed
- <what changed, user-facing words> (#PR or branch)
-->
