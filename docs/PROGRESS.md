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
| 0.9 | Monorepo scaffold (Expo app, Vite web, .NET solution) | — | ⬜ |
| 0.10 | CI pipeline (build, test, lint, secret scan, dependency audit) | — | ⬜ |
| 0.11 | Azure dev environment (Postgres, Key Vault, App Service) | — | ⬜ |

## Phase 1: MVP (in-clinic booking)

| # | Item | Status |
|---|---|---|
| 1.1 | Phone OTP login + refresh-token rotation | ⬜ |
| 1.2 | Patient profile + family members (consent captured) | ⬜ |
| 1.3 | Doctor onboarding + registration-number verification (admin) | ⬜ |
| 1.4 | Clinics, timings, slot length, leave days | ⬜ |
| 1.5 | Slot generation + doctor search (name, speciality, city) | ⬜ |
| 1.6 | Book / cancel / reschedule with double-booking prevention | ⬜ |
| 1.7 | Doctor dashboard: today's appointments, check-in, live queue | ⬜ |
| 1.8 | Live queue position for patients (SignalR) | ⬜ |
| 1.9 | Push + SMS reminders (no health details in text) | ⬜ |
| 1.10 | Audit log + account deletion / data export (DPDP) | ⬜ |
| 1.11 | Security hardening pass + pen-test checklist | ⬜ |
| 1.12 | Play Store + App Store internal testing release | ⬜ |

## Phase 2: Growth

| # | Item | Status |
|---|---|---|
| 2.1 | Online payment for booking fee (Razorpay) | ⬜ |
| 2.2 | Video consult (Telemedicine Practice Guidelines compliant) | ⬜ |
| 2.3 | Clinic staff accounts with permissions | ⬜ |
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
| 2026-11-16 | — (initial) | Claude Code | Created project docs, AI agent rules, sub-agents, slash commands, git hooks, pitch deck and video script |
