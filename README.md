# WhatMatters — AI Code Review Scrutiny Guide

A hierarchical checklist of every significant area in a modern .NET web application, designed to help developers decide **how carefully to scrutinize AI-generated code** in each area.

## Scrutiny Levels

| Symbol | Level | Meaning |
|--------|-------|---------|
| 🔴 | **Critical** | Read every line. Mistakes here cause breaches, data loss, or outages. |
| 🟠 | **High** | Review carefully. Errors here hurt correctness, contracts, or user trust. |
| 🟡 | **Medium** | Spot-check and run tests. Common patterns are usually safe but verify intent. |
| 🟢 | **Low** | Verify style and conventions only; logic is constrained by the framework. |
| ⚪ | **Instruction-file** | Push this rule into an `.editorconfig`, linter config, or Copilot instruction file. Developers should **stop spending review time here**. |

---

## 1. API (ASP.NET Core Web API)

### 1.1 ASP.NET Pipeline

| Item | Scrutiny | Notes |
|------|----------|-------|
| 1.1.1 Middleware registration order (auth, CORS, routing, exception handler) | 🔴 Critical | Wrong order silently bypasses security middleware. |
| 1.1.2 CORS policy configuration | 🔴 Critical | Overly permissive origins expose the API to cross-site attacks. |
| 1.1.3 Exception-handling middleware / global error filter | 🟠 High | Must not leak stack traces or internal details to callers. |
| 1.1.4 HTTPS enforcement and HSTS settings | 🔴 Critical | Must be on in production; verify `UseHttpsRedirection` and `UseHsts` are present. |
| 1.1.5 Rate limiting / throttling middleware | 🟠 High | Prevents abuse; confirm limits are sensible for your traffic profile. |
| 1.1.6 Health-check endpoints | 🟡 Medium | Confirm they don't expose sensitive internal state. |
| 1.1.7 Dependency injection / service registration | 🟡 Medium | Correct lifetime (Singleton vs Scoped vs Transient) prevents subtle bugs. |
| 1.1.8 Logging configuration (log levels, sinks) | 🟡 Medium | Sensitive data must never appear in logs; confirm PII scrubbing. |
| 1.1.9 `appsettings.json` structure and environment overrides | 🟠 High | No secrets in source; all secrets must come from Key Vault or environment variables. |

### 1.2 Security Model

| Item | Scrutiny | Notes |
|------|----------|-------|
| 1.2.1 Authentication scheme (JWT, cookie, OpenID Connect) | 🔴 Critical | Token validation parameters (issuer, audience, signing key, expiry) must be explicit. |
| 1.2.2 Authorization policies and role/claim definitions | 🔴 Critical | Policies must be named and registered; no magic strings scattered in code. |
| 1.2.3 Refresh-token handling and revocation | 🔴 Critical | Tokens must be revocable; storage must be secure. |
| 1.2.4 Password hashing and credential storage | 🔴 Critical | Use ASP.NET Core Identity or an equivalent; never roll your own. |
| 1.2.5 Antiforgery (CSRF) configuration | 🔴 Critical | Required for cookie-based auth; verify middleware and `[ValidateAntiForgeryToken]`. |
| 1.2.6 Secret retrieval from Azure Key Vault at startup | 🔴 Critical | Connection strings, signing keys, and API keys must come from Key Vault. |
| 1.2.7 Managed Identity configuration | 🟠 High | Prefer Managed Identity over service-principal secrets wherever possible. |

### 1.3 Endpoints

#### 1.3.1 Endpoint Security Attributes

| Item | Scrutiny | Notes |
|------|----------|-------|
| Every endpoint has an explicit `[Authorize]` or `[AllowAnonymous]` attribute | 🔴 Critical | Missing `[Authorize]` is a common AI mistake; audit every controller/minimal-API route. |
| Authorization policy or role is the correct one for the action | 🔴 Critical | AI often copies the nearest attribute; verify policy matches the resource. |
| Admin-only endpoints are in a separate route area with a policy | 🔴 Critical | Segregate admin surface; don't rely solely on role checks on individual actions. |

#### 1.3.2 Endpoint Input Validation

