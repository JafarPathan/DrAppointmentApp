# Product Requirements (PRD)

## 1. Problem

**Patients** in Indian cities call clinics, get no answer or a vague "come at 11", then wait 1–2 hours in a crowded waiting room.
**Doctors at small and mid-size clinics** run their queue on paper or WhatsApp. They lose patients to no-shows and can't see tomorrow's load.
Big platforms are built for hospitals and charge high listing fees. Smaller clinics need something simple, cheap and trustworthy with patient data.

## 2. Goal

- A patient can book a confirmed slot with a doctor **in under 60 seconds** and see a live queue position, so they arrive when it's actually their turn.
- A doctor can set up their clinic **in under 10 minutes** and run the day's queue from a phone or laptop.
- **Patient data is never exposed.** Privacy is the brand.

## 3. Users and roles

| Role | Description | Key permissions |
|---|---|---|
| Patient | Account holder, signs in with phone OTP | Manage own profile + family members, book for any of them |
| Family member | Profile under a patient account (child, parent). No login | Data owned by the account holder |
| Doctor | Verified medical practitioner | Own clinics, timings, appointments and queue |
| Clinic staff | Receptionist invited by a doctor | Only what the doctor grants: check-in, walk-ins, queue moves. **No medical notes** |
| Platform admin | Our team | Verify doctors, handle reports. **No routine access to medical data** (break-glass only, audited) |

## 4. MVP features (Phase 1)

### Patient app
1. **Sign in** with mobile number + OTP. Optional biometric app lock.
2. **Profile + family:** name, age/DOB, gender. Add family members ("Book for: Mom"). Consent screen on first use (DPDP).
3. **Home:** big "Book Appointment" button, upcoming appointment card with live queue status, "My doctors" (recently visited).
4. **Search:** by doctor name, clinic name, speciality, area/city. Top specialities as tiles.
5. **Doctor profile:** photo, qualifications, speciality, experience, verified badge, clinics with address, timings, map link and call button.
6. **Pick a slot:** "Book for" selector → date strip → clinic selector → Morning / Noon / Evening tabs → slot chips (available / booked / not available) → purpose (New, Follow-up, Show reports, Vaccination, Other) → **Book**.
7. **Confirmation:** token number, time, clinic, "Add to calendar", directions.
8. **My appointments:** upcoming and past; cancel or reschedule within the doctor's policy.
9. **Live queue:** "You are #3, approx. 25 min" with push alerts at #3 and #1.
10. **Notifications:** booking confirmed, reminder (evening before + 1 hr before), doctor delayed or cancelled.
11. **Privacy centre:** download my data, delete my account, manage consents, see who accessed my data (audit view).

### Doctor dashboard (web, mobile doctor mode later)
1. **Onboarding:** name, qualifications, speciality, registration number + council, ID proof upload. Shows "Pending verification" until an admin approves.
2. **Clinics:** add clinic (name, address, map pin, phone). Per clinic: weekly timings (multiple sessions per day), slot length (e.g. 10 min), max patients per slot, booking window (e.g. 7 days ahead), cancellation cut-off.
3. **Leave / holidays:** block dates or sessions. Affected patients are notified automatically.
4. **Today view:** list by session, statuses `Booked → Checked-in → In consultation → Done / No-show`. One-tap actions.
5. **Live queue control:** next patient, skip, push back, pause ("Doctor running 20 min late" broadcast).
6. **Walk-ins:** add a walk-in to the queue without an app booking (name + phone only).
7. **Patient history (own patients only):** past visits with this doctor and the visit purpose.

### Platform admin
1. Doctor verification queue (check registration number against the NMC / State Medical Council register).
2. Suspend accounts, handle abuse reports.
3. Breach-response runbook trigger (see SECURITY.md).

## 5. Non-functional requirements

| Area | Requirement |
|---|---|
| **Security & privacy** | **See SECURITY.md. Top priority. Blocks release.** |
| Performance | Slot list < 1 s on 4G. API p95 < 300 ms |
| Reliability | Zero double-bookings (enforced by DB constraint). 99.5% uptime target for MVP |
| Offline | Show cached upcoming appointment and token offline. Booking needs network |
| Accessibility | Font scaling up to 200%, screen-reader labels, 4.5:1 contrast, 48dp touch targets |
| Languages | English MVP. Hindi + Marathi in Phase 2 (strings in i18n files from day 1) |
| Devices | Android 8+ and iOS 15+ (check Expo SDK minimums when scaffolding) |
| Data residency | All health data stored in India (Azure Central India) |

## 6. Learnings from the reference app (screenshots reviewed 2026-11-16)

| What we saw | What we'll do |
|---|---|
| Slot screen showed the raw text **"null"** when a doctor had no slots | Proper empty state: "No slots on 16 Nov. Next available: Wed 18 Nov →" |
| "(not measured)" label next to clinic name with no meaning | Never show internal/debug labels. Every label must mean something to a patient |
| Experience shown as "34+ Yrs" with broken glyph rendering | Test custom fonts on real Android + iOS devices. Fall back to system fonts |
| Video consult button looks disabled (grey) with no explanation | Hide unavailable features or explain them ("Video consult coming soon") |
| Purpose picker opens as a dark dropdown over the slot grid | Use a bottom sheet with large options. The Book button stays disabled until slot + purpose are chosen, with a hint |
| Two family members with the same name ("Shahrukh Pathan" 29 and 27) | Show relation + age chip ("Self · 29", "Brother · 27") and avatar initials to tell them apart |
| Location shows "Mumbai" while doctors listed are in Pune | Auto-detect city with permission, otherwise ask once. Show distance on cards |
| No visible queue info on the profile | Show "Next available slot" and "Today's wait ~X min" on profile and search cards |
| Doctor with no photo/qualifications looks untrustworthy | Profile completeness meter for doctors. Verified badge only after admin check |

## 7. Out of scope for MVP

Payments, video consults, prescriptions, lab tests, pharmacy/e-commerce (the reference app's "MicroStore"), insurance and chat. Planned for Phase 2 and later (see PROGRESS.md).

## 8. Success metrics

- Booking completion rate > 70% of slot-screen visits
- No-show rate down 30% for onboarded clinics (reminders + live queue)
- Doctor weekly active rate > 80%
- **Zero security incidents involving patient data**
