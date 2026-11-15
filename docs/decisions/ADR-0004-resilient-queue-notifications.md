# ADR-0004: Resilient live-queue updates and notifications

- **Status:** Accepted
- **Date:** 2026-11-16
- **Debate:** [DEBATE.md Thread 003](../../DEBATE.md) (🟢 consensus between Claude Code and Antigravity)
- **Extends:** ARCHITECTURE.md §4 "Live queue", API.md "Realtime", DESIGN.md §7
- **Amended 2026-11-16 by maintainer decision** (DEBATE Thread 003, `[STAMP] Human`): the first alert fires when **3 patients are ahead** (not "#3"), **once per visit**, by **SMS + app notification**. Where this ADR says "patient is #3", read "3 patients ahead".
- **Amended again 2026-11-16 by maintainer decision** (Thread 003): **all notification SMS are on hold for MVP.** Every alert in this ADR goes by app notification only. The SMS fallback is implemented behind a feature flag (default off). OTP SMS for login and registration isn't affected.

## Context

Patients rely on the live queue ("You are #3 · ~20 min") to decide when to leave home. Indian mobile conditions break a sockets-only design:
- SignalR connections drop on network handovers and when the OS backgrounds or kills the app.
- After `withAutomaticReconnect()` (retries at 0, 2, 10, 30 s, then stops) the server sees a **new connectionId**, and group memberships are lost.
- Azure SignalR stateful reconnect only covers a short window (~30 s). Beyond that, Microsoft says to recover from an application-level store.
- FCM high priority should be used only for messages that show a notification. OEM battery managers (notably Xiaomi/MIUI) can still block notifications for restricted apps.
- Commercial SMS in India costs money per message and must go through registered templates (TRAI DLT; source to be added, see Follow-ups).

## Decision

### Three tiers

| Tier | When | Mechanism |
|---|---|---|
| **1. Realtime** | Queue screen focused, socket connected | SignalR `QueueUpdated` |
| **2. Push** | App backgrounded or killed | High-priority FCM/APNs for **exactly 4 events**: patient reaches **#3**, patient is **#1 / next**, **doctor running late**, **appointment cancelled by doctor** |
| **3. Reconciliation** | App comes to foreground, socket reconnects, a version gap is detected, or the socket is down while the screen is focused | `GET /appointments/{id}/queue` with ETag; 30 s polling with backoff while focused and disconnected |

**SMS fallback** only for **doctor cancellation** and **delays > 30 minutes**. Never for routine position changes.

### Rules

1. **The server is the single source of truth.** Position and ETA are always computed server-side from `appointments`. The client never derives its position by counting events.
2. **Versioning:** every clinic session has a monotonically increasing `session_version`, bumped on every queue change (check-in, call next, skip, no-show, cancel, delay). Every `QueueUpdated` message and every REST queue response includes it.
3. **Gap detection:** if a received `session_version` isn't exactly `last + 1`, or after any reconnect, the client discards local assumptions and refetches over REST.
4. **Reconnect:** in `onreconnected` the client **re-invokes `JoinAppointment(appointmentId)`**, and the server re-checks ownership before re-adding the client to the group, then refetches.
5. **ETag:** `GET /appointments/{id}/queue` returns `ETag: "<session_version>"` and honours `If-None-Match` → `304 Not Modified`, saving data during polling.
6. **Push volume:** high priority only for the 4 events above, deduplicated per appointment per event. Everything else is normal priority or in-app only.
7. **Battery onboarding:** on first booking, detect the device manufacturer and show a one-time tip linking to the right battery setting (Xiaomi, Oppo, Vivo, Realme, Samsung), e.g. "Allow DrBookingApp to run in background so you don't miss your turn."

### 🔒 Privacy rules (Rule 0)