| Item | Scrutiny | Notes |
|------|----------|-------|
| Model validation with Data Annotations or FluentValidation | 🟠 High | Confirm `ModelState.IsValid` or automatic 400 responses are wired up. |
| Query-string and route parameters are validated and range-checked | 🟠 High | Never trust client-supplied IDs or filter values without validation. |
| File upload size and type restrictions | 🔴 Critical | Unrestricted uploads are a common attack vector. |
| Paging parameters have upper-bound limits | 🟡 Medium | Prevents denial-of-service via huge result sets. |

#### 1.3.3 DTO Mapping (Preventing Data Exposure)

| Item | Scrutiny | Notes |
|------|----------|-------|
| Response DTOs contain only fields the caller should see | 🔴 Critical | Never return EF entities directly; verify every DTO property is intentional. |
| DTO→Entity mapping does not allow mass assignment of privileged fields | 🔴 Critical | AI often maps all properties; review that `Id`, `CreatedBy`, `IsAdmin`, etc. are excluded. |
| Mapping library configuration (AutoMapper profiles, Mapster config) | 🟠 High | Confirm `ReverseMap` or wildcard mappings don't accidentally expose internal fields. |
| Null / optional field handling in DTOs | 🟡 Medium | Inconsistent nullability can leak structure or cause downstream errors. |

#### 1.3.4 Error Responses

| Item | Scrutiny | Notes |
|------|----------|-------|
| Error responses use ProblemDetails (RFC 7807) | 🟡 Medium | Consistent format; confirm no raw exception messages in production responses. |
| 401 vs 403 used correctly | 🟠 High | Conflating these leaks information about resource existence. |

### 1.4 NuGet Packages

| Item | Scrutiny | Notes |
|------|----------|-------|
| 1.4.1 Package versions pinned in `.csproj` | 🟠 High | Floating versions (`*`) can introduce breaking changes silently. |
| 1.4.2 Known-vulnerable packages (GitHub Dependabot / `dotnet list package --vulnerable`) | 🔴 Critical | Run before every release; block PRs with known critical CVEs. |
| 1.4.3 Packages with security implications reviewed on introduction (auth, crypto, serialization) | 🔴 Critical | AI sometimes adds an unfamiliar package; vet it before merging. |
| 1.4.4 Unused package references cleaned up | ⚪ Instruction-file | Enforce with a linter or Roslyn analyzer rule; not a manual review concern. |
| 1.4.5 Consistent transitive dependency versions | 🟡 Medium | Binding redirects / `<PackageReference>` overrides checked after major updates. |

### 1.5 Azure Service Integrations (API-side)

| Item | Scrutiny | Notes |
|------|----------|-------|
| 1.5.1 Azure Blob Storage — SAS token scope and expiry | 🔴 Critical | Tokens must be scoped to minimum container/blob and expire promptly. |
| 1.5.2 Azure Blob Storage — public access disabled on containers | 🔴 Critical | Default to private; verify no container is inadvertently public. |
| 1.5.3 Azure AI / OpenAI — prompt injection guard | 🔴 Critical | User input passed to AI prompts must be sanitized or sandboxed. |
| 1.5.4 Azure AI — response content used safely in the UI | 🔴 Critical | AI responses rendered as HTML must be escaped to prevent stored XSS. |
| 1.5.5 Azure Service Bus / Event Grid — message authentication | 🟠 High | Verify subscribers validate message source and schema. |
| 1.5.6 Azure Cognitive Search — index field permissions | 🟠 High | Multi-tenant indexes must filter by tenant ID at query time. |
| 1.5.7 Retry / circuit-breaker policies on all Azure SDK calls | 🟡 Medium | Polly or Azure SDK built-in retry; avoids cascading failures. |

---

## 2. Website (Vue / React + Vuetify / Tailwind)

### 2.1 Pages

#### 2.1.1 Marketing / Public Pages

