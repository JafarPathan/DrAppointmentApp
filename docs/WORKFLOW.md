# Workflows

Two parts: **(A) product workflows** (what users do) and **(B) dev workflow** (how we build with humans + AI agents).

---

## A. Product workflows

### A0. The problem these workflows solve (small Indian clinic)

```
TODAY                                         WITH DRBOOKINGAPP
Patient walks in ─▶ tells name to reception   Patient books in app  ─┐
      ▼                                        OR walks in ─▶ desk adds by phone (registered: direct; new: OTP)
Name written in a register                           ▼
      ▼                                        Token in today's live queue (app + desk see same list)
Waits in clinic 1–2 h, doesn't know position         ▼
      ▼                                        Waits ANYWHERE: sees "You are #5 · ~25 min"
Receptionist shouts names in order                   ▼
      ▼                                        ONE app notification when 3 patients are ahead (SMS on hold)
Not there? Skipped / argument                        ▼
      ▼                                        Doctor/receptionist taps Call next / Not here → move back
Day overloads, doctor rushes                         ▼
                                               Doctor's daily limit reached ─▶ app shows "Full today"
```
Full problem statement: [PRD §1](PRD.md).

### A1. Patient books an appointment

```
Open app ─▶ Logged in? ──no──▶ Phone ─▶ OTP ─▶ Consent ─▶ Profile
              │yes
              ▼
           Home ─▶ Book appointment / Search
              ▼
        Search (name · speciality · clinic · city)
              ▼
        Doctor profile (verified, clinics, timings)
              ▼
        Pick a slot: Book for ▸ Date ▸ Clinic ▸ Session ▸ Slot ▸ Purpose
              ▼
        POST /appointments (Idempotency-Key)
          ├─ 201 ─▶ Confirmation (token #, time, directions) ─▶ reminders scheduled
          └─ 409 ─▶ "Someone just booked that slot" ─▶ refresh slots, keep the selections
```

### A2. Day of the visit (live queue)

```
Reminder push (evening before + 1 h before)
      ▼
Patient opens Live queue ◀── live update ── Desk actions (doctor or receptionist):
      ▼                                     check-in, call next, not here → move back, no-show, delay
"You are #6 · ~30 min"  (updates every time the queue moves)
      ▼
3 patients ahead ─▶ ONE app notification ("3 patients ahead of you. Please reach the clinic.")
      ▼                (sent once per visit, even if the patient is later moved back)
Next ─▶ app notification ("You're next")
      ▼
Reception checks in ─▶ Doctor taps "Call next" ─▶ In consultation ─▶ Completed
```
**Notification SMS is on hold for MVP** (maintainer decision, DEBATE Thread 003). Patients without the app get no alerts; they keep their token from the desk and wait at the clinic. The SMS fallback (ADR-0004) is built behind a feature flag to switch on later.

### A6. Patient at the desk (walk-in): maintainer decision, DEBATE Thread 009

```
Patient arrives at the clinic
      ▼
Doctor / receptionist taps "Add patient" ─▶ enters phone number ─▶ system checks: registered?
      ▼
 ┌──────── REGISTERED on DrBookingApp ────────┐        ┌──────── NOT REGISTERED ────────┐
 │                                            │        │                                │
 │ Seen at THIS clinic before?                │        │ OTP SMS to the patient's phone │
 │  ├ yes → pick from this clinic's past      │        │ (with link to privacy notice)  │
 │  │       patients / family for that phone  │        │            ▼                   │
 │  └ no  → desk types the name the patient   │        │ Patient reads OTP to the desk  │
 │          gives (no stored data shown)      │        │ (or taps link to confirm)      │
 │            ▼                               │        │            ▼                   │
 │ Added DIRECTLY to today's queue (no OTP)   │        │ Desk enters name, age, gender  │
 │            ▼                               │        │            ▼                   │
 │ App notification to patient:               │        │ Account created (phone         │
 │ "Dr X's clinic added you to today's queue. │        │ verified, consent recorded)    │
 │  Token 14. Not you / for family? Tap"      │        │            ▼                   │
 │                                            │        │ Added to today's queue         │
 └─────────────────────┬──────────────────────┘        └───────────────┬────────────────┘
                       └───────────────────┬───────────────────────────┘
                                           ▼
                Token shown on the desk screen (tell / write it for the patient)
                Patient with the app sees live position + gets the "3 ahead" notification
                Patient without the app: no alerts in MVP (SMS on hold), waits at clinic as today
                                           ▼
                Counts against today's capacity (A8)
                Desk screen shows the CLINIC QR ("Scan to track your token", ADR-0010)
                Patient scans ─▶ app opens (if installed) or landing page ─▶ store ─▶ install
                ─▶ log in with the same phone + OTP ─▶ app opens Live queue straight away
                (today's token is already there: the desk created the account on that phone
                number, or used the existing one). The QR holds no personal data.
```
🔒 Safeguards ([ADR-0008](decisions/ADR-0008-desk-patient-registration.md), agreed in Thread 009):
- The desk sees **no stored profile data** (name, family) for a registered patient **unless this clinic has seen them before**.
- The patient is notified in the app and can change or remove the entry.
- Phone lookups are rate-limited per clinic and audit-logged with the device.
- OTP codes are never stored in plain text.
- The desk can't see visits with other doctors.
- OTP SMS still works while notification SMS is on hold.

