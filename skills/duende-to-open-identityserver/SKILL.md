---
name: duende-to-open-identityserver
description: 'Migrate a .NET Duende.IdentityServer solution to Open.IdentityServer packages. Use when replacing Duende NuGet dependencies, updating namespaces/usings, and validating build/test after migration.'
argument-hint: 'Solution path and migration constraints (feeds, target Open.IdentityServer version, strict/no-warning build)'
---

# Duende To Open.IdentityServer Migration

## What This Skill Produces
A repeatable migration workflow that:
1. Replaces Duende.IdentityServer NuGet dependencies with Open.IdentityServer equivalents.
2. Replaces any remaining `Duende.IdentityServer` references with `Open.IdentityServer` across project files and source code.
3. Validates restore, build, and tests, then reports unresolved mapping gaps.

## Mandatory Replacement Rule
- Replace every reference to `Duende.IdentityServer` with `Open.IdentityServer`.
- Start the migration by running a repository-wide find and replace in all `*.cs`, `*.csproj`, and `*.packages.props` files: `Duende.IdentityServer` -> `Open.IdentityServer`.
- Applies to package references, namespaces, usings, config values, comments that drive behavior, and string literals used as identifiers.
- If an occurrence cannot be safely replaced 1:1, keep the best compatible `Open.IdentityServer` form and document the exception in the migration report.

## Change Scope Guard (Mandatory)
- Only make changes explicitly required by this migration:
- Package ID/version updates from Duende to Open.IdentityServer.
- Namespace/using/reference root replacements tied to this migration.
- Explicitly listed builder/config updates in this skill (for example `AddIdentityServer(...)` key management compatibility updates).
- Do not perform unrelated refactors, formatting-only churn, file moves/renames, behavioral rewrites, or dependency upgrades outside the mapped Open.IdentityServer migration set.
- If you notice unrelated issues, report them as follow-up items instead of changing them during this migration.

## When To Use
Use this skill when any of the following are true:
- A solution currently references `Duende.IdentityServer*` packages.
- The team is adopting Open.IdentityServer package feeds and package IDs.
- Compilation fails due to old Duende namespaces or moved types after package replacement.

## Required Inputs
- Repository root path.
- Open.IdentityServer package source/feed configuration.
- Target package version policy (latest stable by default; exact version only when explicitly requested).
- Whether the repository uses Central Package Management (CPM) via `Directory.Packages.props`.

## Migration Procedure

### 1. Global Find And Replace First
1. Run a repository-wide find and replace across all `*.cs`, `*.csproj`, and `*.packages.props` files:
- Find: `Duende.IdentityServer`
- Replace: `Open.IdentityServer`
2. Review replacements and keep any non-1:1 cases as targeted follow-up edits.

### 2. Baseline And Inventory
1. Confirm package management model:
- If CPM is used, package versions are managed in `Directory.Packages.props`.
- If not, versions are in individual `*.csproj` files.
2. Inventory Duende usage:
- Find package refs containing `Duende.IdentityServer` in `Directory.Packages.props` and `*.csproj`.
- Find code references for `using Duende`, `Duende.IdentityServer`, and known namespace roots.
3. Capture a baseline build/test snapshot after the global replacement pass.

### 3. Build Mapping Table Before Editing
Create a migration table with one row per Duende package:
- `oldPackageId`
- `newPackageId`
- `targetVersion`
- `notes` (deprecated, merged, no replacement, or API moved)

Decision point:
- If any package has no clear Open.IdentityServer equivalent, pause and mark as `manual-decision`.

### 3.1 Resolve Latest Open.IdentityServer Versions
1. Query configured NuGet feeds for latest stable versions of each `Open.IdentityServer*` package in the mapping table.
2. Use tooling such as `dotnet list package --outdated` (solution scope) or feed query APIs.
3. Record `currentVersion` and `latestStableVersion` in the mapping table.
4. Set `targetVersion = latestStableVersion` unless user requires a pinned version.

Decision points:
- If a package is not discoverable on current feeds, stop and report `missing-feed-or-auth` with the package ID.
- Do not guess or hardcode a version when latest cannot be verified from configured feeds.

### 3.2 Known Moved And Renamed Types
Some types that were in `Duende.IdentityServer.*` or `Duende.IdentityModel` have moved to a different sub-namespace in Open.IdentityServer, or have been replaced with .NET framework types. Apply these targeted fixes after the global replace if compile errors surface:

| Type | Old namespace | Replacement | Notes |
|---|---|---|---|
| `CryptoRandom` | `Duende.IdentityServer` (via IdentityModel) | `Open.IdentityServer.Utility` | Use `CryptoRandom.CreateUniqueId()` |
| `ToSha256()` extension | `Duende.IdentityServer` (via IdentityModel) | `Open.IdentityServer.Utility` | String hashing extension |
| `JwtClaimTypes`, `OidcConstants`, `StandardScopes` | `Duende.IdentityModel` | `Open.IdentityServer` | Available via global using |
| `IClock` | `Duende.IdentityServer.Services` | `System.TimeProvider` | Use modern .NET timing abstraction |

**Namespace additions needed:**
- Add `using Open.IdentityServer.Utility;` to any file using `CryptoRandom` or `ToSha256()`
- No additional using needed for `TimeProvider` (built-in type)

### 3.3 Known Missing API In Open.IdentityServer v1.0.0 (Server-Side Sessions)

The following types from Duende do **not** exist in Open.IdentityServer v1.0.0. When encountered, wrap the original code in `#if false ... #endif` with a stub replacement and a TODO comment:

