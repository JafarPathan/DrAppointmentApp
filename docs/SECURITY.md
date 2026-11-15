# Security & Privacy

> # **🔒 PRIVACY IS OUR TOP PRIORITY. THE APP MUST NOT BE HACKED AND PATIENT DATA MUST NEVER LEAK.**
> **Every feature is designed, reviewed and tested against this document. A security issue blocks a release, no exceptions.**

Standards we follow: **OWASP ASVS Level 2** (API/web), **OWASP MASVS** (mobile), **OWASP API Security Top 10**, India's **Digital Personal Data Protection (DPDP) Act 2023 + DPDP Rules 2025**, and the **Telemedicine Practice Guidelines** (for video consults, Phase 2).

---

## 1. Data classification

| Class | Examples | Protection |
|---|---|---|
| **Restricted (health / PHI)** | Visit purpose, symptoms, notes, reports, prescriptions, doctor–patient link | Field-level encryption + strict ownership checks + audit every read |
| **Confidential (PII)** | Name, phone, DOB, gender, address, family relations | Encrypted at rest. Phone stored encrypted + as a keyed hash (HMAC) for lookup |
| **Internal** | Clinic schedules, slot config | Normal auth |
| **Public** | Verified doctor's public profile, clinic address | Readable without login (rate-limited) |

**Data minimisation:** collect only what a feature needs. No Aadhaar number. No full address for patients. DOB only where age matters.

## 2. Authentication