- Push and SMS text is **generic**: "Queue update: you're #3. Leave now." / "You're next." / "Your doctor is running about 30 min late." / "Your appointment on 16 Nov was cancelled. Tap to rebook."
- **No speciality, purpose, symptoms or clinic name** by default, because a clinic's name can reveal sensitive care (e.g. fertility, psychiatry). Users can opt in to showing the clinic name.
- Payloads carry only the appointment id. The app fetches details after unlock and authentication.
- `QueueUpdated` sent to a patient contains only `{appointmentId, position, etaMinutes, status, sessionVersion}`, never other patients' data.

### API changes

```
GET /appointments/{id}/queue            (owner only)
  → 200 { position, etaMinutes, doctorStatus, sessionVersion }  ETag: "<sessionVersion>"
  → 304 when If-None-Match matches

SignalR /hubs/queue
  JoinAppointment(appointmentId)        client → server (ownership checked; call again after reconnect)
  QueueUpdated { appointmentId, position, etaMinutes, status, sessionVersion }  server → patient
```

## Alternatives considered

| Option | Why not |
|---|---|
| SignalR only | Dies in background; missed updates → late patients |
| Android foreground service keeping the socket alive | Battery drain; still killed by OEMs; Play policy scrutiny; no iOS equivalent |
| SMS for every queue step | Expensive per message, noisy, needs DLT templates for each |
| Separate `/queue-position` endpoint | Duplicates the existing `/appointments/{id}/queue` |

## Consequences

- ✅ Correct position after any disconnect. Alerts survive backgrounding on most devices.
- ✅ SMS costs are bounded to rare events.
- ⚠️ Delivery is still best-effort on aggressive OEM ROMs. The battery tip reduces but doesn't remove this risk.
- ⚠️ Every queue mutation must bump `session_version` in the same transaction (enforced in the queue command handlers).

## Required tests

1. Disconnect → reconnect → client re-joins the group and receives subsequent updates (integration test with the SignalR test client).
2. Version gap (skip one event) → client refetches and shows the correct position.
3. `If-None-Match` with the current version → 304. After a change → 200 with the new ETag.
4. `JoinAppointment` for someone else's appointment → rejected (authorization matrix).
5. Push payload snapshot tests: no clinic name, speciality or purpose unless opted in.
6. Each push event fires once per appointment (dedup).

## Follow-ups

- Add an authoritative TRAI / TCCCPR source on DLT template registration to this ADR (the claim is currently unsourced).
- PROGRESS 1.9: add "Register SMS templates on DLT" and "OEM battery-settings tip".
- Decide the SMS budget per clinic per month (human).

## Docs to update after this ADR

- `docs/ARCHITECTURE.md` §4 "Live queue": the tiers and rules above.
- `docs/API.md`: `/appointments/{id}/queue` (ETag, `sessionVersion`), Realtime section.
- `docs/DESIGN.md` §7: generic notification copy, clinic-name opt-in.
- `docs/PRD.md` §4: battery tip, notification events.
- `docs/DATABASE.md`: `session_version` column (on a per-clinic-session queue state table, defined when queue state is modelled).

## Sources

- Microsoft Learn, SignalR JavaScript client: https://learn.microsoft.com/aspnet/core/signalr/javascript-client
- Microsoft Learn, SignalR groups: https://learn.microsoft.com/aspnet/core/signalr/groups
- Azure SignalR, client disconnections: https://docs.azure.cn/en-us/azure-signalr/signalr-concept-client-disconnections
- RFC 9110 §13, Conditional Requests: https://www.rfc-editor.org/rfc/rfc9110.html#section-13
- Firebase, Android message priority: https://firebase.google.com/docs/cloud-messaging/doc-revamp/optimize-delivery/android-message-priority?hl=de
- Android Developers Blog, notifying users with FCM: https://android-developers.googleblog.com/2018/09/notifying-your-users-with-fcm.html
- dontkillmyapp, Xiaomi: https://dontkillmyapp.com/xiaomi
- Batch, notification issues on OEM devices: https://help.batch.com/en/articles/2246074-how-can-i-fix-a-notification-issue-on-my-honor-huawei-sony-oppo