| Missing type | Location | Action |
|---|---|---|
| `UserSession` | `Open.IdentityServer.Models` | `#if false` wrap |
| `QueryResult<T>` | `Open.IdentityServer.Models` | `#if false` wrap |
| `ISessionManagementService` | `Open.IdentityServer.Services` | `#if false` wrap; provide no-op stub |
| `SessionQuery` | `Open.IdentityServer.Models` | `#if false` wrap |
| `RemoveSessionsContext` | `Open.IdentityServer.Models` | `#if false` wrap |
| `.AddServerSideSessions()` | builder chain | Comment out with TODO |

**Session management files to disable (with `#if false` + stubs):**
- `Features/SessionManagement/ServerSessionManagementService.cs` — wrap original in `#if false`; provide a no-op stub `IServerSessionManagementService` with only `RemoveSession(sessionId)` and `RemoveAllSessions()` methods
- `Features/SessionManagement/UserProfileSessions.cshtml.cs` — wrap original in `#if false`; provide a stub `PageModel` with `OnGet()`/`OnPost()` returning `Page()`
- `Features/SessionManagement/UserProfileSessions.cshtml` — remove session table and pagination; replace with a Razor comment and an `_Alert` partial using `Localizer["NoDevices"]` / `Localizer["NoSessions_Message"]`
- All corresponding session management test files — wrap entire file in `#if false ... #endif`

**Also disable the Session Management nav menu item in `UserProfileService.GetMenu()`:**
```csharp
// TODO: Re-enable when Open.IdentityServer adds server-side sessions support
// menu.Add(new UserProfileMenuItem
// {
//     Section = UserProfileMenuItem.UserProfileMenuSection.ManageSessions,
//     TitleKey = "ManageSessions",
//     Page = SessionManagementConstants.UserProfilePage
// });
```
This prevents users from navigating to the stub sessions page.

### 4. Replace NuGet Package References
1. Update package IDs and versions using the mapping table and latest verified versions.
2. If CPM is enabled:
- Edit only `Directory.Packages.props` unless a project intentionally overrides versions.
3. If CPM is not enabled:
- Edit each affected `*.csproj` package reference.
4. Remove obsolete Duende package references.

Decision point:
- If both old and new packages are present, remove old Duende references to avoid ambiguous APIs.

### 5. Update IdentityServer Builder Configuration
1. Find all `AddIdentityServer(...)` setups in startup/composition code.
2. Remove LicenseKey configuration (`options.LicenseKey = ...`) — Open.IdentityServer does not require a license key.
3. Remove KeyManagement-enabling configuration (`options.KeyManagement.Enabled = true`) from the IdentityServer builder chain.
4. Comment out `.AddServerSideSessions()` in the builder chain with a TODO (see section 3.3).
5. Add the new compatibility extension method: `.AddCompatibilityKeyStores()`.
6. Ensure the resulting builder pipeline compiles and does not retain conflicting key management setup.

Decision point:
- If custom key storage logic exists, keep behavior-compatible wiring and document any manual follow-up needed.

### 6. Update Namespaces And Usings
1. Replace all reference roots:
- `Duende.IdentityServer` -> `Open.IdentityServer`.
2. Rebuild and inspect compile errors for moved/renamed namespaces.
3. Apply targeted namespace updates per compile errors and package docs.
4. Re-run until no `Duende.IdentityServer` references remain in source and project files.

Decision point:
- If a namespace is not a 1:1 rename, prefer compile-error-driven targeted edits over global replace.

### 7. Validate End To End
1. Restore dependencies from configured feeds.
2. Build solution with normal CI settings.
3. Run relevant test suites.
4. Confirm no `Duende.IdentityServer` package references remain.
5. Confirm no `Duende.IdentityServer` references remain anywhere in source or project files.
6. Confirm each `Open.IdentityServer*` package is at latest stable (or documented pinned exception).
7. Confirm `AddIdentityServer(...)` setup no longer enables KeyManagement and includes `.AddCompatibilityKeyStores()`.

### 8. Report Results
Produce a migration summary including:
- Replaced package list (old -> new).
- Files changed for namespace updates.
- Files changed for `AddIdentityServer(...)` builder updates.
- Any unresolved API/namespace incompatibilities.
- Follow-up actions (manual refactors, config changes, test updates).

## Quality Gates / Completion Checks
Migration is complete only if all are true:
1. Restore succeeds with intended package feeds.
2. Build succeeds without introducing new warnings treated as errors.
3. Target tests pass.
4. Zero remaining Duende package references in project/package files.
5. Zero remaining Duende namespace/usings in production code.
6. All `AddIdentityServer(...)` setups remove KeyManagement enabling and include `.AddCompatibilityKeyStores()`.
7. Any non-1:1 replacements are documented.
8. Open.IdentityServer package versions are latest stable or explicitly justified as pinned.

## Safe Execution Notes
- Prefer small, reviewable edits over broad blind replacement.
- Keep package replacement and namespace replacement as separate commits when possible.
- If CI enforces warning-as-error, resolve dependency advisories and feed issues during migration.
- Reject edits outside migration scope unless the user explicitly requests them.

## Example Prompts
- `/duende-to-open-identityserver Migrate this repository from Duende.IdentityServer to Open.IdentityServer using CPM in Directory.Packages.props.`
- `/duende-to-open-identityserver Replace Duende packages and namespaces, then run restore/build/tests and summarize unresolved API diffs.`
