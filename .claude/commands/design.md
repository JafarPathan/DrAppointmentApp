---
description: Design a screen or flow against the design system (layout, states, components, a11y, privacy)
argument-hint: <screen or flow name>
---

Design this screen/flow for DrBookingApp: **$ARGUMENTS**

Read docs/DESIGN.md (tokens, components, principles, accessibility, privacy-in-UI) and docs/PRD.md (including §6, lessons from the reference app).

Write the spec to `docs/design/<kebab-name>.md`:
1. **Purpose + primary action** (exactly one).
2. **Wireframe:** ASCII layout for a 360dp-wide phone (and desktop if it's a dashboard screen).
3. **Components used** from the design system. Propose any new component with its props and states.
4. **All states:** loading (skeleton), empty (with next-step action), error (friendly + retry), success, offline, and partial data. **Never show null/undefined.**
5. **Copy:** every user-facing string in plain English (keys for i18n), including errors and empty states.
6. **Accessibility:** labels, focus order, touch targets, font scaling, colour-independent states.
7. **Privacy:** what personal/health data appears, whether the screen needs blur/FLAG_SECURE, what's safe in notifications/share text.
8. **Edge cases:** long names, two family members with the same name, no photo, slow network, time-zone boundaries.
9. **API calls** the screen needs (reference docs/API.md).

If it's useful, offer to build a quick HTML mockup.