### A7. Running the queue (doctor or receptionist)

```
Today's list (app bookings + walk-ins, in token order)
      ▼
Call next ─▶ patients' apps updated live, everyone behind moves up
      ▼
Patient not there?
   ├─ "Not here → move back" (by N places, or to end of list)
   │     ▼ everyone's position + ETA updates; patient gets "You've been moved back, new position #8"
   ├─ "No-show" (after the session, or doctor decides) ─▶ frees capacity
   └─ Patient arrives late ─▶ desk taps "Arrived" ─▶ re-inserted (desk chooses where)
      ▼
Doctor running late? "Delay 20 min" ─▶ all waiting patients get an app notification (SMS on hold)
      ▼
Manual reorder (emergency, elderly, etc.) ─▶ drag in list; reason optional; logged in audit
      ▼
Patient now 3 ahead?
   ├─ Has the app ─▶ automatic app notification (free)
   └─ No app ─▶ no remote alert in MVP (SMS and WhatsApp on hold)
                 BUT the patient can install the app any time, even while waiting,
                 log in with the same phone + OTP, and immediately see today's token and live queue
```
Patients with the app can also open it or **pull to refresh** at any time to see their current position. WhatsApp reminder options are parked in DEBATE Thread 010.
Every queue change updates all patients' positions live and is recorded in the audit log (what changed, when, from which signed-in device).

### A8. Doctor sets daily capacity ("Full today")

```
Doctor ─▶ Clinic settings ─▶ per session or per day: "Max patients" (e.g. Morning 30, Evening 20)
      ▼
App bookings + walk-ins both count towards it
      ▼
Limit reached ─▶ Patient app shows "Full today" + "Next available: Tue 17 Nov"
             ─▶ Desk shows a warning "Today is full (30/30)"
             ─▶ Doctor (only) can override to add an emergency/extra patient
      ▼
Doctor can raise or lower the limit for today only (e.g. leaving early)
   └─ Lowering below the number already booked never cancels silently;
      affected patients are shown to the doctor to decide (keep, or cancel + notify)
```
Both booking styles are supported, and the doctor chooses per clinic ([ADR-0006](decisions/ADR-0006-token-queue-and-capacity.md)). In **Token queue** the limit is per session, and app bookings and walk-ins share it. The doctor's "extra" patient is flagged and doesn't count against the limit. In **Time slots** the limit is the sum of the slots; how walk-ins fit there is still open (needs a debate thread).

### A9. Single doctor-side login (doctor alone, or with a receptionist): maintainer decision, DEBATE Thread 008

