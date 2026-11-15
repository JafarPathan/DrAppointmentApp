# DrBookingApp (working name)

A doctor-appointment booking platform for **Android and iOS**. Patients find a doctor, pick a slot and book. Doctors and clinic staff run their day from their own account: slots, the live queue, and patient check-in.

> **🔒 PRIVACY IS PRIORITY #1. This app handles health data. Every feature, PR and AI-agent change must follow [docs/SECURITY.md](docs/SECURITY.md). If you are unsure whether something is safe, it is not safe. Stop and ask.**

---

## Who uses it

| Role | App | What they do |
|---|---|---|
| **Patient** | Mobile app (Android / iOS) | Sign in with OTP, add family members, search doctors and clinics, book, cancel or reschedule slots, see their place in the live queue |
| **Doctor** | Doctor web dashboard + mobile (doctor mode) | Manage clinics, timings, slot length, leave days, the live queue and appointments |
| **Clinic staff** (receptionist) | Doctor web dashboard | Check patients in, add walk-ins, move the queue (only what the doctor allows) |
| **Platform admin** | Admin area in the web dashboard | Verify doctors (NMC/State Medical Council registration), handle abuse reports and support |

## Tech stack (chosen for a React + .NET + SQL team)

| Layer | Choice | Why |
|---|---|---|
| Mobile (Android + iOS) | **React Native + Expo (TypeScript)** | One codebase for both stores. It's React, which the team already knows |
| Doctor/Admin web | **React + Vite + TypeScript** | Same skills, and it shares types and API client with mobile |
| Backend API | **ASP.NET Core (.NET 10 LTS) Web API** | Built-in Identity, Data Protection, rate limiting and policy authorization. Team knows .NET |
| Database | **PostgreSQL** via EF Core (SQL Server also works) | Row-level security, cheap managed hosting, atomic row-level capacity checks so slots are never over-booked (ADR-0002) |
| Realtime queue | **SignalR** | Live "you are #3 in queue" updates |
| Hosting | **Azure, Central India region** | Health data stays in India. Key Vault for keys |

Full reasoning: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) and [docs/decisions/ADR-0001-tech-stack.md](docs/decisions/ADR-0001-tech-stack.md).

## Repository layout (target)

```
apps/
  mobile/            React Native (Expo) patient app (+ doctor mode later)
  doctor-web/        React dashboard for doctors (incl. Desk mode) and platform admin
  web-landing/       Static clinic-QR landing site + app-association files (separate domain, ADR-0010)
backend/
  openapi/v1.json                OpenAPI 3.0 doc generated at build time, committed (ADR-0005)
  package.json                   Thin wrapper so Turborepo can run dotnet build/test
  src/DrBooking.Api/             ASP.NET Core API (controllers, auth, SignalR hubs)
  src/DrBooking.Application/     Use cases, validation, DTOs
  src/DrBooking.Domain/          Entities, rules (no framework code)
  src/DrBooking.Infrastructure/  EF Core, encryption, SMS, storage
  tests/                         Unit + integration tests
packages/
  api-client/        Orval-generated TanStack Query hooks + Zod schemas (from backend/openapi), shared by mobile + web
pnpm-workspace.yaml  pnpm workspaces, nodeLinker: hoisted (ADR-0005)
turbo.json           Turborepo task pipeline
docs/                Product, architecture, security, design, workflow, pitch
.claude/             AI sub-agents ("bots") and slash commands
.githooks/           Git hooks (docs-sync check, secret scan)
```

## Start here

| If you are... | Read |
|---|---|
| New dev on the team | [CONTRIBUTING.md](CONTRIBUTING.md), then [docs/WORKFLOW.md](docs/WORKFLOW.md) |
| An AI coding agent | [AGENTS.md](AGENTS.md). Those rules are mandatory |
| Checking what's done | [docs/PROGRESS.md](docs/PROGRESS.md) and [CHANGELOG.md](CHANGELOG.md) |
| Building a feature | [docs/PRD.md](docs/PRD.md), [docs/API.md](docs/API.md), [docs/DATABASE.md](docs/DATABASE.md) |
| Designing a screen | [docs/DESIGN.md](docs/DESIGN.md) |
| Pitching to doctors / making videos | [docs/presentation/PITCH_DECK.md](docs/presentation/PITCH_DECK.md), [docs/presentation/VIDEO_SCRIPT.md](docs/presentation/VIDEO_SCRIPT.md) |

## One-time setup after cloning

```bash
git config core.hooksPath .githooks
```

This turns on the pre-commit hook, which blocks commits that contain secrets and warns when code changed but the docs did not.
