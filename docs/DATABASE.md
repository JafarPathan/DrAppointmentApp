# Database (PostgreSQL)

> Auto-updated: any migration must update this file (AGENTS.md Rule 1). 🔐 = field-level encrypted (`bytea`, AES-256-GCM via `IFieldEncryptor`). # = HMAC blind index for lookup.

Conventions: `snake_case`, PKs are `uuid` (UUIDv7), all timestamps `timestamptz` in UTC, soft-delete via `deleted_at` where needed, `created_at` / `updated_at` on every table.

## Entity overview

```
users ──< family_members
  │
  ├──< appointments >── doctors ──< doctor_clinics >── clinics
  │                        │                            │
  │                        └──< clinic_sessions (weekly timings) ┘
  │                        └──< doctor_leaves
  │                        doctor_clinics ──< slot_inventory ──< appointments   (ADR-0002)
  │                        doctor_clinics ──< daily_token_counters             (ADR-0002)
  ├──< refresh_tokens
  ├──< trusted_devices                                                       (ADR-0007)
  ├──< consents
  └──< audit_log (append-only)
clinic_staff (deferred to Phase 2: MVP uses a single doctor-side login, ADR-0007)
```

## Tables

### users
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| role | text | `Patient`, `Doctor`, `ClinicStaff`, `PlatformAdmin` |
| phone_enc 🔐 | bytea | |
| phone_hash # | bytea UNIQUE | HMAC-SHA256 for login lookup |
| full_name_enc 🔐 | bytea | |
| dob_enc 🔐 | bytea | nullable |
| gender | text | nullable |
| status | text | `Active`, `Suspended`, `PendingDeletion` |
| mfa_secret_enc 🔐 | bytea | doctors/staff/admin |
| created_at, updated_at, deleted_at | timestamptz | |

### family_members
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| owner_user_id | uuid FK users | RLS key |
| relation | text | `Self`, `Spouse`, `Child`, `Parent`, `Sibling`, `Other` |
| full_name_enc 🔐, dob_enc 🔐 | bytea | |
| gender | text | |
| is_minor | bool | derived. Drives parental-consent rules |

> Every patient gets a `Self` family member, so appointments always point to `family_members.id`.

### doctors
| Column | Type | Notes |
|---|---|---|
| id | uuid PK, FK users.id | |
| display_name | text | public |
| qualifications | text | public, e.g. "MBBS, DNB" |
| speciality_id | uuid FK specialities | |
| experience_since_year | int | compute "14+ yrs" at runtime |
| registration_no_enc 🔐 | bytea | |
| registration_council | text | |
| verification_status | text | `Pending`, `Verified`, `Rejected` |
| photo_blob_key | text | public photo only |
| is_listed | bool | shows in search only if verified + listed |
| doctor_pin_hash | text | **Argon2id** hash of the private Doctor PIN (exits Desk mode, ADR-0007). Plus `doctor_pin_failed_count` and `doctor_pin_locked_until` (5 failures → OTP + second factor) |

### specialities
`id`, `name`, `slug`, `icon`, `sort_order`

### clinics
`id`, `name`, `address_line`, `area`, `city`, `pincode`, `lat`, `lng`, `phone`, `timezone` (default `Asia/Kolkata`), `public_code` (random, rotatable, UNIQUE; used in the clinic QR link `/c/<public_code>`, ADR-0010; never the DB id)

### doctor_clinics
`id`, `doctor_id`, `clinic_id`, **`queue_mode`** (`Token` | `TimeSlot`, required at setup, ADR-0006), `slot_minutes` (TimeSlot only, e.g. 10), `patients_per_slot` (TimeSlot only, default 1), **`avg_consult_minutes`** (Token ETA, default 10), `booking_window_days` (7), `cancel_cutoff_minutes` (60), `consult_fee` (nullable), `is_active`

### clinic_sessions (weekly recurring timings)
`id`, `doctor_clinic_id`, `day_of_week` (0–6), `start_time` (local `time`), `end_time`, `label` (`Morning`/`Noon`/`Evening`, derived if null), **`max_patients`** (Token mode session limit, ADR-0006)