| Item | Scrutiny | Notes |
|------|----------|-------|
| No authenticated data rendered server-side into public pages | 🔴 Critical | Server-side rendering must not leak session data into public HTML. |
| Third-party scripts (analytics, chat) loaded via CSP-safe pattern | 🟠 High | Inline scripts and `eval` blocked by Content Security Policy. |
| SEO metadata does not expose internal identifiers | 🟡 Medium | Check `<meta>` tags, canonical URLs, and structured data. |

#### 2.1.2 Authentication Pages (Login, Register, Forgot Password, MFA)

| Item | Scrutiny | Notes |
|------|----------|-------|
| Login form does not leak whether an email exists (timing/message) | 🔴 Critical | Generic "invalid credentials" message required. |
| Password field autocomplete and storage attributes correct | 🟠 High | `autocomplete="current-password"` or `"new-password"` as appropriate. |
| MFA/2FA enforcement for privileged accounts | 🔴 Critical | Verify MFA cannot be bypassed by navigating directly to post-login routes. |
| Auth tokens stored securely (HttpOnly cookie preferred over localStorage) | 🔴 Critical | AI often defaults to localStorage; this exposes tokens to XSS. |
| Rate limiting / lockout on login endpoint (frontend must respect 429) | 🟠 High | UI should surface lockout messaging clearly. |
| Post-login redirect validates target URL (open redirect prevention) | 🔴 Critical | Never redirect to an arbitrary `?returnUrl=` without an allow-list. |

#### 2.1.3 Main Application Pages

| Item | Scrutiny | Notes |
|------|----------|-------|
| Route guards enforce authentication and authorization before rendering | 🔴 Critical | All protected routes must check auth state; not just hide a nav link. |
| Tenant / user scope applied to every data fetch | 🔴 Critical | Verify every API call includes the correct user/tenant context. |
| Framework component usage follows library conventions | 🟡 Medium | Vuetify/Tailwind used consistently; no ad-hoc inline styles that conflict. |
| Accessibility (ARIA roles, keyboard navigation) on interactive elements | 🟡 Medium | Especially important for forms and modals. |
| Page titles and breadcrumbs updated on route change | ⚪ Instruction-file | Define a standard pattern in the component guide; don't review manually. |

#### 2.1.4 Error / Status Pages (404, 403, 500)

| Item | Scrutiny | Notes |
|------|----------|-------|
| Error pages do not render internal error details | 🔴 Critical | Stack traces or exception messages must never reach the browser in production. |
| Correct HTTP status codes returned (not always 200) | 🟠 High | SSR frameworks sometimes return 200 for error pages. |

### 2.2 NPM Packages

| Item | Scrutiny | Notes |
|------|----------|-------|
| 2.2.1 Package versions pinned in `package.json` (no `^` or `~` in production deps) | 🟠 High | Floating ranges cause non-deterministic builds. |
| 2.2.2 Known-vulnerable packages (`npm audit` / Dependabot) | 🔴 Critical | Run in CI; block on critical severity. |
| 2.2.3 New packages with security implications reviewed on introduction | 🔴 Critical | Auth helpers, crypto libs, HTTP clients must be vetted. |
| 2.2.4 Bundle size impact of new packages | 🟡 Medium | Check with `webpack-bundle-analyzer` or Vite's rollup visualizer. |
| 2.2.5 License compliance | 🟡 Medium | GPL/LGPL packages in a commercial product need legal sign-off. |
| 2.2.6 Sorting and formatting of `package.json` | ⚪ Instruction-file | Automate with `sort-package-json`; not a manual review concern. |

### 2.3 Component Architecture

| Item | Scrutiny | Notes |
|------|----------|-------|
| 2.3.1 Components receive only the props they need (no prop drilling of full objects) | 🟡 Medium | Prevents accidental exposure of sensitive fields in child components. |
| 2.3.2 Sensitive data (tokens, PII) not stored in component local state beyond its use | 🔴 Critical | Minimize time-in-memory for credentials and PII. |
| 2.3.3 Event emitter payloads do not carry full server response objects | 🟠 High | Emit only the specific fields needed by the parent. |
| 2.3.4 v-html / dangerouslySetInnerHTML use is explicitly justified and content sanitized | 🔴 Critical | Every use is a potential XSS vector; require a comment explaining the sanitization. |
| 2.3.5 Component naming and file structure follow the project convention | ⚪ Instruction-file | Enforce with an ESLint plugin rule. |
| 2.3.6 Reusable components placed in a shared library | ⚪ Instruction-file | Define in the contributing guide; not a per-PR review concern. |

