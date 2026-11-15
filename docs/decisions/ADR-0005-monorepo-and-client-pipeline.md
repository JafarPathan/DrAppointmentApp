# ADR-0005: Monorepo tooling and typed API client pipeline

- **Status:** Accepted
- **Date:** 2026-11-16
- **Debate:** [DEBATE.md Thread 004](../../DEBATE.md) (🟢 consensus between Claude Code and Antigravity)
- **Extends:** ADR-0001 (tech stack), PROGRESS 0.9 (scaffold) and 0.10 (CI)

## Context

One repo holds an Expo app (`apps/mobile`), a Vite web dashboard (`apps/doctor-web`), a shared TypeScript API client (`packages/api-client`) and an ASP.NET Core 10 backend (`backend/`). We need fast, cached builds, and React Native's Metro bundler must work in a monorepo. The frontend types must never drift from the backend API, and nobody should hand-write fetch calls (AGENTS.md Rule 3).

## Decision

### 1. Workspaces: pnpm with hoisted node_modules
`pnpm-workspace.yaml` at the repo root:
```yaml
packages:
  - "apps/*"
  - "packages/*"
nodeLinker: hoisted   # React Native / Metro expect a flat node_modules (Expo monorepo guide)
```
Check the current Expo monorepo guide when scaffolding. If the pnpm version in use expects this setting elsewhere, follow the guide.

### 2. Task runner: Turborepo
`turbo.json` caches `lint`, `typecheck`, `test` and `build` across packages. The .NET backend sits outside the pnpm workspace but is wired in through a thin `backend/package.json` whose scripts call `dotnet build` / `dotnet test`, so `turbo run test` covers everything. The backend keeps its own `.sln`.

### 3. OpenAPI document, generated at build time and committed
- Use the built-in **`Microsoft.AspNetCore.OpenApi`** (not Swashbuckle) and target **OpenAPI 3.0** for the widest generator compatibility.
- Add **`Microsoft.Extensions.ApiDescription.Server`** with `OpenApiGenerateDocuments=true` and `OpenApiDocumentsDirectory=../openapi`, so every `dotnet build` writes **`backend/openapi/v1.json`**.
- **Commit `v1.json`**, so frontend devs don't need the .NET SDK to regenerate the client.

### 4. Build-time detection: startup must be "inert-safe"
Build-time generation **launches the app's entry point** through the `GetDocument.Insider` tool. Anything that needs secrets or a network at startup must be skipped:
```csharp
// BuildContext.cs
public static class BuildContext
{
    // Build-time OpenAPI generation runs the app via the GetDocument.Insider tool.
    // This name is an implementation detail; CI fails loudly if generation breaks (see §6).
    public static bool IsOpenApiGeneration =>
        Assembly.GetEntryAssembly()?.GetName().Name == "GetDocument.Insider";
}

// Program.cs
if (!BuildContext.IsOpenApiGeneration)
{
    // Key Vault, DbContext health checks, migrations, Hangfire, SMS/push clients, SignalR backplane
}
```
**Never** detect this with a made-up environment name. Only the entry-assembly check is documented.

### 5. Client generation: Orval only
`packages/api-client/orval.config.ts` defines **two outputs** from `backend/openapi/v1.json`:
- `api-query`: `client: 'react-query'` generates TanStack Query hooks + TS types.
- `api-zod`: `client: 'zod'` generates Zod schemas, shared by mobile and web forms.

A **custom mutator** (`src/http.ts`) is the only place HTTP happens:
- injects the in-memory access token;
- on 401: refresh once (rotation, SECURITY §2), retry once, otherwise sign out;
- adds an `Idempotency-Key` header on booking POSTs;
- passes these through explicit options or headers, **not React Query `meta`** (Orval issue #1317: `meta` isn't forwarded).

`openapi-typescript` isn't used (Orval already generates types).

### 6. CI checks
1. Build the backend → `backend/openapi/v1.json` **must exist** (catches a broken `GetDocument.Insider` detection).
2. Run Orval → `git diff --exit-code backend/openapi packages/api-client` (**zero drift**).
3. `turbo run lint typecheck test`.

### 7. 🔒 API docs UI
Scalar or Swagger UI is mapped **only in Development / local**. Production serves no OpenAPI UI. *(Not debated: whether the raw OpenAPI JSON endpoint is mapped in production. Recommendation: don't map it, since the committed file is the source for client generation. Raise a thread if anyone needs it.)*

## Alternatives considered

| Option | Why not |
|---|---|
| npm workspaces | No task caching; slower |
| Yarn Berry (PnP) | PnP fights React Native; team unfamiliar |
| Nx | Heavier learning curve for a side project |
| openapi-typescript + hand-written hooks | Manual hooks drift from the API |
| Swashbuckle | No longer in the default templates; the built-in package is the supported path |
| OpenAPI 3.1 | Newer default, but some generator tooling is less stable. Revisit once Orval's 3.1 support is verified |

## Consequences

- ✅ One `pnpm install`, cached builds, typed hooks shared by mobile and web, drift caught in CI.
- ✅ CI and builds never need secrets or a database.
- ⚠️ Hoisting gives up some of pnpm's strict dependency isolation.
- ⚠️ `GetDocument.Insider` is an implementation detail. CI check §6.1 guards it.

## Docs to update

- `docs/ARCHITECTURE.md` §2: pnpm (hoisted), Turborepo, Orval, OpenAPI 3.0, build-time document.
- `README.md` layout: `backend/openapi/`.
- `CONTRIBUTING.md` setup: pnpm.
- `docs/PROGRESS.md` 0.9 / 0.10: scaffold + CI checks above.

## Sources

- Expo, Work with monorepos: https://docs.expo.dev/guides/monorepos/
- Callstack, React Native monorepo with pnpm: https://callstack.com/blog/react-native-monorepo-with-pnpm-workspaces
- Microsoft Learn, Generate OpenAPI documents (build-time): https://learn.microsoft.com/aspnet/core/fundamentals/openapi/aspnetcore-openapi
- Microsoft Learn, Using OpenAPI documents: https://learn.microsoft.com/aspnet/core/fundamentals/openapi/using-openapi-documents
- Build-time detection in .NET 10 (community write-up): https://dev.to/nausaf/accept-the-official-hack-build-time-openapi-detection-in-net-10-minimal-apis-25bc
- Orval, custom instance: https://orval.dev/guides/custom-axios
- Orval configuration reference: https://orval.dev/reference/configuration
- Orval issue #1317: https://www.github.com/anymaniax/orval/issues/1317
