# Workflows

Two parts: **(A) product workflows** (what users do) and **(B) dev workflow** (how we build with humans + AI agents).

---

## A. Product workflows

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
Patient opens Live queue ◀── SignalR QueueUpdated ── Doctor dashboard actions
      ▼                                                (check-in, next, skip, delay)
"You are #3 · ~20 min"  ─▶ push at #3 ("Leave now") ─▶ push at #1 ("You're next")
      ▼
Reception checks in ─▶ Doctor taps "Call next" ─▶ In consultation ─▶ Completed
```

### A3. Doctor onboarding

```
Sign up (phone OTP + MFA) ─▶ Profile ─▶ Add clinic (map pin)
 ─▶ Weekly timings + slot length ─▶ Upload registration proof
 ─▶ Status: Pending ─▶ Admin verifies against NMC/State council register
 ─▶ Verified ✔ ─▶ Listed in search ─▶ Share clinic booking link/QR with patients
```

### A4. Doctor cancels a session / goes on leave

```
Doctor adds leave ─▶ System finds affected appointments
 ─▶ Status CancelledByDoctor ─▶ Push + SMS "Dr X is unavailable on 16 Nov. Tap to rebook"
 ─▶ Deep link opens the slot screen at the next available date
```

### A5. Patient deletes their account (DPDP right to erasure)

```
Privacy centre ─▶ Delete account ─▶ OTP re-confirm ─▶ 7-day grace (can undo)
 ─▶ Job: wipe PII/PHI except what law requires us to keep (kept encrypted, access-restricted)
 ─▶ Revoke all tokens ─▶ Confirmation SMS
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
