# ADR-0008: Adding and registering patients at the desk

- **Status:** Accepted (engineering). **Legal sign-off pending** on consent (see Open items)
- **Date:** 2026-11-16
- **Debate:** [DEBATE.md Thread 009](../../DEBATE.md) (maintainer decision + mitigations, 🟢 consensus between Claude Code and Antigravity)
- **Related:** ADR-0006 (capacity), ADR-0007 (Desk mode, device audit), Thread 002 (anti-enumeration), Thread 005 (minors)

## Context

Many patients walk in without the app. The desk (doctor, or receptionist in Desk mode) must add them to today's queue by phone number. The maintainer decided:
- **Registered phone** → add **directly** to the queue, no OTP; the patient gets the update in the app.
- **New phone** → send an **OTP** and register them as a new patient.

## Decision

### 1. Phone check
`POST /desk/patients/check` with `{ phone }`:
- Looks up `users.phone_hash` (HMAC blind index, SECURITY §4).
- Returns `{ registered: bool, knownToThisClinic: [{ familyMemberId, displayName }] }`.
- `knownToThisClinic` lists **only** patients/family members on that phone **who have visited this clinic before**. For a registered patient new to this clinic, the list is **empty**: the desk sees **no stored name, family or history from other clinics**.
- **Rate-limited per clinic** (starting value: alert after 30 checks/hour that don't lead to a queue add; tunable) and **audit-logged** with `device_id` (ADR-0007).

> Accepted trade-off (maintainer decision): because registered numbers skip the OTP step, the desk learns whether a number is on the platform. Thread 002's anti-enumeration rule still applies to every **other** lookup.

### 2. Registered patient → direct add
`POST /desk/queue` with `{ phone, familyMemberId? | displayName, sessionId }`:
- Claims capacity (ADR-0002/0006) and assigns a token.
- If not known to this clinic, the desk types the name the patient gives (`displayName`). It's stored on the appointment for this clinic's view only, and never overwrites the patient's profile.
- **App notification** to the account: "Dr <name>'s clinic added you to today's queue. Token 14. Not you, or for a family member? Tap to change." The patient can **reassign it to a family member or remove it**. Removal frees capacity and is audit-logged.

### 3. New patient → scoped OTP registration
1. `POST /desk/patients/otp` with `{ phone }` sends an **OTP SMS** (OTP SMS stays on; only notification SMS is on hold):
   > "Your OTP for registering at <clinic> on DrBookingApp is 482913. Sharing it means you agree to our privacy notice: <short link>"
2. The patient **reads the OTP to the desk** (or taps the link to confirm on their phone).
3. `POST /desk/patients/verify` with `{ phone, code, name, age, gender, sessionId }` creates the account (phone verified), a **`Self` family member**, a **consent record**, and the queue entry.

**OTP scoping:** `otp_requests.purpose = DeskRegistration`, bound to the clinic, **10 min**, **single use**, stored **hashed**, attempt-limited. It **can't create an app session**. The patient later logs into the app with their own fresh OTP.

**Consent record:** `consents.purpose = Account` (+ `ShareWithDoctor` for this clinic), `method = DeskOtp`, `actor_device_id`, notice version.

**Minors:** register under a parent's phone as a family member (Thread 005 rules apply).

### 4. After registration
The desk shows the token and tells the patient they can **install the app and log in with the same number** to track it live (PRD §4, ADR-0010 clinic QR).

## Alternatives considered

| Option | Why not |
|---|---|
| OTP for everyone, desk never learns registration status (Claude's original) | Maintainer chose less friction for returning patients |
| No OTP for new patients | No proof the phone works or that the person received the notice |
| Show the stored profile of any registered phone | Leaks names and family across clinics |
| Patient must self-register in the app | Excludes elderly and first-time patients |

## Consequences

- ✅ Returning patients are added in seconds; new patients are verified and consented.
- ✅ No cross-clinic data shown to the desk.
- ⚠️ The desk learns platform membership (accepted). Mitigated by rate limits and audit.
- ⚠️ The receptionist hears the OTP (scoped, single-use, can't log in).

## Required tests

1. `check` for a phone registered elsewhere → `registered: true`, `knownToThisClinic: []`, no name returned.
2. Rate limit triggers an alert. Every check is audit-logged with the device.
3. Direct add → notification sent. Patient "remove" → capacity freed + audited.
4. Desk OTP can't be used on the app login endpoint. Expired or reused → rejected.
5. Verify creates the account + Self member + consent with `method = DeskOtp`.

## Open items

- **Legal review** (with Threads 005, 006): is a returning patient asking at the desk, plus the in-app notice and undo, sufficient for sharing with a new doctor? Is OTP read-aloud after a notice link sufficient consent evidence?
- Final DLT registration of the OTP template happens when SMS goes live.

## Docs to update

- `docs/API.md`: rename `/desk/patients/lookup` → `/desk/patients/check`; confirm the other desk endpoints.
- `docs/DATABASE.md`: `otp_requests.purpose`, `clinic_id`; `consents.method`, `actor_device_id`; `appointments.display_name_enc`.
- `docs/SECURITY.md` §2, §8: desk OTP + consent method.

## Sources

- DPDP Act s.5–6 (notice, consent), secondary summaries: https://www.dsci.in/files/content/documents/2025/consent-management.pdf · https://dpdp.myndsolution.com/wiki/act/section-6-consent/
- CipherSweet (blind index leakage): https://ciphersweet.paragonie.com/security
