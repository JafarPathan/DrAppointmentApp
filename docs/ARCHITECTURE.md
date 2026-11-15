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
 │ Reminders · slot materialisation · data retention   │
 │ cleanup · audit export                              │
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
| Jobs | Hangfire (Postgres storage) or .NET hosted services | |
| Logs/APM | Serilog → Application Insights, **with a PHI-scrubbing enricher** | |
| CI/CD | GitHub Actions → Azure App Service; EAS for mobile | |
| IaC | Bicep or Terraform | Later |

## 3. Backend layers (Clean Architecture)

```
DrBooking.Api            → HTTP, auth setup, SignalR hubs, middleware
DrBooking.Application    → use cases (BookAppointment, CheckIn...), validators, DTOs, interfaces
DrBooking.Domain         → entities, value objects, domain rules (slot rules, status transitions)
DrBooking.Infrastructure → EF Core DbContext, migrations, encryption, SMS, push, blob storage
```

Dependencies point inward. Domain has no NuGet framework dependencies.

## 4. Key flows

### Booking (no double-booking)
1. App calls `GET /doctors/{id}/slots?clinicId&date`. The server computes slots from the clinic schedule minus leaves and existing bookings.
2. User picks a slot and calls `POST /appointments` with an `Idempotency-Key` header.
3. The server validates: slot exists in the schedule, is in the future and inside the booking window, and the patient belongs to the caller.
4. Insert into `appointments`. A **partial unique index** on `(doctor_id, clinic_id, slot_start) WHERE status IN ('Booked','CheckedIn','InConsultation')` guarantees one booking per slot (or a capacity check under `SERIALIZABLE` when a slot allows more than one patient).
5. Assign the token number (per clinic + date sequence), commit, push the SignalR update and queue reminders.

### Live queue
- The doctor dashboard moves a patient's status, the API updates the DB and broadcasts to SignalR group `clinic:{clinicId}:{date}:{sessionId}`.
- Patients join only the group for their **own** appointment (the server checks ownership on `JoinQueue`). They receive only `{position, etaMinutes, status}`, never other patients' names.

## 5. Environments

| Env | Purpose | Data |
|---|---|---|
| local | Docker Postgres, dev keys | Fake seed data only |
| dev | Shared test cloud env | Fake data only |
| staging | Release candidate, store internal testing | Fake data only |
| prod | Real users | Real data, restricted access, all access audited |

**Production data never leaves prod.** No prod DB dumps on laptops.