| Item | Rule |
|---|---|
| Patient login | Phone + 6-digit OTP. OTP valid 5 min, max 5 tries, then 15-min lockout. OTP stored **hashed**. Rate limit per phone + per IP + per device |
| Doctor/staff login | Phone OTP **+ second factor** (TOTP authenticator app or email OTP) for dashboard access |
| **Single doctor-side login (MVP), [ADR-0007](decisions/ADR-0007-single-doctor-login-desk-mode.md)** | Doctor and receptionist share one doctor login. **Accepted trade-off:** the audit log identifies the *device*, not the *person*. Mitigations: **random `device_id`** in secure storage (no fingerprinting); named trusted devices with device-bound refresh tokens; "Signed-in devices" list with instant sign-out; **Desk mode** (hides patient history, settings and devices; **enforced server-side**). **Two different PINs:** a *Desk Unlock PIN* (shared with the receptionist, unlocks Desk mode only, 5 failures → Doctor PIN) and a *Doctor PIN* (private, Argon2id on the server, exits Desk mode, 5 failures → OTP + second factor). The two PINs must differ. **Revisit separate receptionist logins before Phase 2 adds medical records** |
| Desk patient check, [ADR-0008](decisions/ADR-0008-desk-patient-registration.md) | `POST /desk/patients/check`. Registered phone → added without OTP, so the desk learns platform membership (accepted by maintainer). Stored names and family members shown **only if this clinic has seen the patient before**. Patient notified in app with "Not me" undo. Checks rate-limited per clinic and audit-logged with device. New phone → **scoped `DeskRegistration` OTP** (clinic-bound, 10 min, single use, hashed, **can't create an app session**); consent recorded with `method = DeskOtp` |
| Admin login | SSO/TOTP MFA required, IP allow-list, short session (30 min idle) |
| Access token | JWT, **10-minute life**, signed with an asymmetric key (RS256/ES256) from Key Vault, `aud`/`iss` validated, minimal claims (user id, role). **No PII inside the JWT** |
| Refresh token | Opaque random 256-bit value, stored **hashed** server-side, 30-day life, **rotated on every use**. Reuse of an old token revokes the whole token family (theft detection) |
| Mobile storage | Refresh token in `expo-secure-store` (Keychain/Keystore). Access token in memory only |
| Web storage | Refresh token in an `HttpOnly; Secure; SameSite=Strict` cookie. Access token in memory. CSRF token for cookie-auth endpoints |
| Logout | Revoke the refresh token family server-side. "Log out of all devices" option |
| Device binding | Refresh token tied to a device id. New device login → push/SMS alert to the user |
| Biometric lock | Optional local app lock (doesn't replace server auth) |

## 3. Authorization

1. **Deny by default:** global `[Authorize]` fallback policy. Public endpoints are explicitly `[AllowAnonymous]` and listed in API.md.
2. **Role policies:** `Patient`, `Doctor`, `ClinicStaff`, `PlatformAdmin`.
3. **Resource-based checks on every request** (this is where most health apps get hacked: IDOR/BOLA):
   - Patient → only own `user_id` and linked `family_member` rows.
   - Doctor → only appointments where `doctor_id = me`.
   - Staff → only clinics they're assigned to, and only the permissions granted (`queue:manage`, `checkin`, ...). **Never** medical notes.
   - Admin → no medical data by default. **Break-glass** access requires a reason, is time-limited, is audited and notifies the security owner.
4. **IDs:** public IDs are random UUIDv7/ULID, never sequential integers.
5. **Defence in depth:** PostgreSQL **Row-Level Security** on PHI tables, keyed on the session user id set per request.

## 4. Encryption

| Where | How |
|---|---|
| In transit | TLS 1.2+ only (prefer 1.3), HSTS, no plain HTTP. **Certificate/public-key pinning** in the mobile app (with a backup pin) |
| At rest (DB, disks, backups) | Azure-managed encryption (AES-256), customer-managed key in Key Vault |
| Field level | **AES-256-GCM envelope encryption**: data key per tenant/table, wrapped by a Key Vault master key. Implemented once in `IFieldEncryptor`. Used for name, phone, DOB, visit purpose, notes and any PHI |
| Searchable fields | Phone → HMAC-SHA256 blind index (separate key) for exact-match lookup |
| Files (reports, ID proofs) | Encrypted blob storage, private container, short-lived (≤5 min) read-only SAS URLs generated only after an ownership check |
| Passwords (if ever added) | Argon2id |
| Key rotation | Master keys rotated yearly or on incident. Data keys re-wrapped, no downtime |

**Never** write custom crypto primitives. Use .NET `AesGcm` and Azure Key Vault SDKs only, wrapped in our one service.

## 5. API protection

- Rate limiting (ASP.NET Core rate limiter + WAF): OTP endpoints are strict, search is moderate.
- Input validation on every DTO (FluentValidation), max lengths, allow-lists for enums.
- Output DTOs only. **Never** return EF entities (avoids leaking fields).
- `Idempotency-Key` on booking/payment POSTs.
- Security headers on web: CSP, `X-Content-Type-Options`, `Referrer-Policy`, `frame-ancestors 'none'`.
- CORS: explicit allow-list of our web origins only.
- **Clinic QR / public web ([ADR-0010](decisions/ADR-0010-clinic-qr-app-links.md))**: QR links carry only the clinic's random, rotatable `public_code`, **never** a phone, name, token or appointment id. The landing page and app-association files live on a **separate static domain** from the API (no shared cookies or origins), with no cookies, analytics or third-party scripts. No third-party deep-link SDKs.
- **OpenAPI ([ADR-0005](decisions/ADR-0005-monorepo-and-client-pipeline.md))**: API docs UI is only in local/dev. Builds and CI never need secrets (`BuildContext.IsOpenApiGeneration` skips Key Vault and DB at build time).
- Generic error messages to clients (ProblemDetails). Details go to server logs without PHI.
- Pagination limits (max 50) to stop bulk scraping. Doctor search is public but rate-limited and returns no patient data.

## 6. Mobile app hardening (MASVS)

- No secrets in the app bundle (anything in a JS bundle is public).
- Certificate pinning. Block cleartext traffic (Android `usesCleartextTraffic=false`, iOS ATS on).
- Hide sensitive screens in the app switcher (blur) and block screenshots on PHI screens (Android `FLAG_SECURE`).
- Root/jailbreak detection → show a warning (don't block outright; log a risk signal).
- Disable debug logs in release builds. ProGuard/R8 + Hermes bytecode.
- Deep links validated. No tokens in URLs.
- Clear the cache and secure store on logout.

## 7. Logging, audit and monitoring

- **Audit log** (append-only table, hash-chained): who read or changed which PHI record, when, from which device/IP. Patients can view their own access history.
- **PHI scrubber** in Serilog drops or masks names, phones and free text.
- Alerts: login spikes, OTP abuse, 403 spikes (IDOR probing), refresh-token reuse, admin break-glass use.
- Log retention: at least **1 year** for security logs (DPDP Rules requirement).

## 8. DPDP Act & Rules compliance checklist

> Check the official Gazette text before launch. Commencement dates for the main obligations were phased in after notification (Nov 2025), and commentators disagree on exact dates. Treat everything below as required by launch anyway.

- [ ] **Clear consent notice** listing exact data fields + purpose for each, in plain language (+ Hindi/Marathi later)
- [ ] Separate consent for: account, sharing with a chosen doctor, reminders, optional analytics
- [ ] Withdraw consent as easily as giving it → triggers deletion unless the law requires retention (e.g. medical records rules)
- [ ] **Verifiable parental consent** for users under 18. Family members who are minors are managed by a parent account
- [ ] Data principal rights in-app: access, correction, erasure, grievance, nominee
- [ ] Grievance officer contact published in app + website
- [ ] **Breach response: notify the Data Protection Board within 72 hours** and affected users without delay (runbook in §10)
- [ ] Reasonable security safeguards (this whole document) + logs retained 1 year
- [ ] Data processing agreements with every processor (cloud, SMS, push, video, payments)
- [ ] Retention schedule: delete inactive accounts after a defined period, after notifying the user

## 9. Secure development process

| Stage | Control |
|---|---|
| Design | Threat model (STRIDE) for every new feature touching PHI, in its plan doc |
| Code | `security-auditor` agent + human review for sensitive areas |
| Commit | Pre-commit secret scan (`.githooks/pre-commit`). Add gitleaks in CI |
| CI | `dotnet list package --vulnerable`, `npm audit`, CodeQL/Semgrep, dependency review |
| Test | Automated authz tests: every endpoint × every role × "someone else's id" must return 403/404 |
| Release | OWASP ZAP baseline scan on staging. **Third-party pen test before public launch** |
| Ops | Least-privilege cloud roles, no shared accounts, MFA everywhere, prod access only via just-in-time elevation |

## 10. Incident / breach runbook (summary)

1. **Contain:** revoke tokens, rotate keys, block the attacker IP/device, take the affected endpoint offline if needed.
2. **Assess:** what data, how many users, from the audit logs.
3. **Notify:** Data Protection Board within 72 h. Affected users without delay, in plain language, with what they should do.
4. **Fix + post-mortem** (blameless) → ADR + tests so it can't recur.

## 11. Security owner

Name one person as security owner before launch: ______. They approve break-glass, own the runbook and sign off on releases.
