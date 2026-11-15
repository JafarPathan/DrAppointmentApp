# Product Requirements (PRD)

## 1. Problem: how a small Indian clinic runs today

### 1.1 The day today (as-is)
1. A patient walks in and **tells the receptionist their name**. The receptionist writes it in a register or notebook and gives them a number.
2. The patient **sits in the waiting room**, often for 1–2 hours, with no idea how many people are ahead of them. They can't step out for tea, go home or go back to work, because they might miss their name being called.
3. The receptionist **calls names out loud in register order**. If a patient isn't there (stepped out, stuck in traffic), the receptionist either skips them or waits, and the order becomes confusing and argued over.
4. People phone the clinic to ask "how long?" or "is the doctor in today?", interrupting the receptionist and doctor.
5. The doctor has **no control over the day's load**. Too many patients arrive, and the doctor rushes or stays late. Nobody outside the clinic knows the day is already full.
6. Many patients (elderly people, first-time visitors) have **never heard of any app**. Any solution must also work for people who simply walk in.
7. Small clinics often have **one receptionist or none**. Sometimes the doctor manages the queue alone.
8. Patient names and phone numbers sit in notebooks and WhatsApp chats: a privacy risk under the DPDP Act.

### 1.2 What we'll change (to-be)
| Today | With DrBookingApp |
|---|---|
| Wait in the clinic for hours | Book or register, then **wait anywhere**. Get **one app notification when 3 patients are ahead of you** (SMS alerts on hold for MVP) |
| Don't know your position | **Live queue** in the app: "You are #5 · about 25 min". Updates every time the queue moves |
| Name shouted, miss it if you stepped out | Queue moves on the receptionist's or doctor's tap. Late patients are **moved back, not lost** |
| Phone calls asking "how long?" | Patients see it themselves |
| No limit on the day's patients | Doctor sets **how many patients per day / per session**. The app shows **"Full today"** and stops bookings when full |
| Walk-ins with no app are left out | Desk **adds them on the spot** by phone number. **Already registered** → added straight to the queue, updates in their app. **New** → registered by OTP. They can **download the app and log in with the same number anytime** to track their token live |
| Doctor alone or with a receptionist | **One doctor-side login** for MVP. The doctor runs the desk alone, or the receptionist uses the same login on the desk device (in **Desk mode**, which hides patient history) |
| Names in notebooks | Encrypted, access-logged, DPDP-compliant |