### 2.4 Client-Side Models / State Management

| Item | Scrutiny | Notes |
|------|----------|-------|
| 2.4.1 Store (Pinia / Redux / Vuex) does not persist sensitive fields to localStorage | 🔴 Critical | Tokens, PII, and role data must not survive a page refresh in plaintext. |
| 2.4.2 Actions/thunks that call the API validate the response shape before committing | 🟠 High | Unexpected fields from the server can corrupt client state. |
| 2.4.3 Client-side authorization checks (show/hide UI) are not the only access control | 🔴 Critical | Server must enforce; client checks are UX only. |
| 2.4.4 TypeScript types for API responses match the actual DTOs | 🟠 High | Drift between client and server models causes runtime errors. |
| 2.4.5 Stale-cache invalidation strategy on mutation | 🟡 Medium | Verify that writes trigger appropriate cache busts. |

### 2.5 Frontend Security

| Item | Scrutiny | Notes |
|------|----------|-------|
| 2.5.1 Content Security Policy headers set by the server | 🔴 Critical | Blocks XSS, clickjacking, and data exfiltration; must be explicit, not `unsafe-inline`. |
| 2.5.2 Subresource Integrity (SRI) for any CDN-loaded scripts | 🔴 Critical | Without SRI, a compromised CDN can inject code. |
| 2.5.3 User-supplied content rendered as text, not HTML | 🔴 Critical | Framework escaping is default; verify any bypass is intentional and sanitized. |
| 2.5.4 Sensitive query-string parameters avoided | 🟠 High | Tokens or PII in URLs appear in browser history and server logs. |
| 2.5.5 Environment variables / build-time secrets not embedded in the bundle | 🔴 Critical | `VITE_*` / `REACT_APP_*` values are visible to anyone who downloads the JS. |

### 2.6 API Communication Layer

| Item | Scrutiny | Notes |
|------|----------|-------|
| 2.6.1 Authorization header / cookie attached correctly to all authenticated requests | 🔴 Critical | Missing credentials on some routes is a common AI oversight. |
| 2.6.2 Axios / Fetch interceptor handles 401 (token refresh or logout) | 🟠 High | Silent 401 failures leave users in broken state. |
| 2.6.3 API base URL comes from environment config, not hardcoded | 🟡 Medium | Hardcoded URLs break in non-production environments. |
| 2.6.4 Errors surfaced to the user with a meaningful message | 🟡 Medium | Raw error objects must not be displayed. |

---

## 3. Data Project (EF Core + Business Logic Library)

### 3.1 Entity / Class Design

| Item | Scrutiny | Notes |
|------|----------|-------|
| 3.1.1 Core domain model changes | 🔴 Critical | Adding/removing/renaming fields has cascading effects on migrations, APIs, and UIs. |
| 3.1.2 Foreign key relationships and cascade delete rules | 🔴 Critical | Incorrect cascade rules cause unintended data deletion. |
| 3.1.3 Soft-delete pattern applied consistently | 🟠 High | Global query filter must be present so soft-deleted rows are never accidentally returned. |
| 3.1.4 Multi-tenancy: every entity with tenant scope has a `TenantId` FK and global query filter | 🔴 Critical | Missing tenant filter is a data isolation breach. |
| 3.1.5 Audit fields (`CreatedAt`, `UpdatedAt`, `CreatedBy`) set via base class or interceptor | 🟡 Medium | Consistent if done in one place; verify the interceptor is registered. |
| 3.1.6 Navigation property configuration (lazy vs eager loading) | 🟠 High | Lazy loading can cause N+1 query explosions; prefer explicit `.Include()`. |
| 3.1.7 Owned entity and value object usage | 🟡 Medium | Verify `OwnsOne`/`OwnsMany` config matches the intended table structure. |
| 3.1.8 Property naming and XML doc comments | ⚪ Instruction-file | Enforce with Roslyn analyzer + style guide; not a manual review concern. |

