# Design system & UX

Goal: **calm, trustworthy, fast.** A worried patient or a 60-year-old parent should book in under 60 seconds without help. Doctors should run their queue with one thumb.

## 1. Principles

1. **One primary action per screen.** It's always the big button at the bottom, in thumb reach.
2. **Never a dead end.** Every empty state suggests what to do next ("No slots today. Next available: Wed 18 Nov →").
3. **Show trust signals:** verified badge, qualifications, "Your data is encrypted" microcopy at booking and on the privacy screen.
4. **Plain words.** "Book for", "Your turn in ~20 min". No jargon, no internal labels, never `null`.
5. **Forgiving:** undo for cancel (5 s snackbar), confirm only for destructive actions.
6. **Accessible by default** (see §6).

## 2. Brand tokens (starting point, tweak with a designer)

The reference app uses saturated purple + neon lime. That's high-energy but tiring and low-contrast. We keep a confident primary with a calmer palette.

| Token | Light | Dark | Use |
|---|---|---|---|
| `primary` | `#5B3DF5` (indigo-violet) | `#8B7BFF` | main buttons, selected slot |
| `primary-contrast` | `#FFFFFF` | `#0F0B24` | text on primary |
| `accent` | `#14B8A6` (teal) | `#2DD4BF` | available, success, live-queue pulse |
| `warning` | `#F59E0B` | `#FBBF24` | doctor delayed |
| `danger` | `#E11D48` | `#FB7185` | booked/unavailable, cancel |
| `surface` | `#FFFFFF` | `#121218` | cards |
| `background` | `#F6F6FA` | `#0A0A0F` | screens |
| `text` | `#1A1A2E` | `#ECECF4` | body |
| `text-muted` | `#5C5C70` | `#A0A0B4` | secondary text (still ≥ 4.5:1) |

- **Type:** system font (SF Pro / Roboto) or Inter. Sizes 12 / 14 / 16 (body) / 20 / 24 / 32. Never below 12.
- **Spacing:** 4-pt grid (4, 8, 12, 16, 24, 32). Screen gutter 16.
- **Radius:** 12 cards, 999 chips/pills, 16 bottom sheets.
- **Icons:** one set only (e.g. Lucide / Phosphor).
- **Dark mode** from day 1.

## 3. Core components

| Component | Notes |
|---|---|
| `PrimaryButton` | full width, 52 px tall, loading + disabled states, haptic on press |
| `BookForSelector` | avatar initials + name + relation/age chip ("Self · 29"). Opens a bottom sheet with "+ Add family member" |
| `DateStrip` | horizontal scroll of 7–14 days, each showing day + date + a dot if slots are available. Better than the reference app's single-date arrows |
| `SessionTabs` | Morning / Afternoon / Evening, each with a count ("Morning · 8 open") |
| `SlotChip` | states: available (outline), selected (filled primary), booked (strikethrough, muted), unavailable (hidden or muted). **Colour + icon/text**, never colour alone |
| `PurposeSheet` | bottom sheet with large rows + icons: New, Follow-up, Show reports, Vaccination, Other |
| `DoctorCard` | photo/initials, name, verified badge, speciality, exp, distance, **next available slot** |
| `QueueCard` | big token number, "You are #3 · ~20 min", progress ring, doctor status (In clinic / Running late 15 min). Shows a subtle "Updating…" state while resyncing after a reconnect; never shows a stale position as live. **Pull to refresh** at any time (calls `GET /appointments/{id}/queue`), with an "Updated just now / 2 min ago" label |
| `BatteryTipSheet` | one-time bottom sheet after first booking, per OEM (Xiaomi, Oppo, Vivo, Realme, Samsung): "Allow DrBookingApp to run in background so you don't miss your turn" + button to the right settings screen |
| `SessionCard` (Token mode, ADR-0006) | session name + time range, "12 of 30 left", "your turn around 11:40" (always worded as an estimate), "Full today" state with next available date |
| `DeskQueueRow` | token, name, status chip, position; actions Check in / Not here / No-show; drag handle (reorder). "+1 extra" badge for doctor overrides |
| `PinPad` | 4–6 digit entry, used for the Desk PIN and the Doctor PIN. Clearly labelled which PIN is asked for; attempts left shown after a failure |
| `DeskModeBanner` | persistent strip "Desk mode · Desk PC" with a lock icon; "Exit" asks for the Doctor PIN |
| `ClinicQrPanel` (ADR-0010) | large QR + "Scan to track your token" + one line "Log in with the number you gave at the desk". Used on the desk screen and the printable standee |
| `EmptyState` | illustration + one sentence + one action |
| `PrivacyNote` | small lock icon + one line, at booking and profile |

