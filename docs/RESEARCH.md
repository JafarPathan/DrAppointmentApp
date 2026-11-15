# Research notes (2026-11-16)

Quick web research to shape the PRD. Re-check before relying on anything legal.

## 1. Reference app reviewed (user screenshots)

A Syntagi-style app: home with "Book appointment", a "Book for" family switcher, speciality tiles, doctor profile with clinics + timings, and a slot picker with Morning/Noon/Evening tabs and purpose selection. **What to keep and what to fix:** see PRD §6.

## 2. Market patterns

- Practo-style platforms bundle discovery, booking, a clinic calendar with multi-doctor scheduling and slot management, and a staff app for receptionists to manage the queue.
- Queue-first products are a growing niche: predictive queue management (CareClues Command), patient-side live queue position (NoQueue), reception queue tools (Docterz Pro).
- Lesson from LocDoc's changelog: simplify the staff UI and make live queue management **optional** per clinic.
- Common must-haves: smart filters (location, speciality, rating), easy profile management, role-based access for clinic staff.

**Our angle:** small/mid clinics in Tier-1/2 cities, live queue + privacy-first, cheap and simple for doctors.

## 3. Open-source repos worth reading (check licenses before reusing any code)

| Repo | Stack | Worth looking at |
|---|---|---|
| AnhCaooo/app-doctor | Expo + React Native + Firebase | Booking flow with in-person / audio / video choice |
| structlooper/modern-healthcare-mobile-application | React Native | Doctor + patient appointment and calendar screens |
| satyam1013/doctor-appointment | React Native + NestJS | Booking + video consult structure |
| randyjap/DrAppointment | React Native | SMS confirmation + reminders |
| akshaychavan010101/Hospital-Booking-App | React + Express | Web booking flow |
| OpenCare (FOSS Hack 25) | — | Self-hosted scheduler ideas |

None of these are production-grade on security. Use them for UI/flow ideas only.

## 4. Legal / compliance (India)

- **DPDP Act 2023 + DPDP Rules 2025** (notified Nov 2025, phased commencement; sources disagree on exact dates, so check the Gazette).
  - Clear consent notices naming the exact data fields and their purpose (health + location example given by commentators).
  - Breach: report to the Data Protection Board **within 72 hours**, inform affected users without delay.
  - Verifiable parental consent for under-18s (narrow exemptions exist for clinical establishments and health professionals; a general app likely doesn't qualify).
  - Reasonable security safeguards, logs retained ~1 year. Penalties up to ₹250 crore.
- **Telemedicine Practice Guidelines (2020)**: needed before launching video consults (Phase 2).
- **NMC / State Medical Council registration** to verify doctors.
- **ABDM / ABHA**: optional health-ID integration later.

## Sources

- https://awesome.ecosyste.ms/projects/github.com%2Fanhcaooo%2Fapp-doctor
- https://awesome.ecosyste.ms/projects/github.com%2Fstructlooper%2Fmodern-healthcare-mobile-application
- https://awesome.ecosyste.ms/projects/github.com%2Fsatyam1013%2Fdoctor-appointment
- https://github.com/randyjap/DrAppointment
- https://www.github.com/akshaychavan010101/Hospital-Booking-App
- https://fossunited.org/hack/fosshack25/p/2k0pkn23f4
- https://ourtake.bakerbotts.com/post/102lund/india-notifies-final-rules-for-digital-data-protection-act
- https://www.mondaq.com/india/data-protection/1706890/a-closer-look-at-the-dpdp-rules-2025
- https://theleaflet.in/explainer/indias-new-digital-personal-data-protection-rules-2025-a-detailed-reading
- https://ca.hoganlovells.com/en/publications/indias-digital-personal-data-protection-act-2023-brought-into-force-
- https://www.scribd.com/document/1046552196/Practo-Help
- https://www.slideshare.net/slideshow/careclues-command-pms-tracker-qms-benefits-features-pricing/238494622
- https://apps.apple.com/in/app/noqueue-skip-the-wait/id6759590485
- https://apps.apple.com/cd/app/docterz-pro/id6746152811
- https://apps.apple.com/in/app/locdoc/id6757434799