### 3.2 EF Core Migrations

| Item | Scrutiny | Notes |
|------|----------|-------|
| 3.2.1 Migration makes no unintended table or column drops | 🔴 Critical | AI sometimes generates a drop when a rename was intended. Read every `Down` method. |
| 3.2.2 Data migration (seeding, backfilling) in a migration script is safe for production data | 🔴 Critical | Verify the script handles NULLs, large row counts, and lock duration. |
| 3.2.3 Migration is reversible (the `Down` method is correct) | 🟠 High | Irreversible migrations complicate rollbacks. |
| 3.2.4 New indexes exist for foreign keys and common query predicates | 🟡 Medium | EF does not auto-index every FK; missing indexes hurt performance. |
| 3.2.5 Migration applied via CI, not manually in production | 🟡 Medium | Verify the deployment pipeline runs migrations before the new API starts. |
| 3.2.6 Auto-generated migration file contents (boilerplate) | ⚪ Instruction-file | Trust the EF tooling; review only the non-boilerplate changes listed above. |

### 3.3 Business Logic and Services

| Item | Scrutiny | Notes |
|------|----------|-------|
| 3.3.1 Authorization checks performed in the service layer (not just the controller) | 🔴 Critical | Services called from multiple entry points (API, background jobs) must be self-defending. |
| 3.3.2 Ownership / tenancy validated before any read or write | 🔴 Critical | "Can this user access this resource?" must be checked in the service, not assumed from the route. |
| 3.3.3 Domain rule violations return typed exceptions or `Result<T>`, not HTTP status codes | 🟠 High | Services must not depend on the HTTP layer. |
| 3.3.4 Transaction boundaries for multi-step operations | 🟠 High | Partial writes on failure can corrupt state. |
| 3.3.5 Background / queued job authorization | 🔴 Critical | Jobs triggered by user actions must execute with that user's permissions, not elevated service permissions. |
| 3.3.6 Unit of Work / Repository pattern correctly scoped | 🟡 Medium | Shared `DbContext` across requests causes concurrency bugs. |
| 3.3.7 Method names and XML doc | ⚪ Instruction-file | Style enforced by analyzer; not a manual review concern. |

### 3.4 Data Validation

| Item | Scrutiny | Notes |
|------|----------|-------|
| 3.4.1 Entity validation (FluentValidation / Data Annotations) enforced before `SaveChangesAsync` | 🟠 High | DB-level constraints are the last resort; application-level validation must fire first. |
| 3.4.2 Input strings sanitized where stored value may be rendered as HTML | 🔴 Critical | Stored XSS via a business-layer field that bypasses DTO validation. |
| 3.4.3 Numeric ranges and enum values validated | 🟡 Medium | Invalid enum values cause runtime exceptions at serialization time. |

### 3.5 Security Aspects

| Item | Scrutiny | Notes |
|------|----------|-------|
| 3.5.1 Raw SQL / `FromSqlRaw` usage parameterized correctly | 🔴 Critical | Any string interpolation into raw SQL is a SQL injection risk. |
| 3.5.2 EF LINQ queries do not accidentally return cross-tenant rows | 🔴 Critical | Double-check that global query filters are not disabled (`IgnoreQueryFilters`). |
| 3.5.3 Sensitive fields (SSN, payment info) encrypted at rest | 🔴 Critical | Verify column-level encryption or value converter is applied. |
| 3.5.4 Logging of entity changes does not include sensitive field values | 🔴 Critical | Audit logs should record "field changed" but not old/new values for PII fields. |

---

## 4. Deployments & Infrastructure

### 4.1 CI/CD Pipeline (YAML — GitHub Actions / Azure DevOps)

