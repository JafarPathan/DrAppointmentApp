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
| `QueueCard` | big token number, "You are #3 · ~20 min", progress ring, doctor status (In clinic / Running late 15 min) |
| `EmptyState` | illustration + one sentence + one action |
| `PrivacyNote` | small lock icon + one line, at booking and profile |

## 4. Screen list (MVP)

### Patient app
1. Splash → Onboarding (3 slides: Book · Skip the wait · Your data is private)
2. Phone entry → OTP (auto-read on Android via SMS Retriever) → Consent screen
3. Profile setup (name, DOB, gender) → optional "Add family"
4. **Home:** greeting, upcoming `QueueCard`, big *Book appointment*, My doctors, specialities grid
5. **Search:** search bar (doctor, clinic, speciality), city chip, speciality tiles, recent searches, results list
6. **Doctor profile:** header card, verified badge, about, clinics (address, timings, Call, Directions), sticky *Book* button
7. **Pick a slot:** BookFor → DateStrip → clinic selector → SessionTabs → SlotChips → PurposeSheet → *Book*
8. **Confirmation:** token, time, clinic, Add to calendar, Directions, Share (no health details)
9. **Appointments:** Upcoming / Past tabs, cancel/reschedule
10. **Live queue:** full-screen QueueCard, auto-updating
11. **Profile & Family**, **Privacy centre** (consents, download data, delete account, access log), **Settings** (language, biometric lock, notifications)

### Doctor dashboard (web, responsive)
1. Login + MFA → Onboarding wizard (profile → clinic → timings → verification upload)
2. **Today:** session switcher, counters (Booked / Waiting / Done / No-show), queue list with big *Call next* button
3. Calendar (week view) with leaves
4. Clinics & timings editor (visual weekly grid)
5. Staff & permissions
6. Profile completeness meter

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
- Blur the app in the task switcher. Block screenshots on records screens (Android).
- Shared links show only doctor + time, never purpose.
- The lock icon + "Encrypted" microcopy appears where users enter personal info.
