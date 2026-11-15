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
  ├──< refresh_tokens
  ├──< consents
  └──< audit_log (append-only)
clinic_staff (user ↔ clinic + permissions)
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

### specialities
`id`, `name`, `slug`, `icon`, `sort_order`

### clinics
`id`, `name`, `address_line`, `area`, `city`, `pincode`, `lat`, `lng`, `phone`, `timezone` (default `Asia/Kolkata`)

### doctor_clinics
`id`, `doctor_id`, `clinic_id`, `slot_minutes` (e.g. 10), `patients_per_slot` (default 1), `booking_window_days` (7), `cancel_cutoff_minutes` (60), `consult_fee` (nullable), `is_active`

### clinic_sessions (weekly recurring timings)
`id`, `doctor_clinic_id`, `day_of_week` (0–6), `start_time` (local `time`), `end_time`, `label` (`Morning`/`Noon`/`Evening`, derived if null)

### doctor_leaves
`id`, `doctor_clinic_id` (null = all clinics), `start_at`, `end_at`, `reason_enc` 🔐

### appointments
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| booked_by_user_id | uuid FK users | RLS key |
| family_member_id | uuid FK family_members | the actual patient |
| doctor_id, clinic_id | uuid | |
| slot_start, slot_end | timestamptz | UTC |
| token_no | int | per doctor + clinic + local date |
| purpose | text | enum: `New`, `FollowUp`, `ShowReports`, `Vaccination`, `Emergency`, `Other` |
| purpose_note_enc 🔐 | bytea | optional free text |
| status | text | `Booked`, `CheckedIn`, `InConsultation`, `Completed`, `NoShow`, `CancelledByPatient`, `CancelledByDoctor` |
| source | text | `App`, `WalkIn` |
| idempotency_key | text | UNIQUE per booked_by_user_id |
| cancelled_at, checked_in_at, started_at, completed_at | timestamptz | |

**Constraints:**
```sql
-- one active booking per slot (when patients_per_slot = 1)
CREATE UNIQUE INDEX ux_appt_active_slot
  ON appointments (doctor_id, clinic_id, slot_start)
  WHERE status IN ('Booked','CheckedIn','InConsultation');

-- a patient can't double-book the same doctor on the same slot
CREATE UNIQUE INDEX ux_appt_member_slot
  ON appointments (family_member_id, doctor_id, slot_start)
  WHERE status IN ('Booked','CheckedIn','InConsultation');
```

### clinic_staff
`id`, `user_id`, `clinic_id`, `doctor_id`, `permissions` (text[]: `checkin`, `queue:manage`, `walkin:add`, `schedule:view`), `invited_by`, `revoked_at`

### refresh_tokens
`id`, `user_id`, `family_id` (rotation chain), `token_hash` (SHA-256), `device_id`, `expires_at`, `used_at`, `revoked_at`, `replaced_by_id`

### otp_requests
`id`, `phone_hash`, `code_hash`, `attempts`, `expires_at`, `ip_hash`, `created_at` (purged after 24 h)

### consents
`id`, `user_id`, `purpose` (`Account`, `ShareWithDoctor`, `Reminders`, `Analytics`, `ParentalConsent`), `version`, `granted_at`, `withdrawn_at`

### audit_log (append-only, no UPDATE/DELETE grants)
`id` (bigserial), `at`, `actor_user_id`, `actor_role`, `action` (`READ`, `CREATE`, `UPDATE`, `DELETE`, `BREAK_GLASS`), `entity`, `entity_id`, `ip_hash`, `device_id`, `reason`, `prev_hash`, `row_hash` (hash chain for tamper detection)

## Row-Level Security (defence in depth)

```sql
ALTER TABLE appointments ENABLE ROW LEVEL SECURITY;
CREATE POLICY appt_patient ON appointments FOR SELECT
  USING (booked_by_user_id = current_setting('app.user_id')::uuid
         OR doctor_id = current_setting('app.user_id')::uuid);
-- staff access is via a SECURITY DEFINER function that checks clinic_staff.permissions
```
The API sets `app.user_id` per request from the validated JWT.

## Migrations

- EF Core migrations in `DrBooking.Infrastructure/Migrations`.
- Every migration must be reviewed by the `db-reviewer` agent: reversible, no table locks on big tables, no data loss, encryption preserved.
- Seed data: fake only (`backend/tests/Fixtures`).
