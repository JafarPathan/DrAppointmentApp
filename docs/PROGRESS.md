# Progress tracker

> **Auto-updated.** Every feature branch must update this file (see [AGENTS.md](../AGENTS.md) Rule 1). The `docs-keeper` agent and `/sync-docs` do it for you.
> Status: ⬜ not started · 🟨 in progress · ✅ done · ⏸️ parked

**Current phase:** Phase 0: Foundation
**Last updated:** 2026-11-16

---

## Phase 0: Foundation (docs, repo, tooling)

| # | Item | Owner | Status |
|---|---|---|---|
| 0.1 | Product requirements (PRD) | — | ✅ |
| 0.2 | Architecture + tech-stack ADR | — | ✅ |
| 0.3 | Security & privacy design | — | ✅ |
| 0.4 | DB schema draft | — | ✅ |
| 0.5 | API contract draft | — | ✅ |
| 0.6 | Design system + screen list | — | ✅ |
| 0.7 | AI agent rules, sub-agents, slash commands, git hooks | — | ✅ |
| 0.8 | Pitch deck + AI video script | — | ✅ |
| 0.9 | Monorepo scaffold per ADR-0005: pnpm (hoisted) + Turborepo, Expo app, Vite web, static `web-landing`, .NET solution, `BuildContext.IsOpenApiGeneration`, Orval (react-query + zod) | — | ⬜ ready |
| 0.10 | CI pipeline: build, test, lint, secret scan, dependency audit + ADR-0005 checks (OpenAPI doc produced, `git diff --exit-code` drift check) | — | ⬜ ready |
| 0.12 | Choose product name + buy web domain (needed for clinic QR / app links, ADR-0010) | maintainer | ⬜ |
| 0.11 | Azure dev environment (Postgres, Key Vault, App Service) | — | ⬜ |

## Phase 1: MVP (in-clinic booking)

| # | Item | Status |
|---|---|---|
| 1.1 | Phone OTP login + refresh-token rotation | ⬜ |
| 1.2 | Patient profile + family members (consent captured) | ⬜ |
| 1.3 | Doctor onboarding + registration-number verification (admin) | ⬜ |
| 1.4 | Clinics, timings, slot length, leave days | ⬜ |
| 1.5 | Slot generation + doctor search (name, speciality, city) | ⬜ |
| 1.6 | Book / cancel / reschedule with over-booking prevention (ADR-0002: `slot_inventory`, token counters, nightly reconcile) | ⬜ |
| 1.6a | ↳ Concurrency integration test: N parallel bookings on capacity k → exactly k succeed | ⬜ |
| 1.6b | ↳ Load test (k6/NBomber), ~200 concurrent bookings on one slot | ⬜ |
| 1.7 | Doctor dashboard: today's appointments, check-in, live queue | ⬜ |
| 1.8 | Live queue for patients (ADR-0004: SignalR + versioned resync + ETag polling) | ⬜ |
| 1.8a | ↳ Design queue-state table (`session_version`) via `/plan live-queue` | ⬜ |
| 1.9 | Push notifications (ADR-0004: 4 high-priority events, generic text, clinic name opt-in) | ⬜ |
| 1.9a | ↳ ⏸️ Notification SMS on hold (maintainer). Build behind a feature flag, off; DLT templates + TRAI source when switched on | ⏸️ |
| 1.9b | ↳ OEM battery-settings tip | ⬜ |
| 1.9c | ↳ "3 patients ahead" alert: once per visit, app notification (maintainer decision) | ⬜ |
| 1.10 | Audit log + account deletion / data export (DPDP) | ⬜ |
| 1.13 | Both booking styles (Token queue / Time slots), doctor chooses at setup, session capacity, "Full today", doctor "extra" token, integer queue positions (ADR-0006) | ⬜ |
| 1.13a | ↳ TimeSlot-mode walk-ins: open a DEBATE thread before building | ⬜ |
| 1.18 | Clinic QR ("Scan to track your token / book next visit") + Universal/App Links + static landing page + auto-open today's queue after login (ADR-0010; needs 0.12 domain) | ⬜ |
| 1.17 | ⏸️ WhatsApp reminders on hold (maintainer). MVP = app notifications + pull-to-refresh on Live queue (Thread 010) | ⏸️ |
| 1.14 | Desk add-patient: `/desk/patients/check`; registered → add directly + "Not me" notice; new → scoped OTP registration (ADR-0008; legal sign-off pending on consent) | ⬜ |
| 1.15 | Desk: call next (server-picked), not here / move back, no-show, reorder with If-Match, all audited (ADR-0006) | ⬜ |
| 1.16 | Single doctor-side login + trusted devices (random device ID) + Desk mode enforced server-side + Desk PIN / Doctor PIN (ADR-0007) | ⬜ |
| 1.11 | Security hardening pass + pen-test checklist | ⬜ |
| 1.12 | Play Store + App Store internal testing release | ⬜ |

