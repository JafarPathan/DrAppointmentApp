# ADR-0002: Slot concurrency via lazily materialized slot inventory

- **Status:** Accepted
- **Date:** 2026-11-16
- **Debate:** [DEBATE.md Thread 001](../../DEBATE.md) (🟢 consensus between Claude Code and Antigravity)
- **Supersedes:** the double-booking approach in ARCHITECTURE.md §4 steps 4–5 (partial unique index only / SERIALIZABLE capacity check)

## Context

A doctor can allow more than one patient per slot (`doctor_clinics.patients_per_slot`). The original design enforced "one active booking per slot" with a partial unique index, which only works when capacity = 1. For capacity > 1 it fell back to a SERIALIZABLE count check. In PostgreSQL that raises `40001` serialization failures under contention, and the application has to retry the whole transaction. During morning booking rushes that means retry storms and slower, flakier booking.

Requirements:
- **Never over-book**, for any capacity ≥ 1.
- No retry loops on the booking path.
- Correctness must not depend on a background job having run.
- Business rules stay visible and testable in C#, not hidden in the database.

## Decision

### 1. New table `slot_inventory`, created lazily at booking time

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| doctor_clinic_id | uuid FK `doctor_clinics` | |
| slot_start, slot_end | timestamptz | UTC |
| max_capacity | int | copied from `patients_per_slot` when the row is created |
| booked_count | int | default 0, `CHECK (booked_count >= 0 AND booked_count <= max_capacity)` |
| status | text | `Open`, `Blocked` |
| version | int | incremented on every change |
| created_at, updated_at | timestamptz | |

`UNIQUE (doctor_clinic_id, slot_start)`. No PHI/PII is stored here.

### 2. New table `daily_token_counters`

