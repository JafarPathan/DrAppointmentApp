# Architecture

## 1. Big picture

```
 ┌────────────────────┐     ┌────────────────────────┐
 │ Patient mobile app │     │ Doctor / Admin web     │
 │ React Native+Expo  │     │ React + Vite           │
 │ (Android + iOS)    │     │                        │
 └─────────┬──────────┘     └───────────┬────────────┘
           │ HTTPS (TLS 1.2+, pinned)   │ HTTPS + HttpOnly cookies
           │ JWT access token           │
           ▼                            ▼
 ┌────────────────────────────────────────────────────┐
 │  Azure Front Door / WAF  (rate limit, bot filter)  │
 └──────────────────────────┬─────────────────────────┘
                            ▼
 ┌────────────────────────────────────────────────────┐
 │ ASP.NET Core API (.NET 10)                         │
 │  - REST controllers   - SignalR QueueHub           │
 │  - AuthN: OTP + JWT   - AuthZ: policies + resource │
 │  - FluentValidation   - Rate limiting              │
 │  - Audit middleware   - Field encryption           │
 └───────┬───────────────┬───────────────┬────────────┘
         ▼               ▼               ▼
 ┌──────────────┐ ┌──────────────┐ ┌───────────────────┐
 │ PostgreSQL   │ │ Key Vault    │ │ Blob Storage      │
 │ (encrypted   │ │ (keys,       │ │ (encrypted files, │
 │  at rest +   │ │  secrets)    │ │  private, SAS)    │
 │  field enc.) │ └──────────────┘ └───────────────────┘
 └──────────────┘
         ▲
         │ background jobs (Hangfire / hosted services)
 ┌───────┴─────────────────────────────────────────────┐
 │ Reminders · slot-inventory reconcile · optional     │
 │ slot pre-generation · retention cleanup · audit     │
 └──────────────┬──────────────────────────────────────┘
                ▼
     SMS gateway (OTP, reminders) · FCM / APNs (push)
```

## 2. Tech choices

| Concern | Choice | Notes |
|---|---|---|
| Mobile | React Native + Expo (TypeScript), Expo Router | EAS Build for store builds, EAS Update for OTA JS fixes |
| Mobile state | TanStack Query (server), Zustand (UI) | |
| Forms | react-hook-form + zod | Same zod schemas reused on web |
| Secure storage | `expo-secure-store` | Refresh token only. Access token kept in memory |
| Biometrics | `expo-local-authentication` | Optional app lock |
| Push | `expo-notifications` (FCM + APNs) | Payload has no health details |
| Web | React + Vite + TypeScript, TanStack Query, a component lib (shadcn/ui or MUI) | |
| API | ASP.NET Core 10, minimal API or controllers, Clean Architecture layers | |
| ORM | EF Core (+ Dapper for hot read paths if needed) | |
| DB | PostgreSQL 16+ (Azure Flexible Server, Central India) | SQL Server also works; Postgres picked for cost + RLS |
| Realtime | SignalR (Azure SignalR Service in prod) | Queue updates per clinic session group |
| Jobs | Hangfire (Postgres storage) or .NET hosted services | Includes the nightly `slot_inventory` reconcile (ADR-0002). Booking correctness never depends on a job |
| Logs/APM | Serilog → Application Insights, **with a PHI-scrubbing enricher** | |
| CI/CD | GitHub Actions → Azure App Service; EAS for mobile | CI checks from ADR-0005: OpenAPI doc produced, zero client drift, lint/typecheck/test |
| IaC | Bicep or Terraform | Later |
| Monorepo ([ADR-0005](decisions/ADR-0005-monorepo-and-client-pipeline.md)) | **pnpm workspaces** (`nodeLinker: hoisted`) + **Turborepo** | .NET backend wired in via a thin `backend/package.json` |
| API contract | Built-in `Microsoft.AspNetCore.OpenApi`, **OpenAPI 3.0**, generated at build time to `backend/openapi/v1.json` (committed) | Startup skips Key Vault/DB when `BuildContext.IsOpenApiGeneration` (`GetDocument.Insider`) |
| API client | **Orval** → TanStack Query hooks + Zod schemas in `packages/api-client` | Custom mutator handles token, refresh, `Idempotency-Key` |
| App links + public web ([ADR-0010](decisions/ADR-0010-clinic-qr-app-links.md)) | iOS Universal Links + Android App Links via Expo; static landing site on a **separate domain** (`<web-domain>`, TBD) | Clinic QR → `/c/<publicCode>`. No third-party deep-link SDK; Firebase Dynamic Links is shut down |

## 3. Backend layers (Clean Architecture)

```
DrBooking.Api            → HTTP, auth setup, SignalR hubs, middleware
DrBooking.Application    → use cases (BookAppointment, CheckIn...), validators, DTOs, interfaces
DrBooking.Domain         → entities, value objects, domain rules (slot rules, status transitions)
DrBooking.Infrastructure → EF Core DbContext, migrations, encryption, SMS, push, blob storage
```

Dependencies point inward. Domain has no NuGet framework dependencies.

## 4. Key flows