## Phase 2: Growth

| # | Item | Status |
|---|---|---|
| 2.1 | Online payment for booking fee (Razorpay) | ⬜ |
| 2.1a | Automatic WhatsApp reminders via the WhatsApp Business API (paid; verify Meta India pricing first, Thread 010) | ⬜ |
| 2.2 | Video consult (Telemedicine Practice Guidelines compliant) | ⬜ |
| 2.3 | Separate receptionist logins with permissions (revisit **before** prescriptions/records, maintainer chose a single login for MVP) | ⬜ |
| 2.4 | Health records upload (encrypted file storage) | ⬜ |
| 2.5 | Prescriptions (e-Rx PDF, signed) | ⬜ |
| 2.6 | Ratings & reviews (verified visits only) | ⬜ |
| 2.7 | Hindi + Marathi languages | ⬜ |
| 2.8 | ABHA (ABDM) health ID linking | ⬜ |

## Phase 3: Scale

| # | Item | Status |
|---|---|---|
| 3.1 | Doctor subscription plans + billing | ⬜ |
| 3.2 | Predicted wait time (based on past queue speed) | ⬜ |
| 3.3 | Multi-doctor clinics / hospitals | ⬜ |
| 3.4 | Analytics for doctors (no-show rate, peak hours) | ⬜ |

---

## Progress log (append only, newest first)

| Date | Branch / PR | Who (dev / agent) | What changed |
|---|---|---|---|
| 2026-11-16 | — (main, uncommitted) | Claude Code | Wrote ADR-0005 (monorepo + client), ADR-0006 (booking styles + capacity), ADR-0007 (single login + Desk mode), ADR-0008 (desk registration), ADR-0010 (clinic QR). Updated DATABASE, API, SECURITY, ARCHITECTURE, PRD, WORKFLOW, DESIGN, README, CONTRIBUTING to match |
| 2026-11-16 | — (main, uncommitted) | Claude Code | Opened DEBATE Thread 011 (clinic QR to install/track). Docs note walk-ins can install the app and see their token |
| 2026-11-16 | — (main, uncommitted) | Claude Code | Maintainer decision: WhatsApp reminders on hold (Thread 010). Added pull-to-refresh on Live queue to PRD and DESIGN |
| 2026-11-16 | — (main, uncommitted) | Claude Code | Maintainer decision: both booking styles, the doctor chooses (Thread 007). Opened Thread 010 (free WhatsApp reminder). Replied to Antigravity on Threads 004, 007, 008, 009 |
| 2026-11-16 | — (main, uncommitted) | Claude Code | Recorded the maintainer's decisions: notification SMS on hold, single doctor-side login, desk add-patient (registered → direct, new → OTP). Updated PRD, WORKFLOW, API, ARCHITECTURE, DESIGN, SECURITY, ADR-0004 and pitch material |
| 2026-11-16 | — (main, uncommitted) | Claude Code | Added the small-clinic problem statement (PRD §1) and desk workflows A0, A6–A9. Recorded the maintainer's "3 ahead" decision (ADR-0004 amended). Opened DEBATE Threads 007–009 |
| 2026-11-16 | — (main, uncommitted) | Claude Code | Updated DATABASE, ARCHITECTURE §4, API, DESIGN §7, PRD §4–5 and PROGRESS Phase 1 to match ADR-0002 and ADR-0004. Walk-in slot handling flagged as an open question |
| 2026-11-16 | — (main, uncommitted) | Claude Code | Added ADR-0002 (slot concurrency via lazy slot inventory, DEBATE Thread 001) and ADR-0004 (resilient live-queue notifications, Thread 003). Docs not yet updated to match |
| 2026-11-16 | — (initial) | Claude Code | Created project docs, AI agent rules, sub-agents, slash commands, git hooks, pitch deck and video script |
