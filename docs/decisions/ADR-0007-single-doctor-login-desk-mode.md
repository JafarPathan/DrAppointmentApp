# ADR-0007: Single doctor-side login with trusted devices and Desk mode (MVP)

- **Status:** Accepted
- **Date:** 2026-11-16
- **Debate:** [DEBATE.md Thread 008](../../DEBATE.md) (maintainer decision + mitigations, 🟢 consensus between Claude Code and Antigravity)
- **Revisit:** **before Phase 2** adds prescriptions, reports or clinical notes (PROGRESS 2.3)

## Context

Small clinics have either one doctor doing everything, or a doctor plus one receptionist. The maintainer decided the doctor side has **one login** for MVP, used by the doctor and, where there is one, the receptionist.

Risks of a single login, accepted by the maintainer but to be kept small:
- The audit log can't tell *which person* acted.
- A receptionist who leaves can't be removed separately.
- The desk could see everything the doctor sees.

## Decision

### 1. One doctor account, signed in on trusted devices
- The doctor signs in **once per device** with phone OTP **+ second factor** (SECURITY §2).
- On first launch the app generates a **random `device_id`** (UUID) and stores it in **`expo-secure-store`** (iOS Keychain / Android Keystore). **No hardware or browser fingerprinting** (unstable, a privacy concern, restricted by Apple).
- The device is registered in `trusted_devices` with a **name** the doctor chooses ("Desk PC", "Dr phone"). The refresh token is bound to that `device_id`.
- **Signed-in devices** screen: list, rename, **sign out any device instantly** (revokes its refresh token family). This is how a doctor removes a receptionist who leaves.
- *Later hardening (not MVP):* a device-bound key pair in secure hardware, so a refresh token only works with that device's signature.

### 2. Desk mode
A per-device mode, switched on by the doctor:

| Shown in Desk mode | Hidden in Desk mode |
|---|---|
| Today's queue list, add patient (ADR-0008), check in, call next, not here / move back, no-show, reorder | Patient history and past visits, visit purpose notes, clinical data (Phase 2), clinic settings, capacity and timings, devices list, doctor override "extra" token (ADR-0006) |

### 3. Two separate PINs
| PIN | Who knows it | Unlocks | Storage | Attempt limit |
|---|---|---|---|---|
| **Desk Unlock PIN** | Doctor + receptionist | Unlocks **that device** into **Desk mode only**, after the idle lock (e.g. 15 min) | Per device, salted hash in secure storage | 5 failures → device locked until the **Doctor PIN** is entered |
| **Doctor PIN** | Doctor only | Exits Desk mode; doctor-only actions on any device | Account-level, **Argon2id hash on the server** | 5 failures → full re-auth with OTP + second factor |

The two PINs **must differ** (enforced at setup). The receptionist never learns the Doctor PIN or the doctor's OTP or second factor.

### 4. Audit
Every action writes `audit_log` with `actor_user_id` (the doctor account), **`device_id` and `device_name`**, and whether the device was in Desk mode. The patient-facing access log (PRD §4, Privacy centre) shows "Dr X's clinic (desk)" vs "Dr X".

### 5. Phase 2 condition
Before prescriptions, reports or clinical notes are built, separate receptionist logins (`clinic_staff`, per-person credentials and permissions) must be revisited in a new DEBATE thread. That data is far more sensitive than a queue list.

## Alternatives considered

| Option | Why not (for MVP) |
|---|---|
| Separate receptionist login (Claude's original Option B) | Maintainer chose simplicity for MVP; revisit in Phase 2 |
| Literal shared login with no mitigations | No revocation per device, desk sees patient history |
| One PIN for unlock and exit | The receptionist would know the PIN that exits Desk mode, defeating it |
| Device fingerprinting | Unstable, a privacy concern, restricted by Apple App Store rules |

## Consequences

- ✅ Zero extra setup for single-doctor clinics; one tap to hand the desk over.
- ✅ Desk can't see patient history; any device can be revoked instantly.
- ⚠️ Audit identifies the **device**, not the person (accepted for MVP).
- ⚠️ Exiting Desk mode needs network access (Doctor PIN checked server-side).

## Required tests

1. Desk mode hides history/settings in the UI **and** the API rejects those calls from a Desk-mode device (server-side check, not just hidden buttons).
2. Desk Unlock PIN can't exit Desk mode. Same-PIN setup is rejected.
3. 5 wrong Desk PINs → Doctor PIN required. 5 wrong Doctor PINs → OTP + second factor required.
4. Revoking a device → its next API call is 401 and its refresh fails.
5. Audit rows carry `device_id`, `device_name` and the Desk-mode flag.

## Docs to update

- `docs/SECURITY.md` §2–3: two PINs, random device ID, server-side Desk-mode enforcement.
- `docs/DATABASE.md`: `trusted_devices`; `clinic_staff` marked deferred; `audit_log` device columns; doctor PIN hash.
- `docs/API.md`: devices and Desk-mode endpoints.

## Sources

- Apple App Store Review Guidelines (§5.1.2, data use / fingerprinting): https://developer.apple.com/app-store/review/guidelines/
- Expo SecureStore: https://docs.expo.dev/versions/latest/sdk/securestore/
- NIST SP 800-63B (rev 3; §5.2.2 rate limiting, §7.2 re-authentication): https://pages.nist.gov/800-63-3/sp800-63b.html
- Shared-login risks (secondary): https://nhimg.org/glossary/shared-login/