```
Doctor signs up ─▶ ONE doctor-side login (phone OTP + second factor)
      ▼
 ┌─ Doctor alone ─────────────────────┐   ┌─ Doctor + receptionist ──────────────────────────────┐
 │ Doctor runs desk + consultations   │   │ Doctor signs in the desk device once, names it        │
 │ from phone or laptop               │   │ ("Desk PC") and sets a DESK PIN for the receptionist  │
 │                                    │   │            ▼                                          │
 │                                    │   │ Doctor switches the device to DESK MODE               │
 │                                    │   │  - Shows: Today list, add patient, check in,          │
 │                                    │   │    call next, move back, no-show, reorder             │
 │                                    │   │  - Hides + server blocks: patient history, settings,  │
 │                                    │   │    devices, "extra" token                             │
 │                                    │   │  - Locks after idle → receptionist enters DESK PIN    │
 │                                    │   │    (5 wrong → needs the Doctor PIN)                   │
 │                                    │   │  - Leaving Desk mode needs the private DOCTOR PIN     │
 │                                    │   │    (different from the Desk PIN)                      │
 └────────────────────────────────────┘   └──────────────────────────────────────────────────────┘
      ▼
"Signed-in devices" list ─▶ doctor can sign out any device instantly (e.g. receptionist leaves)
Every action is logged with the device it came from
```
Known trade-off, accepted for MVP: with one login, the log shows *which device*, not *which person*. Separate receptionist logins should be revisited **before Phase 2** adds prescriptions, reports or clinical notes. Design: [ADR-0007](decisions/ADR-0007-single-doctor-login-desk-mode.md).

### A3. Doctor onboarding

```
Sign up (phone OTP + MFA) ─▶ Profile ─▶ Add clinic (map pin)
 ─▶ Choose booking style (required, maintainer decision, Thread 007):
      [ Token queue: Recommended for most clinics ]  sessions + max patients per session
      [ Time slots ]                                 sessions + slot length + patients per slot
 ─▶ Weekly timings ─▶ Upload registration proof
 ─▶ Status: Pending ─▶ Admin verifies against NMC/State council register
 ─▶ Verified ✔ ─▶ Listed in search ─▶ "Print clinic QR" standee for reception (ADR-0010)
 ─▶ Optional: sign in the desk device, set Desk PIN + Doctor PIN, switch on Desk mode (ADR-0007)
```

### A4. Doctor cancels a session / goes on leave

```
Doctor adds leave ─▶ System finds affected appointments
 ─▶ Status CancelledByDoctor ─▶ App notification "Dr X is unavailable on 16 Nov. Tap to rebook" (SMS on hold)
 ─▶ Deep link opens the slot screen at the next available date
```

### A5. Patient deletes their account (DPDP right to erasure)

```
Privacy centre ─▶ Delete account ─▶ OTP re-confirm ─▶ 7-day grace (can undo)
 ─▶ Job: wipe PII/PHI except what law requires us to keep (kept encrypted, access-restricted)
 ─▶ Revoke all tokens ─▶ Confirmation (SMS on hold in MVP; shown in app before logout)
```

---

## B. Dev workflow (humans + AI agents)

```
 1. Pick an item from docs/PROGRESS.md
 2. Dev creates branch:  feature/<name>      ◀── AI agent must ASK the dev to do this
 3. /plan <feature>       → plan in docs/plans/<feature>.md (story, API, DB, screens, threats, tests)
 4. /design <screen>      → screen spec (if UI)
 5. Build (small commits, Conventional Commits, ask before committing)
 6. Agents recheck:
      code-reviewer  → bugs/logic
      security-auditor → auth, IDOR, PHI, crypto   (mandatory for sensitive areas)
      test-engineer  → write/run unit + integration + authz tests
      ui-reviewer    → design system + a11y        (UI only)
      db-reviewer    → migration safety            (DB only)
 7. /sync-docs (docs-keeper) → CHANGELOG, PROGRESS log, API/DB docs, pitch deck, video script
 8. /doctor               → full health check must be green
 9. Open PR (template checklist) → human review → squash merge to main
10. CI deploys to dev → staging → store internal testing → prod (manual approval)
```

`/feature <name>` runs steps 2–8 in order and stops for human approval at the branch, commit and PR steps.

### Release flow (mobile)

1. Bump version (semver) + build number.
2. `eas build --profile production` for Android (AAB) and iOS (IPA).
3. Internal testing track (Play Console) / TestFlight → smoke test on real devices.
4. Store listing: privacy policy URL, **Data safety form (Play)** and **App Privacy labels (Apple)** filled in to match SECURITY.md exactly.
5. Staged rollout 10% → 50% → 100% while watching the crash rate.
6. JS-only hotfixes via EAS Update. Anything native means a new store build.

### Store-review gotchas for health apps

- Apple: needs a privacy policy, account deletion inside the app, and a reviewer demo account with fake data. Medical claims must be backed up.
- Google Play: Health apps declaration + Data safety form. Explain the SMS permission if OTP auto-read uses it (prefer the SMS Retriever API, which needs no permission).
