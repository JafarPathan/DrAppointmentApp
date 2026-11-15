---
description: Plan a feature end-to-end (story, API, DB, screens, threat model, tests, tasks)
argument-hint: <feature name or description>
---

Plan this feature for DrBookingApp: **$ARGUMENTS**

Read AGENTS.md, docs/PRD.md, docs/ARCHITECTURE.md, docs/SECURITY.md, docs/DATABASE.md and docs/API.md first. Look at existing code if there is any.

Write the plan to `docs/plans/<kebab-feature-name>.md` with these sections:
1. **User story + acceptance criteria** (Given/When/Then), for each role affected.
2. **Out of scope.**
3. **API changes:** endpoints, request/response, who can call them, status codes.
4. **DB changes:** tables/columns, which are encrypted 🔐, indexes, RLS, migration plan.
5. **Screens:** list them, with states (loading/empty/error). Suggest running `/design` for each.
6. **🔒 Threat model (STRIDE):** spoofing, tampering, repudiation, info disclosure (IDOR!), DoS, elevation, with a mitigation for each. What PHI is touched and how it's protected.
7. **Tests:** unit, integration, authorization matrix, E2E.
8. **Task breakdown:** small ordered steps, each sized for one commit (Conventional Commit message suggested).
9. **Docs to update:** CHANGELOG, PROGRESS, API/DB, pitch/video if user-visible.

Then remind the dev to create the branch `feature/<name>` if not already on one. Don't write code yet.
