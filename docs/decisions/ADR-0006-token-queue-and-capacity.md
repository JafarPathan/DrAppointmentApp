# ADR-0006: Booking styles (token queue / time slots), session capacity and queue ordering

- **Status:** Accepted
- **Date:** 2026-11-16
- **Debate:** [DEBATE.md Thread 007](../../DEBATE.md) (🟢 consensus between Claude Code and Antigravity + maintainer decision on modes)
- **Extends:** [ADR-0002](ADR-0002-slot-concurrency-inventory.md) (capacity claim), [ADR-0004](ADR-0004-resilient-queue-notifications.md) (`session_version`)

## Context

Small Indian clinics mostly run a **token queue**: patients register, get a number and are called in order (PRD §1). Some doctors (dentists, physios, specialists) work with **fixed time slots**. The maintainer wants:
- a per-day or per-session patient limit, with "Full today" in the app;
- walk-ins added at the desk;
- the doctor or receptionist able to move patients who aren't there.

ADR-0002 already guarantees no over-booking via `slot_inventory`. This ADR makes the capacity unit configurable and defines how the queue is ordered and changed.

## Decision

### 1. Two booking styles; the doctor chooses (maintainer decision)
`doctor_clinics.queue_mode` = `Token` | `TimeSlot`, **required at clinic setup** (no silent default). The setup screen marks Token as "Recommended for most clinics". Changing the mode applies to **future dates only**; existing bookings keep their mode.

| | Token mode | TimeSlot mode |
|---|---|---|
| Doctor sets | Sessions + `max_patients` per session | Sessions + `slot_minutes` + `patients_per_slot` |
| `slot_inventory` row = | one per **(doctor_clinic, session, date)**, `slot_start` = session start, `max_capacity` = session limit | one per **time slot** (as in ADR-0002) |
| Patient sees | "Morning · 12 of 30 left · your turn around 11:40" (estimate, never a promise) | Slot chips 11:00, 11:10, … |
| Booking result | Token + **estimated** time | Token + slot time |

ETA = (patients ahead × average consult minutes), with average consult minutes initially set by the doctor (default 10) and later learned (PROGRESS Phase 3).

### 2. Capacity, "Full today" and walk-ins
- In Token mode, **app bookings and walk-ins both claim the session's `slot_inventory` capacity** with the same atomic `UPDATE ... WHERE booked_count < max_capacity` (ADR-0002). When it's full: the patient app shows "Full today. Next available: <date>", and the desk shows "30/30 full".
- **Daily limit** = sum of the session limits. The doctor can change today's limits only. Lowering below current bookings follows ADR-0002 §6 (`max_capacity = GREATEST(new, booked_count)`, affected patients listed for the doctor).
- **Doctor override ("extra" token):** only the doctor (not Desk mode, ADR-0007) can add a patient beyond the limit. The appointment gets `is_over_capacity = true` and **does not touch `slot_inventory`**, so `CHECK (booked_count <= max_capacity)` always holds. The desk shows "30/30 + 1 extra". The patient app still shows "Full today". It's audit-logged.
- **Reconcile job** (ADR-0002 §7) counts only `is_over_capacity = false` active appointments.
- **Open (not debated):** how walk-ins work in **TimeSlot** mode (e.g. claim the next free slot today, or doctor override only). This needs a DEBATE thread **before** the TimeSlot walk-in screen is built. Token-mode walk-ins are fully decided above.

### 3. Token number vs queue position
- `appointments.token_no`: assigned once (ADR-0002 `daily_token_counters`), **never changes**. It's what's written on paper and shown to the patient.
- `appointments.queue_position` (**integer**): the current calling order within that session and date.
- **Renumbering:** any position change runs in one transaction: `SELECT ... FROM appointments WHERE <session, date> FOR UPDATE`, rewrite the positions 1..n, bump `session_version`. Sessions are small (≤ ~100 rows), so this is cheap. Integers avoid the precision exhaustion of fractional or floating ranks.