### session_capacity_overrides (ADR-0006)
`doctor_clinic_id`, `clinic_session_id`, `local_date`, `max_patients`: "today only" limit changes by the doctor

### doctor_leaves
`id`, `doctor_clinic_id` (null = all clinics), `start_at`, `end_at`, `reason_enc` 🔐

### appointments
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| booked_by_user_id | uuid FK users | RLS key |
| family_member_id | uuid FK family_members | the actual patient |
| doctor_id, clinic_id | uuid | |
| slot_inventory_id | uuid FK slot_inventory | capacity claim (ADR-0002). In Token mode, the session row. **Null for doctor "extra" tokens** (`is_over_capacity`) |
| slot_start, slot_end | timestamptz | UTC. Token mode: session start/end |
| token_no | int | per doctor + clinic + local date, from `daily_token_counters` (ADR-0002). **Never changes** |
| queue_position | int | current calling order within session + date. Renumbered 1..n under a session row lock (ADR-0006) |
| is_over_capacity | bool | doctor override beyond the limit. Doesn't touch `slot_inventory` (ADR-0006) |
| display_name_enc 🔐 | bytea | name typed at the desk for a registered patient new to this clinic (ADR-0008). This clinic's view only; never overwrites the profile |
| purpose | text | enum: `New`, `FollowUp`, `ShowReports`, `Vaccination`, `Emergency`, `Other` |
| purpose_note_enc 🔐 | bytea | optional free text |
| status | text | `Booked`, `CheckedIn`, `NotPresent`, `InConsultation`, `Completed`, `NoShow`, `CancelledByPatient`, `CancelledByDoctor`. Active = Booked, CheckedIn, NotPresent, InConsultation (ADR-0006) |
| source | text | `App`, `Desk` (ADR-0008) |
| idempotency_key | text | UNIQUE per booked_by_user_id |
| cancelled_at, checked_in_at, started_at, completed_at | timestamptz | |

**Constraints:**
```sql
-- a patient can't double-book the same doctor on the same slot
CREATE UNIQUE INDEX ux_appt_member_slot
  ON appointments (family_member_id, doctor_id, slot_start)
  WHERE status IN ('Booked','CheckedIn','NotPresent','InConsultation');
```
> Slot **capacity** is enforced by `slot_inventory` (below), not by an index on `appointments`. See [ADR-0002](decisions/ADR-0002-slot-concurrency-inventory.md).
>
> **Decided (ADR-0006):** in Token mode, walk-ins claim the same session capacity as app bookings.
> **Still open (needs a DEBATE thread before building):** walk-ins in **TimeSlot** mode, and whether to add a unique index on token numbers as a safety net.

