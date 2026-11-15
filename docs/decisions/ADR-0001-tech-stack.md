# ADR-0001: Tech stack

- **Status:** Accepted
- **Date:** 2026-11-16

## Context
We need Android + iOS apps, a doctor dashboard and a secure backend. The team knows React, .NET, SQL and some Node. It's a side project, so one codebase per layer and low hosting cost matter. Health data means security and India data residency come first.

## Decision
- **Mobile:** React Native + Expo (TypeScript). One codebase, reuses React skills, EAS handles store builds.
- **Web dashboard:** React + Vite + TypeScript.
- **Backend:** ASP.NET Core (.NET 10 LTS). Mature auth, Data Protection, policy authorization, rate limiting, SignalR. The team knows C#.
- **DB:** PostgreSQL (EF Core). Row-level security, partial unique indexes for double-booking prevention, cheap managed hosting. SQL Server remains a drop-in alternative through EF Core.
- **Cloud:** Azure Central India (Key Vault, App Service, Postgres Flexible Server, Blob, SignalR Service).

## Alternatives considered
- **Flutter:** great UI performance, but Dart is a new language for the team.
- **Native Kotlin + Swift:** two codebases, too costly for a side project.
- **Node/NestJS backend:** fine, but .NET gives stronger built-in security primitives and the team knows it better.
- **Firebase:** fast to start, but harder to enforce fine-grained authorization, field encryption and audit for health data.

## Consequences
- Native modules (cert pinning, FLAG_SECURE) need Expo config plugins or a dev client.
- Two languages (TS + C#). The shared API client is generated from OpenAPI to keep them in sync.

---
*ADR template: copy this file as `ADR-XXXX-<title>.md` with sections Context / Decision / Alternatives / Consequences.*