### 4. Absence and status rules
| Action | Effect |
|---|---|
| **Not here → move back** (by N, or to end) | Status `NotPresent`. Keeps token and capacity; `queue_position` moved |
| **Arrived** (after NotPresent) | Back to `CheckedIn`; desk chooses the position |
| **No-show** | Status `NoShow`; **frees capacity** (decrement in the same transaction, ADR-0002) |
| Check in / Call next / Start / Complete | Status transitions (below) |

Statuses: `Booked`, `CheckedIn`, `NotPresent`, `InConsultation`, `Completed`, `NoShow`, `CancelledByPatient`, `CancelledByDoctor`. Active (count towards capacity) = `Booked`, `CheckedIn`, `NotPresent`, `InConsultation`.

### 5. Concurrency between devices (doctor's phone + desk PC)
- **Position-dependent actions** (move back by N, drag-reorder) require **`If-Match: <session_version>`**. A mismatch returns **409**, and the UI refreshes, because the user acted on an order they saw.
- **Status transitions** (check in, call next, start, complete, no-show, not-here → end) **don't** use If-Match. They're guarded updates: `UPDATE appointments SET status=@to ... WHERE id=@id AND status=@from`, with 0 rows → 409 only if *that patient* already changed.
- **Call next** picks the next patient **server-side** (lowest `queue_position` with status `CheckedIn`) under the session row lock, so two devices pressing "Call next" together never call the same patient.
- Every mutation bumps `session_version` (ADR-0004) and writes `audit_log` with the device (ADR-0007).

## Alternatives considered

| Option | Why not |
|---|---|
| Time slots only | Forces fake times on token clinics; walk-ins don't fit |
| Token only | Excludes doctors who need fixed appointments |
| Hybrid (slots + walk-in pool per session) | Too complex for MVP |
| Fractional / float `queue_position` | Precision runs out after repeated "move between"; harder to read |
| `If-Match` on every mutation | Desk would get 409s whenever any booking arrives during the rush |
| Over-capacity tokens counted in `booked_count` | Violates ADR-0002's `CHECK`, or causes reconcile drift |

## Consequences

- ✅ One booking engine for both styles, no new concurrency mechanism.
- ✅ The desk rarely sees conflicts; reorders stay safe.
- ⚠️ ETA is an estimate; UI copy must never present it as a promise.
- ⚠️ TimeSlot-mode walk-ins need a short follow-up thread.

## Required tests

1. Token mode: N parallel claims (app + desk) on a session of capacity k → exactly k succeed.
2. Doctor override → appointment created with `is_over_capacity`, `booked_count` unchanged, reconcile reports no drift.
3. Two devices "Call next" at once → two different patients, never the same.
4. Reorder with a stale `If-Match` → 409; with the current version → success, positions contiguous 1..n.
5. Status transition racing a booking arrival → no 409 for the transition.
6. NoShow → capacity freed and bookable. NotPresent → capacity kept.
7. Mode switch → existing bookings unchanged; new dates use the new mode.

## Docs to update

- `docs/DATABASE.md`: `doctor_clinics.queue_mode`, `max_patients` per session, `avg_consult_minutes`; `appointments.queue_position` (int), `is_over_capacity`, status `NotPresent`; resolve the open walk-in note.
- `docs/API.md`: desk queue actions (If-Match on reorder only).
- `docs/ARCHITECTURE.md` §4: booking styles + session capacity.

## Sources

- ADR-0002 (this repo).
- PostgreSQL, Explicit Locking: https://www.postgresql.org/docs/current/explicit-locking.html
- Martin Fowler, Optimistic Offline Lock: https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html
- arXiv 2026 OPD preprint (token FCFS prevalence): https://arxiv.org/pdf/2604.00215
- NHSRC Tamil Nadu OPD re-engineering: https://qps.nhsrcindia.org/sites/default/files/2021-05/Addressing%20Overcrowding%20in%20OPD%20by%20Process%20Mapping%20and%20Process%20Re-Engineering%20-%20Tamil%20Nadu.pdf