`doctor_clinic_id`, `local_date` (date in the clinic's time zone), `next_token_no` (int). `PRIMARY KEY (doctor_clinic_id, local_date)`.

### 3. `appointments.slot_inventory_id`

New FK column. `ux_appt_member_slot` (one active booking per family member per doctor per slot) is **kept**. `ux_appt_active_slot` is dropped, because capacity is now enforced by `slot_inventory`.

### 4. Booking transaction (Read Committed, one transaction, in `BookAppointmentCommandHandler`)

1. **Validate on the server** that the requested `slot_start` belongs to the doctor's schedule for that clinic, is in the future, is inside the booking window, and isn't on leave. A client can never create an arbitrary inventory row.
2. **Lazy upsert:**
   ```sql
   INSERT INTO slot_inventory (id, doctor_clinic_id, slot_start, slot_end, max_capacity, status)
   VALUES (@id, @dc, @start, @end, @cap, 'Open')
   ON CONFLICT (doctor_clinic_id, slot_start) DO NOTHING;
   ```
3. **Atomic capacity claim:**
   ```sql
   UPDATE slot_inventory
      SET booked_count = booked_count + 1, version = version + 1, updated_at = now()
    WHERE doctor_clinic_id = @dc AND slot_start = @start
      AND status = 'Open' AND booked_count < max_capacity
   RETURNING id;
   ```
   0 rows → roll back and return **409 Conflict** ("Someone just booked that slot").
   Under Read Committed, a concurrent updater waits on the row lock, then re-evaluates the `WHERE` clause against the newly committed row. If the slot is now full, it updates 0 rows.
4. **Token number:**
   ```sql
   INSERT INTO daily_token_counters (doctor_clinic_id, local_date, next_token_no)
   VALUES (@dc, @date, 1)
   ON CONFLICT (doctor_clinic_id, local_date)
   DO UPDATE SET next_token_no = daily_token_counters.next_token_no + 1
   RETURNING next_token_no;
   ```
   Never use `MAX(token_no) + 1`, because it races.
5. **Insert the appointment** with `slot_inventory_id` and `token_no`, then commit.
6. **After commit:** publish the SignalR update and schedule reminders (see ADR-0004).

### 5. Cancellation and status changes

`CancelAppointmentCommandHandler` (patient or doctor) and the no-show/complete handlers decrement `booked_count` only when the appointment leaves an active status. They do this in the **same transaction** as the status change. **No database triggers.**

### 6. Leaves and capacity reductions

When a doctor adds leave, removes a session or lowers `patients_per_slot` on slots that already have bookings:
- Set the affected `slot_inventory` rows to `Blocked`. New bookings fail the `status = 'Open'` check.
- Route existing appointments in those slots through the cancel-and-notify flow (WORKFLOW A4). **Never silently over-book or silently drop.**
- Lowering `patients_per_slot` sets `max_capacity = GREATEST(new_capacity, booked_count)` on existing future inventory rows, so the `CHECK` constraint always holds. Slots where `booked_count` exceeds the new capacity keep their bookings and are listed for the doctor to resolve (keep, or cancel and notify). New rows use the new capacity.

### 7. Drift protection

Because `booked_count` is denormalized, a **nightly Hangfire job** compares it with `COUNT(appointments WHERE status IN ('Booked','CheckedIn','InConsultation'))` per slot. It alerts on any mismatch, and repairs it only after a human acknowledges the alert.

### 8. Reads

Slot lists are still **computed** from `clinic_sessions` minus `doctor_leaves`, then overlaid with `slot_inventory` counts where rows exist. They're cacheable per `(doctor_clinic_id, local_date)` and invalidated on booking, cancel or schedule change. An optional rolling 14-day pre-generation job may create inventory rows ahead of time **for read performance only**. Correctness never depends on it.

## Alternatives considered

| Option | Why not |
|---|---|
| A. Partial unique index only | Supports capacity = 1 only |
| B. SERIALIZABLE + count check | `40001` failures need retries of the whole transaction; retry storms at peak |
| C. Advisory lock + count | Locks are invisible in the schema and easy to forget on a new code path; no durable capacity record |
| D. Eagerly materialized slots (mandatory generator job) | Booking breaks or shows no slots if the job fails or lags; needs regeneration on every schedule edit |
| DB trigger maintaining `booked_count` | Hides business logic from C#, harder to test, possible lock-ordering surprises |

## Consequences

- ✅ Any capacity works the same way. Losers get a clean 409 with no retries. There's a durable per-slot capacity record.
- ✅ Booking doesn't depend on any background job.
- ⚠️ `booked_count` is denormalized, so it's guarded by single-transaction handlers and the nightly reconcile.
- ⚠️ The schedule-edit handlers get more complex (blocking + cancel-and-notify).

## Required tests (blocking for implementation)

1. **Concurrency integration test** (Testcontainers PostgreSQL): N parallel booking requests (e.g. 50) on one slot with capacity k (1, 2, 3) → **exactly k succeed**, the rest get 409, and `booked_count = k`. This is required because the PostgreSQL documentation of Read Committed re-evaluation has been questioned. We prove the behaviour with a test rather than trusting the docs alone.
2. Parallel bookings get unique, gap-free tokens per clinic per day.
3. Cancel → `booked_count` decrements, and a freed slot is bookable again.
4. Leave added on a booked slot → row `Blocked`, appointments cancelled and notified, no new bookings accepted.
5. Booking a `slot_start` that isn't in the schedule → 400/404, and no inventory row is created.
6. Same family member double-booking the same slot → rejected by `ux_appt_member_slot`.
7. The reconcile job detects an artificially introduced drift.
8. A load test (k6 or NBomber) with ~200 concurrent bookings on one slot, run in PROGRESS Phase 1.

## Docs to update after this ADR (next step, per AGENTS.md Rule 0.5)

- `docs/DATABASE.md`: add `slot_inventory` and `daily_token_counters`; add `appointments.slot_inventory_id`; replace `ux_appt_active_slot`.
- `docs/ARCHITECTURE.md` §4: replace the booking flow with §4 of this ADR.
- `docs/PROGRESS.md`: concurrency test + load test under item 1.6.

## Sources

- PostgreSQL, Transaction Isolation (Read Committed): https://www.postgresql.org/docs/current/transaction-iso.html
- PostgreSQL, Serialization Failure Handling: https://www.postgresql.org/docs/16/mvcc-serialization-failure-handling.html
- PostgreSQL, Explicit Locking: https://www.postgresql.org/docs/current/explicit-locking.html
- pgsql-hackers, Read Committed docs discussion: https://www.postgresql.org/message-id/CAJ7c6TMBTN3rcz4%3DAjYhLPD_w3FFT0Wq_C15jxCDn8U4tZnH1g%40mail.gmail.com
- pgsql-bugs #18750: https://www.postgresql.org/message-id/e8839c8.226.193fc4d0c40.Coremail.yrshen@stu.xidian.edu.cn
