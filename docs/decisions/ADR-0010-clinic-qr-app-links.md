# ADR-0010: Clinic QR code and app links ("Scan to track your token")

- **Status:** Accepted
- **Date:** 2026-11-16
- **Debate:** [DEBATE.md Thread 011](../../DEBATE.md) (🟢 consensus between Claude Code and Antigravity)
- **Note:** ADR-0009 is reserved for reminder channels (Thread 010, on hold).

## Context

Walk-ins without the app get no remote updates in MVP (SMS and WhatsApp are on hold). They can install the app and log in with the phone number the desk used, and their token is already there (ADR-0008). We want a scan-and-go path, **without any personal data in a QR code or URL** (SECURITY: no personal data in URLs).

Firebase Dynamic Links shut down on 25 Aug 2025. Google recommends native Android App Links + iOS Universal Links. Deferred deep linking needs paid third-party SDKs, which add tracking.

## Decision

### 1. One static QR per clinic, no personal data
- Each clinic gets a **random, rotatable public code** (`clinics.public_code`, e.g. `k7m2qp`; not the DB id).
- The QR encodes **`https://<web-domain>/c/<public_code>`**. `<web-domain>` is a placeholder until the maintainer picks the product name and buys a domain. **Don't hard-code a domain we don't own.**
- It's used as a **printed standee/poster at reception** (the doctor dashboard has "Print clinic QR"), **on the desk screen** next to a newly added walk-in, and optionally on printed token slips (open question for the maintainer).
- A photographed QR reveals only the public clinic. The doctor can **rotate** the code (old links show "This link has changed. Ask the clinic.").

### 2. App installed → open the app
- **iOS Universal Links** (`ios.associatedDomains: ["applinks:<web-domain>"]`) and **Android App Links** (intent filter with `autoVerify`, path `/c/*`), configured through Expo. They need a development or store build (not Expo Go).
- `<web-domain>` serves `/.well-known/apple-app-site-association` and `/.well-known/assetlinks.json`. These contain only app IDs and signing-cert fingerprints, nothing secret.
- Expo Router route `/c/[code]` opens that clinic's page (doctor info + Book next visit).

### 3. App not installed → static landing page
- A **static, cookie-less, tracker-free** page at `/c/<public_code>` (no analytics SDKs, no ad pixels). It shows public clinic info, detects Android or iOS, and shows the right store button plus:
  > "After installing, log in with the phone number you gave at the desk. Your token will be there."

### 4. After login → today's queue first
On launch after login, if the account (any family member) has an **active appointment today**, the app opens **Live queue** directly. This needs no link context at all, so it works on iOS without deferred deep linking.

### 5. Hosting / domain separation
The landing page and association files are served from a **separate static web domain** (CDN), **not** the API domain. Auth cookies and API origins never share an origin with the public page.

### 6. Later (optional, Android only)
Pass the clinic code through the Play Store link (`referrer=`) and read it with the **Play Install Referrer API** on first launch to open the clinic page after a fresh install. It's free and first-party. iOS keeps behaviour §4.

## Alternatives considered

| Option | Why not |
|---|---|
| Per-patient QR with phone or appointment ID | Personal data in the URL; anyone photographing the screen gets it |
| Per-patient opaque one-time token | Needs deferred linking (paid SDK on iOS); more moving parts |
| Third-party deep-link SDK (Branch, AppsFlyer, …) | Cost + tracking + an extra data processor |
| Landing page on the API domain | Mixes public page and auth origin |
| Verbal instruction only | Slow and often skipped at a busy desk |

## Consequences

- ✅ Free, no personal data, no tracking, works for both platforms.
- ✅ The same QR drives repeat bookings ("Book your next visit").
- ⚠️ iOS first launch after install lands on Home (then auto-opens today's queue after login), not on the clinic page.
- ⚠️ Needs a real domain before launch.

## Required tests

1. A QR/URL contains only `public_code`. A lint test fails on any `phone`, `token` or `appointment` query parameter in generated links.
2. Installed app opens `/c/<code>` on iOS and Android (device test on store/dev builds).
3. Landing page has no cookies or third-party requests (automated check).
4. After login with an active appointment today → Live queue opens. Otherwise → Home.
5. Rotated code → old link shows the "link has changed" page.

## Docs to update

- `docs/ARCHITECTURE.md` §2: web landing domain + app links.
- `docs/DATABASE.md`: `clinics.public_code`.
- `docs/PRD.md` §4: "Print clinic QR" (doctor); auto-open today's queue (patient).
- `docs/WORKFLOW.md` A6: desk shows the clinic QR.
- `docs/DESIGN.md`: QR standee, desk QR panel, landing page.
- `docs/SECURITY.md` §5: public code rules, separate static domain.

## Sources

- Pragmatic Engineer, Firebase Dynamic Links shutdown: https://blog.pragmaticengineer.com/google-shutting-down-firebase-dynamic-links/
- Android App Links: https://developer.android.com/training/app-links
- Apple Universal Links: https://developer.apple.com/ios/universal-links/
- Expo, iOS Universal Links: https://docs.expo.dev/linking/ios-universal-links/
- Expo, Linking overview: https://docs.expo.dev/linking/overview/
- Android, Play Install Referrer API: https://developer.android.com/google/play/installreferrer/igetinstallreferrerservice