| Item | Scrutiny | Notes |
|------|----------|-------|
| 4.1.1 Secrets referenced via vault / environment secret, never hard-coded | 🔴 Critical | Any `echo $SECRET` in a step that logs to stdout is a leak. |
| 4.1.2 Least-privilege service connection / identity used for deployment | 🔴 Critical | Deployment identity must not have subscription-owner rights. |
| 4.1.3 Approval gates on production deployments | 🔴 Critical | No automated push-to-prod without a human approval step. |
| 4.1.4 Security scan steps (dependency audit, SAST, container scan) run before deploy | 🔴 Critical | Confirm `npm audit`, `dotnet list package --vulnerable`, and any SAST tool run in the pipeline. |
| 4.1.5 Migration step order relative to API deployment (migrate first, then deploy) | 🟠 High | Deploy before migrate can cause runtime errors; verify ordering. |
| 4.1.6 Environment promotion strategy (dev → staging → prod) | 🟠 High | Same artifact promoted, not rebuilt per environment. |
| 4.1.7 Pipeline caching for NuGet and npm | ⚪ Instruction-file | Standard pattern; define once in the pipeline template. |
| 4.1.8 Indentation and YAML formatting | ⚪ Instruction-file | Use a YAML linter in the pipeline; not a manual review concern. |

### 4.2 Infrastructure as Code (Terraform / Bicep)

| Item | Scrutiny | Notes |
|------|----------|-------|
| 4.2.1 Network access rules (firewall, NSG, private endpoints) | 🔴 Critical | Default "allow all" rules are dangerous; verify explicit allow-lists. |
| 4.2.2 Azure Key Vault access policies / RBAC assignments | 🔴 Critical | Only the identities that need secrets should have `Get`/`List`; no wildcard. |
| 4.2.3 Storage account public access disabled, HTTPS-only enforced | 🔴 Critical | Default storage settings are often permissive. |
| 4.2.4 Diagnostic settings / Azure Monitor logging enabled on all resources | 🟠 High | Without logging, incidents cannot be investigated. |
| 4.2.5 Azure App Service / AKS VNET integration | 🟠 High | API must not be directly internet-facing if a gateway or Front Door is used. |
| 4.2.6 Terraform state stored in secured, remote backend with locking | 🟠 High | Local state or unprotected remote state is a secret-leak risk. |
| 4.2.7 Resource tagging for cost allocation | ⚪ Instruction-file | Define a tagging module; not a per-PR review concern. |
| 4.2.8 Resource naming conventions | ⚪ Instruction-file | Define in a naming module or variable; not a per-PR review concern. |

### 4.3 Azure Key Vault

| Item | Scrutiny | Notes |
|------|----------|-------|
| 4.3.1 All application secrets (connection strings, API keys, signing keys) stored in Key Vault | 🔴 Critical | No hard-coded secrets in `appsettings.json`, environment variable plain text, or pipeline variables in the clear. |
| 4.3.2 Key rotation policy configured for cryptographic keys | 🔴 Critical | Keys must have an expiry and a rotation reminder or automation. |
| 4.3.3 Soft-delete and purge protection enabled | 🟠 High | Prevents accidental or malicious permanent deletion of secrets. |
| 4.3.4 Access audit logging sent to Log Analytics | 🟠 High | Required to detect unauthorized secret access. |

### 4.4 Azure Blob Storage

| Item | Scrutiny | Notes |
|------|----------|-------|
| 4.4.1 Containers set to private; no anonymous read access | 🔴 Critical | Verify at the Terraform/Bicep level, not just the SDK call. |
| 4.4.2 SAS tokens scoped to minimum permissions and short-lived | 🔴 Critical | Overly broad or long-lived SAS tokens are equivalent to public access. |
| 4.4.3 User-Delegation SAS preferred over account-key SAS | 🟠 High | Account-key SAS grants broad implicit access if the key is compromised. |
| 4.4.4 Blob lifecycle policy for expiry of temporary/upload blobs | 🟡 Medium | Prevents unbounded storage growth. |

### 4.5 Azure AI / OpenAI Integration

| Item | Scrutiny | Notes |
|------|----------|-------|
| 4.5.1 API key stored in Key Vault; retrieved via Managed Identity | 🔴 Critical | Never hard-code AI API keys. |
| 4.5.2 Prompt templates reviewed for injection risk before shipping | 🔴 Critical | Untrusted user input concatenated into prompts enables prompt injection. |
| 4.5.3 Content filtering / moderation layer on AI responses before display | 🔴 Critical | Prevent AI responses from surfacing harmful or confidential training data. |
| 4.5.4 Token-usage limits configured to prevent runaway costs | 🟠 High | Set `max_tokens` and monitor usage; alert on anomalies. |
| 4.5.5 AI model version pinned, not "latest" | 🟡 Medium | Model version drift can silently change behavior; pin and test on upgrade. |

