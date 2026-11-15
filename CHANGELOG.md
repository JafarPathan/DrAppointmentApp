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
- `DEBATE.md` cross-agent debate hub with the "debate first, change later" and "detailed, researched stamps" rules.
- ADR-0002: no over-booking for any slot capacity (lazily created slot inventory + atomic capacity claim).
- Problem statement for small Indian clinics and new workflows: walk-in registration at the desk, running the queue, daily capacity ("Full today"), doctor handing the desk to a receptionist.
- "3 patients ahead of you" alert, sent once as an app notification (notification SMS on hold for MVP; OTP SMS only).
- Desk add-patient: registered phone → added straight to the queue with an in-app notice; new phone → OTP registration.
- Single doctor-side login for MVP, with Desk mode, app-lock PIN and a signed-in devices list.
- Doctors choose their booking style per clinic: token queue or time slots.
- Live queue: pull to refresh at any time to see your current position.
- WhatsApp reminders parked for a later release (on hold).
- ADR-0005: pnpm + Turborepo monorepo, OpenAPI 3.0 generated at build time, Orval typed client, CI drift check.
- ADR-0006: doctors choose token queue or time slots; session limits with "Full today"; doctor "extra" patient; safe queue reordering.
- ADR-0007: single doctor-side login with trusted devices, server-enforced Desk mode, and separate Desk PIN / Doctor PIN.
- ADR-0008: desk add-patient (registered → direct with "Not me" undo; new → scoped OTP registration).
- ADR-0010: clinic QR ("Scan to track your token") with app links and a tracker-free landing page; app opens today's queue after login.
- ADR-0004: reliable live-queue updates (realtime + push for 4 key events + versioned resync), with privacy-safe notification text.

<!-- Template for new entries:
### Added | Changed | Fixed | Security | Removed
- <what changed, user-facing words> (#PR or branch)
-->