### slot_inventory (ADR-0002)
Created **lazily** inside the booking transaction (`INSERT ... ON CONFLICT DO NOTHING`), optionally pre-generated 14 days ahead for read speed only. No PHI/PII.

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| doctor_clinic_id | uuid FK doctor_clinics | |
| slot_start, slot_end | timestamptz | UTC |
| max_capacity | int | when the row is created: TimeSlot → `patients_per_slot`; Token → session `max_patients` (or today's override) (ADR-0006) |
| booked_count | int | default 0. Changed only by Application command handlers in the same transaction as the appointment change. **No DB triggers** |
| status | text | `Open`, `Blocked` (leave, removed session, or doctor action) |
| version | int | +1 on every change |
| created_at, updated_at | timestamptz | |

```sql
ALTER TABLE slot_inventory ADD CONSTRAINT ux_slot UNIQUE (doctor_clinic_id, slot_start);
ALTER TABLE slot_inventory ADD CONSTRAINT ck_slot_capacity
  CHECK (booked_count >= 0 AND booked_count <= max_capacity);
```

**Capacity claim** (Read Committed; 0 rows → 409):
```sql
UPDATE slot_inventory
   SET booked_count = booked_count + 1, version = version + 1, updated_at = now()
 WHERE doctor_clinic_id = @dc AND slot_start = @start
   AND status = 'Open' AND booked_count < max_capacity
RETURNING id;
```
Lowering `patients_per_slot` sets `max_capacity = GREATEST(new_capacity, booked_count)` on existing future rows. Over-capacity slots are listed for the doctor to resolve.
A **nightly reconcile job** compares `booked_count` with the count of active appointments **where `is_over_capacity = false`** and alerts on drift.

### daily_token_counters (ADR-0002)
| Column | Type | Notes |
|---|---|---|
| doctor_clinic_id | uuid FK | PK part 1 |
| local_date | date | clinic-local date, PK part 2 |
| next_token_no | int | assigned with `INSERT ... ON CONFLICT DO UPDATE SET next_token_no = next_token_no + 1 RETURNING` |

Never compute tokens with `MAX(token_no) + 1`.

### Queue state (planned, ADR-0004)
Live-queue updates need a per-clinic-session `session_version` (monotonic, bumped in the same transaction as every queue change) plus doctor status and delay minutes. The table isn't designed yet. It will be added when the queue feature is planned (`/plan live-queue`), following ADR-0004.

### trusted_devices (ADR-0007)
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| doctor_id | uuid FK doctors | |
| device_id | uuid | **random**, generated by the app on first launch, kept in `expo-secure-store`. **No fingerprinting** |
| device_name | text | chosen by the doctor ("Desk PC") |
| desk_mode | bool | Desk mode on this device |
| desk_pin_set | bool | Desk Unlock PIN configured (the PIN hash lives in the device's secure storage) |
| last_active_at, revoked_at | timestamptz | revoking also revokes the device's refresh-token family |
| UNIQUE | (doctor_id, device_id) | |

### clinic_staff (⏸️ deferred to Phase 2, ADR-0007)
Separate receptionist logins aren't used in MVP. Revisit before prescriptions/reports. Draft: `id`, `user_id`, `clinic_id`, `doctor_id`, `permissions` (text[]), `invited_by`, `revoked_at`

### refresh_tokens
`id`, `user_id`, `family_id` (rotation chain), `token_hash` (SHA-256), `device_id`, `expires_at`, `used_at`, `revoked_at`, `replaced_by_id`

### otp_requests
`id`, `phone_hash`, `code_hash`, `purpose` (`Login`, `DeskRegistration`, ADR-0008), `clinic_id` (DeskRegistration only), `attempts`, `expires_at` (Login 5 min, DeskRegistration 10 min), `used_at` (single use), `ip_hash`, `created_at` (purged after 24 h). A `DeskRegistration` OTP **can't create an app session**

### consents
`id`, `user_id`, `purpose` (`Account`, `ShareWithDoctor`, `Reminders`, `Analytics`, `ParentalConsent`), `method` (`AppTap`, `DeskOtp`, ADR-0008), `actor_device_id` (desk device, if any), `clinic_id` (for `ShareWithDoctor`), `version`, `granted_at`, `withdrawn_at`

### audit_log (append-only, no UPDATE/DELETE grants)
`id` (bigserial), `at`, `actor_user_id`, `actor_role`, `action` (`READ`, `CREATE`, `UPDATE`, `DELETE`, `BREAK_GLASS`), `entity`, `entity_id`, `ip_hash`, `device_id`, `device_name`, `desk_mode` (bool, ADR-0007), `reason`, `prev_hash`, `row_hash` (hash chain for tamper detection)

## Row-Level Security (defence in depth)

```sql
ALTER TABLE appointments ENABLE ROW LEVEL SECURITY;
CREATE POLICY appt_patient ON appointments FOR SELECT
  USING (booked_by_user_id = current_setting('app.user_id')::uuid
         OR doctor_id = current_setting('app.user_id')::uuid);
```
The API sets `app.user_id` per request from the validated JWT. **Desk mode** restrictions (ADR-0007) are enforced in the API authorization layer per device: history and settings endpoints reject Desk-mode devices. They aren't just hidden in the UI.

## Migrations

- EF Core migrations in `DrBooking.Infrastructure/Migrations`.
- Every migration must be reviewed by the `db-reviewer` agent: reversible, no table locks on big tables, no data loss, encryption preserved.
- Seed data: fake only (`backend/tests/Fixtures`).