### Booking (no over-booking, any capacity): [ADR-0002](decisions/ADR-0002-slot-concurrency-inventory.md)
1. App calls `GET /doctors/{id}/slots?clinicId&date`. The server **computes** slots from `clinic_sessions` minus `doctor_leaves`, overlays `slot_inventory` counts where rows exist, and caches per `(doctor_clinic, date)`.
2. User picks a slot and calls `POST /appointments` with an `Idempotency-Key` header.
3. The server validates: the slot exists in the schedule, is in the future and inside the booking window, isn't on leave, and the family member belongs to the caller. Clients can never create an arbitrary slot.
4. **One Read Committed transaction** in `BookAppointmentCommandHandler`:
   1. `INSERT INTO slot_inventory ... ON CONFLICT (doctor_clinic_id, slot_start) DO NOTHING` (lazy creation).
   2. `UPDATE slot_inventory SET booked_count = booked_count + 1 ... WHERE status='Open' AND booked_count < max_capacity RETURNING id`. **0 rows → 409 Conflict**, with no retry loop.
   3. Take the next token from `daily_token_counters` (`ON CONFLICT DO UPDATE ... RETURNING`).
   4. Insert the appointment (`slot_inventory_id`, `token_no`) and commit.
5. After commit: bump the queue `session_version`, publish the SignalR update, schedule reminders.

Cancellations and status changes decrement `booked_count` in the same transaction. Leaves and removed sessions set slots to `Blocked` and run cancel-and-notify (WORKFLOW A4). A nightly Hangfire job reconciles `booked_count` and alerts on drift. **No DB triggers.**

### Booking styles and the clinic queue: [ADR-0006](decisions/ADR-0006-token-queue-and-capacity.md)
- Each clinic has `queue_mode` = **Token** or **TimeSlot**, chosen by the doctor at setup.
- **Token mode:** one `slot_inventory` row per session per day, `max_capacity` = session limit. App bookings and desk walk-ins claim it the same way, which gives "Full today". ETA = patients ahead × `avg_consult_minutes`, shown as an estimate.
- **Doctor "extra" token:** `is_over_capacity = true`, bypasses `slot_inventory`, doctor only (not Desk mode), audited.
- **Order:** `token_no` never changes. `queue_position` (integer) is renumbered 1..n under a session row lock.
- **Concurrency:** status transitions are guarded updates without If-Match. "Call next" is chosen server-side under the lock. Only reorders need `If-Match: sessionVersion`.

### Desk and devices: [ADR-0007](decisions/ADR-0007-single-doctor-login-desk-mode.md), [ADR-0008](decisions/ADR-0008-desk-patient-registration.md)
- One doctor-side login. Each device is registered with a random `device_id`. Desk mode is enforced **server-side** per device. Desk Unlock PIN and Doctor PIN are separate.
- Desk add-patient: `POST /desk/patients/check` → registered: direct add + in-app "Not me" notice; new: scoped `DeskRegistration` OTP → account + consent + queue entry.

### Clinic QR: [ADR-0010](decisions/ADR-0010-clinic-qr-app-links.md)
- Static QR per clinic → `https://<web-domain>/c/<publicCode>` → app (if installed) or static landing page → store. After login, the app opens Live queue when there's an appointment today.

### Live queue: [ADR-0004](decisions/ADR-0004-resilient-queue-notifications.md)
- The doctor dashboard (or a Desk-mode device) moves a patient's status. The API updates the DB, **bumps the session's `session_version` in the same transaction**, and broadcasts to SignalR group `clinic:{clinicId}:{date}:{sessionId}`.
- Patients join only the group for their **own** appointment (`JoinAppointment`; the server checks ownership). They receive only `{appointmentId, position, etaMinutes, status, sessionVersion}`, never other patients' data.
- **The server is the single source of truth.** Position and ETA are always computed server-side, never by the client counting events.

| Tier | When | Mechanism |
|---|---|---|
| 1. Realtime | Queue screen focused, socket up | SignalR `QueueUpdated` |
| 2. Push | Backgrounded or killed | High-priority FCM/APNs for **4 events only**: **3 patients ahead** (once per visit), patient is next, doctor running late, doctor cancelled |
| 3. Reconciliation | Foreground, reconnect, version gap, or socket down while focused | `GET /appointments/{id}/queue` with ETag/304; 30 s polling with backoff while focused and disconnected |

- **Reconnect:** `withAutomaticReconnect()` gives a new connectionId, so in `onreconnected` the client re-invokes `JoinAppointment` and then refetches over REST.
- **Gap detection:** if `sessionVersion` isn't `last + 1`, the client refetches.
- **SMS: ⏸️ on hold for MVP** (maintainer decision). Built behind a feature flag (default off) for "3 patients ahead", doctor cancellation and delays > 30 min. Only OTP SMS (login and registration) is sent.
- **OEM battery tip** on first booking (Xiaomi, Oppo, Vivo, Realme, Samsung).
- 🔒 Push and SMS text is generic, with no clinic name (opt-in), speciality or purpose. See DESIGN §7.

## 5. Environments

| Env | Purpose | Data |
|---|---|---|
| local | Docker Postgres, dev keys | Fake seed data only |
| dev | Shared test cloud env | Fake data only |
| staging | Release candidate, store internal testing | Fake data only |
| prod | Real users | Real data, restricted access, all access audited |

**Production data never leaves prod.** No prod DB dumps on laptops.