---

## 5. General Architectural Concerns

| Item | Scrutiny | Notes |
|------|----------|-------|
| 5.1 Cross-cutting security policy: auth enforced at API boundary, business, and data layers | 🔴 Critical | Defense in depth; no single layer can be the only check. |
| 5.2 Multi-tenancy isolation strategy applied consistently across all layers | 🔴 Critical | Verify tenant scope is threaded from HTTP request through service to DB query. |
| 5.3 Secrets management strategy (Key Vault + Managed Identity as standard) | 🔴 Critical | Any deviation requires explicit justification. |
| 5.4 Error and exception handling strategy consistent across API, services, and background jobs | 🟠 High | No swallowed exceptions; structured logging on all errors. |
| 5.5 Observability: structured logging, distributed tracing (App Insights / OTLP), and alerting | 🟠 High | Must be in place before production; retroactively adding it is painful. |
| 5.6 Performance: N+1 query patterns and unbounded result sets | 🟠 High | AI-generated LINQ is prone to N+1; always review generated queries with EF logging in dev. |
| 5.7 API versioning strategy | 🟡 Medium | Breaking changes to the API must be versioned; clients cannot all be updated simultaneously. |
| 5.8 Dependency update / patching cadence | 🟡 Medium | Automate with Dependabot; review weekly for critical CVEs. |
| 5.9 Code formatting and style rules | ⚪ Instruction-file | Enforce with `.editorconfig`, ESLint, Prettier, and Roslyn analyzers. Stop reviewing manually. |
| 5.10 XML / JSDoc comment completeness | ⚪ Instruction-file | Enforce with a documentation linter; not a manual review concern. |
| 5.11 Unit test file and method naming | ⚪ Instruction-file | Define a naming convention in the contributing guide; not a per-PR concern. |
| 5.12 Folder and project structure conventions | ⚪ Instruction-file | Document once in the architecture guide; not a per-PR concern. |

---

## What Belongs in Instruction / Copilot Files (Not Manual Review)

The following are **valid engineering standards** that consume review time without catching meaningful bugs. Automate them with tooling and move them into Copilot instruction files, `.editorconfig`, or linter configs so that reviewers can stop spending attention on them:

- **Code formatting** — indentation, brace style, blank lines (`.editorconfig` + Prettier + `dotnet format`)
- **Naming conventions** — variable/method/class casing (Roslyn analyzers + ESLint)
- **XML / JSDoc comment style** — enforce with documentation linters
- **`package.json` field ordering** — `sort-package-json` as a pre-commit hook
- **YAML / JSON formatting** — `yamllint` + Prettier in CI
- **Terraform resource naming and tagging** — naming module + `tflint` rules
- **Standard boilerplate migrations** — trust EF tooling; only review the non-boilerplate diff
- **Unit test file naming and placement** — document once in the contributing guide
- **Standard DI registration patterns** — document the pattern; flag only deviations
- **Reusable component placement and folder structure** — document in the component guide
- **Console.Log / Debug.Log removal** — ESLint `no-console` rule; automated
- **`TODO` comment tracking** — enforce with an issue tracker, not a review nit

---

## Using the Interactive Review Prompt

This repo includes a [GitHub Copilot prompt](.github/prompts/code-review.prompt.md) that interactively guides you through the review process.

### How to use it

1. Open a Copilot Chat session in VS Code or GitHub.com
2. Type `@workspace /code-review` (if using the prompt file as a reusable prompt) or reference the file directly
3. Share the code you want reviewed — a diff, snippet, or description of changes
4. Copilot will categorize the changes, apply the appropriate scrutiny level, and walk you through the critical items first

The prompt distills everything in this document into a conversational flow so you don't have to remember the full checklist yourself.