## 4. Screen list (MVP)

### Patient app
1. Splash → Onboarding (3 slides: Book · Skip the wait · Your data is private)
2. Phone entry → OTP (auto-read on Android via SMS Retriever) → Consent screen
3. Profile setup (name, DOB, gender) → optional "Add family"
4. **Home:** greeting, upcoming `QueueCard`, big *Book appointment*, My doctors, specialities grid. **Appointment today → open Live queue straight after login** (ADR-0010)
5. **Search:** search bar (doctor, clinic, speciality), city chip, speciality tiles, recent searches, results list
6. **Doctor profile:** header card, verified badge, about, clinics (address, timings, Call, Directions), sticky *Book* button
7. **Pick a slot:** BookFor → DateStrip → clinic selector → SessionTabs → SlotChips → PurposeSheet → *Book*
8. **Confirmation:** token, time, clinic, Add to calendar, Directions, Share (no health details)
9. **Appointments:** Upcoming / Past tabs, cancel/reschedule
10. **Live queue:** full-screen QueueCard, auto-updating
11. **Profile & Family**, **Privacy centre** (consents, download data, delete account, access log), **Settings** (language, biometric lock, notifications)

### Doctor dashboard (web, responsive)
1. Login + MFA → Onboarding wizard (profile → clinic → **booking style choice: Token queue (Recommended) / Time slots**, required → timings + limits → verification upload)
2. **Today:** session switcher, counters (Booked / Waiting / Done / No-show / "30/30 + 1 extra"), `DeskQueueRow` list with big *Call next* button, *Add patient* (phone → check → direct add or OTP registration)
3. Calendar (week view) with leaves; "today only" limit change
4. Clinics & timings editor (visual weekly grid; slot fields only in Time-slot mode)
5. **Devices & Desk mode** (ADR-0007): signed-in devices list (rename, sign out), set Desk PIN, set Doctor PIN, switch Desk mode on
6. **Clinic QR** (ADR-0010): preview, print A5 standee, regenerate code
7. Profile completeness meter
*(Staff & permissions screen removed for MVP: single doctor-side login.)*

### Public web (separate static domain, ADR-0010)
- `/c/<code>` landing page: clinic name + doctor, Play Store / App Store button (OS-detected), "After installing, log in with the phone number you gave at the desk. Your token will be there." No cookies, no trackers.

## 5. UX research notes (common patterns in top booking apps)

- Show the **next available slot** on search cards. It's the #1 thing users compare.
- **Smart filters:** speciality, area/distance, availability today, gender of doctor, fee.
- **Live queue / wait time** is the strongest differentiator for Indian clinics (patients hate waiting rooms).
- Keep clinic-staff UI very simple. Products in this space have made live queue management *optional* for clinics after feedback that it added work.
- Reminders cut no-shows. Always include an easy cancel link so the slot goes back to someone else.

## 6. Accessibility checklist

- [ ] Every touchable has an `accessibilityLabel` and a ≥ 48×48 dp target
- [ ] Text scales to 200% without clipping (test with the largest system font)
- [ ] Contrast ≥ 4.5:1 for text, ≥ 3:1 for UI elements
- [ ] Status never shown by colour alone
- [ ] Screen reader order makes sense (test TalkBack + VoiceOver)
- [ ] Respect "reduce motion"

## 7. Privacy in the UI

- No health details in notifications: "Appointment reminder: tomorrow 11:00 AM" (no doctor speciality, no purpose).
- **Queue notifications (ADR-0004) use only this copy:**
  - "3 patients ahead of you. Please reach the clinic." (sent once, app notification; SMS on hold for MVP)
  - Desk add: "Dr <name>'s clinic added you to today's queue. Token 14. Not you, or for a family member? Tap to change."
  - "You're next."
  - "Your doctor is running about 30 min late."
  - "Your appointment on 16 Nov was cancelled. Tap to rebook."
- **Clinic name is off by default** in push/SMS (it can reveal sensitive care such as fertility or psychiatry). Users can opt in under Settings → Notifications.
- Push payloads carry only the appointment id. Details load inside the app after unlock.
- Blur the app in the task switcher. Block screenshots on records screens (Android).
- Shared links show only doctor + time, never purpose.
- The lock icon + "Encrypted" microcopy appears where users enter personal info.