### 1.3 Evidence
- Indian OPD studies report long waits against very short consultations. A New Delhi AYUSH hospital time-motion study found ~2 h 17 min total time after joining the queue against ~3 min 22 s in the consultation room. A North India tertiary study reports ~111 min mean wait against ~5 min consultation. *(These are hospital OPD studies; we found no published study of small private clinics, so treat them as directional.)*
- Token + call display systems cut OPD journey time in a Tamil Nadu government re-engineering project (57 → 36 min) by letting patients wait seated instead of crowding the counter.
- Token systems are still the dominant (FCFS) model in Indian OPDs, so our design must fit **token order**, not just Western-style fixed time slots.
- Sources: [NHSRC Tamil Nadu OPD report](https://qps.nhsrcindia.org/sites/default/files/2021-05/Addressing%20Overcrowding%20in%20OPD%20by%20Process%20Mapping%20and%20Process%20Re-Engineering%20-%20Tamil%20Nadu.pdf) · [IJCMPH waiting-time study](https://www.ijcmph.com/index.php/ijcmph/article/download/16572/9846/86585) · [Token systems field study, Management Science 2023](https://pubsonline.informs.org/doi/fpi/10.1287/mnsc.2022.4389) · [arXiv 2026 OPD preprint](https://arxiv.org/pdf/2604.00215)

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
| Clinic staff (receptionist) | **MVP: no separate account.** Uses the doctor's login on a desk device the doctor signed in (maintainer decision, DEBATE Thread 008). Separate receptionist logins to be revisited before Phase 2 adds medical records | In **Desk mode**: add/register patients, check in, call next, move back, no-show, reorder. Patient history and settings hidden and blocked server-side; receptionist uses the **Desk PIN**, and only the **Doctor PIN** leaves Desk mode (ADR-0007) |
| Platform admin | Our team | Verify doctors, handle reports. **No routine access to medical data** (break-glass only, audited) |

## 4. MVP features (Phase 1)

### Patient app
1. **Sign in** with mobile number + OTP. Optional biometric app lock.
2. **Profile + family:** name, age/DOB, gender. Add family members ("Book for: Mom"). Consent screen on first use (DPDP).
3. **Home:** big "Book Appointment" button, upcoming appointment card with live queue status, "My doctors" (recently visited). **If there's an appointment today, the app opens Live queue straight after login** (e.g. a walk-in who just installed via the clinic QR, ADR-0010).
4. **Search:** by doctor name, clinic name, speciality, area/city. Top specialities as tiles.
5. **Doctor profile:** photo, qualifications, speciality, experience, verified badge, clinics with address, timings, map link and call button.
6. **Book:** "Book for" selector → date strip → clinic selector → then, depending on the doctor's booking style:
   - **Token queue:** session cards ("Morning · 12 of 30 left · your turn around 11:40") → purpose → **Book** → token number + estimated time.
   - **Time slots:** Morning / Noon / Evening tabs → slot chips (available / booked / not available) → purpose → **Book**.
   - Purpose options: New, Follow-up, Show reports, Vaccination, Other. When full: "Full today. Next available: <date>".
7. **Confirmation:** token number, time, clinic, "Add to calendar", directions.
8. **My appointments:** upcoming and past; cancel or reschedule within the doctor's policy.
9. **Live queue:** "You are #3, approx. 25 min". It updates by itself while open, and the patient can **pull to refresh** at any time. It stays correct after network drops (live updates + resync, ADR-0004). One-time tip to allow background running on Xiaomi/Oppo/Vivo/Realme/Samsung phones.
10. **Notifications:** booking confirmed, reminder (evening before + 1 hr before). High-priority **app notifications** for exactly 4 events: **3 patients ahead of you** (sent **once** per visit), you're next, doctor running late, doctor cancelled. **Notification SMS is on hold for MVP** (maintainer decision, DEBATE Thread 003). Only OTP SMS for login/registration is sent. The SMS fallback is built behind a feature flag for later. Text is generic, with the clinic name off by default.
10a. **WhatsApp reminders: ⏸️ on hold** (maintainer decision, DEBATE Thread 010). Options parked for later: a free one-tap desk button (click-to-chat), or automatic messages through the WhatsApp Business API (paid, roughly ₹0.14/message incl. GST, to be verified).
10b. **Refresh anytime:** the patient can open the app or **pull down to refresh** on the Live queue screen to see their current position, estimated time and doctor status, even without a notification.
11. **Privacy centre:** download my data, delete my account, manage consents, see who accessed my data (audit view).

### Doctor dashboard (web, mobile doctor mode later)
1. **Onboarding:** name, qualifications, speciality, registration number + council, ID proof upload. Shows "Pending verification" until an admin approves.
2. **Clinics:** add clinic (name, address, map pin, phone). Per clinic the doctor **chooses the booking style** (maintainer decision, DEBATE Thread 007):
   - **Token queue** ("Recommended for most clinics"): sessions + max patients per session. Patients get a token and an estimated time.
   - **Time slots:** sessions + slot length (e.g. 10 min) + patients per slot. Patients pick an exact time.
   - The choice is required at setup (nothing chosen silently) and can be changed later for future dates. Existing bookings are never affected.
   - Also per clinic: booking window (e.g. 7 days ahead), cancellation cut-off.
3. **Leave / holidays:** block dates or sessions. Affected patients are notified automatically.
4. **Today view:** list by session, statuses `Booked → Checked-in → In consultation → Done / No-show`. One-tap actions.
5. **Live queue control:** next patient, skip, push back, pause ("Doctor running 20 min late" broadcast).
6. **Walk-ins and on-the-spot registration** (maintainer decision, DEBATE Thread 009): the desk enters the patient's phone number and the system checks whether it's registered.
   - **Already registered** → added **directly** to today's queue, no OTP. The patient gets an app notification ("Dr X's clinic added you to today's queue. Token 14. Not you, or for a family member? Tap to change"). The desk types the name the patient gives. It sees stored names or family members **only if this clinic has seen that patient before**.
   - **Not registered** → OTP SMS to the phone. The patient tells the desk the code (or confirms via the link), and is registered (name, age, gender) and added to the queue.
   - Without the app, the patient gets the token at the desk (no SMS alerts in MVP). They can **install the app at any time, even while waiting, and log in with the same phone + OTP**. Their token and live queue for today appear straight away, because the desk-created entry belongs to that phone's account. The desk should tell them: "Download the app and log in with this number to track your token."
6a. **Daily / session capacity:** the doctor sets how many patients they can see per day or per session. App bookings and walk-ins both count. When full, the app shows **"Full today"** with the next available day, and the desk sees a warning (the doctor can still add an emergency case). Design: DEBATE Thread 007.
6b. **Not-present handling:** if a called patient isn't there, the doctor or receptionist taps **Not here → move back** (by N places, or to the end) or **No-show**. Everyone's position and ETA updates immediately.
6c. **Single doctor-side login + Desk mode** ([ADR-0007](decisions/ADR-0007-single-doctor-login-desk-mode.md)): the doctor runs everything alone, or signs in the desk device for the receptionist and switches it to **Desk mode** (Today list and desk actions only; patient history, settings and the "extra token" override hidden and blocked by the server).
   - **Two PINs:** a **Desk PIN** the receptionist uses to unlock the desk device after idle, and a private **Doctor PIN** to leave Desk mode. They must be different.
   - A **Signed-in devices** list lets the doctor sign out any device instantly, e.g. when a receptionist leaves. Every action is logged with the device.
6d. **Print clinic QR** ([ADR-0010](decisions/ADR-0010-clinic-qr-app-links.md)): the doctor prints a "Scan to track your token / book your next visit" QR standee for reception. It's also shown on the desk screen after adding a walk-in. The doctor can regenerate it if misused.
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
| Reliability | Zero over-bookings for any slot capacity (atomic capacity claim on `slot_inventory`, ADR-0002, proven by a concurrency test). 99.5% uptime target for MVP |
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
