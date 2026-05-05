# ADM Silver Layer Enterprise Coding Standard

Last updated: 2026-05-05
Scope: `apps-azure`, `apis-azure`, `adm-foundation-layer-agents-azure`

## Purpose

This document turns the current SAST-style review of `ADM_SILVER_LAYER` into coding and architecture standards that every team member must follow.

The goals are:

- reduce security risk fast
- make local/dev/prod behavior predictable
- stop duplication and config drift
- move the codebase toward enterprise-grade maintainability

## Risk Scoring Method

Risk scores below are estimated using the CVSS v3.1 base scoring model from FIRST:

- FIRST CVSS v3.1 Specification: https://www.first.org/cvss/v3-1/specification-document

Weakness mappings and justification references use:

- CWE-798 Hard-coded Credentials: https://cwe.mitre.org/data/definitions/798.html
- CWE-89 SQL Injection: https://cwe.mitre.org/data/definitions/89.html
- CWE-295 Improper Certificate Validation: https://cwe.mitre.org/data/definitions/295.html
- CWE-922 Insecure Storage of Sensitive Information: https://cwe.mitre.org/data/definitions/922.html
- OWASP HTML5 Security Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html

Scores are intentionally conservative and should be treated as engineering prioritization, not as audit-final numbers.

## Current Findings

| ID | Finding | Evidence | Weakness Mapping | Estimated CVSS | Severity |
|---|---|---|---|---|---|
| F1 | Hardcoded DB credentials in active runtime code | `apis-azure/backend/api/routes/utilities.py`, `apis-azure/backend/api/routes/push_to_metastore.py` | CWE-798 | 9.4 | Critical |
| F2 | TLS certificate verification disabled on backend-to-agent calls | `apis-azure/backend/api/routes/agents.py` uses `verify=False` | CWE-295 | 9.1 | Critical |
| F3 | Dynamic SQL and f-string SQL patterns in runtime paths | `apis-azure/backend/api/routes/utilities.py`, `apis-azure/backend/api/routes/push_to_metastore.py` | CWE-89 | 9.8 | Critical |
| F4 | Sensitive workflow state and auth-related cache stored in `localStorage` | `apps-azure/frontend/components/hooks/usePersistentState.ts`, `apps-azure/frontend/utils/restoreSessionData.ts`, `apps-azure/frontend/auth/msalConfig.ts` | CWE-922, OWASP HTML5 Storage guidance | 7.1 | High |
| F5 | Frontend API base URL is hardcoded and falls back to localhost runtime values instead of a proper frontend env contract | `apps-azure/frontend/apiService.ts`, `apps-azure/frontend/vite.config.ts` | Security misconfiguration / config drift | 7.4 | High |
| F6 | Frontend-to-backend and backend-to-agent flow relies on generic routing patterns instead of explicit endpoint contracts | `apps-azure/frontend/apiService.ts`, `apis-azure/backend/api/routes/agents.py`, `adm-foundation-layer-agents-azure/function_app.py` | Architectural isolation weakness | 7.6 | High |
| F7 | Overly permissive CORS/host configuration | `apis-azure/backend/core/config.py`, `apps-azure/frontend/vite.config.ts` | Security misconfiguration | 7.1 | High |
| F8 | Committed `.env` file in agent app | `adm-foundation-layer-agents-azure/.env` | Secret hygiene failure | 6.0 | Medium |
| F9 | Heavy connector duplication across agents | multiple identical `VectorDatabaseConnectors.py` and other connector copies | Maintainability and drift risk | 5.5 | Medium |

## Mandatory Standards

### 1. Secrets and Environment Management

Standard:

- No hardcoded secrets in source code, sample code, comments, or Dockerfiles.
- Every deployable unit must have its own config contract:
  - `apps-azure/.env`
  - `apis-azure/.env`
  - `adm-foundation-layer-agents-azure/.env`
- Production secrets must come from Azure Key Vault or platform secret injection, not from committed files.
- Only `.env.example` files may be committed.
- Frontend runtime URLs must come from `VITE_*` variables, not hardcoded fallbacks in source files or values embedded in `vite.config.ts`.

Why:

- Hardcoded credentials are directly mapped to CWE-798 and are high-likelihood compromise points.
- Separate env files remove cross-service coupling and make local development safer and clearer.
- Frontend URL hardcoding creates environment drift, makes deployments brittle, and can accidentally point dev/test builds at the wrong backend.

Team rule:

- If a developer needs to rotate or change a secret, it must not require a code change.

### 2. Browser Storage Policy

Standard:

- Do not store session payloads, workflow outputs, model artifacts, tokens, or user-sensitive metadata in `localStorage`.
- Use in-memory React state for active work.
- Use backend persistence as the source of truth for recoverable session state.
- If browser persistence is absolutely required for a non-sensitive UI preference, prefer `sessionStorage` and store the minimum possible data.
- MSAL cache should not default to `localStorage` unless security review explicitly approves it for this application.

Why:

- OWASP explicitly recommends avoiding sensitive data in `localStorage`.
- A single XSS issue can expose or poison all `localStorage` values.

Team rule:

- `localStorage` usage is banned unless the item is classified as `non-sensitive-ui-state`.

### 3. API and Agent Endpoint Design

Standard:

- The frontend must not rely on one generic fetch abstraction that simply passes an agent name to a shared backend route for execution.
- The frontend must call explicit business endpoints for each capability.
- Do not route all agent execution through a single generic Function endpoint.
- Each agent must expose an explicit endpoint or explicit API route contract.
- The API layer should call named agent endpoints with typed request/response models.
- Each agent must support its own config, logging, timeout, retry policy, and auth scope.
- The frontend must use a typed API client whose methods map to real API contracts, not generic `executeAgent(agentId, payload)` patterns for production workflows.

Why:

- Per-agent endpoints improve isolation, observability, throttling, rollback safety, and local development.
- A single dynamic endpoint becomes an oversized trust boundary and makes least-privilege difficult.
- Explicit UI-to-API contracts make it clear which screen calls which backend behavior, which reduces accidental misuse and makes review/test coverage deterministic.

Target pattern:

- `/api/v1/agents/data-profiler/execute`
- `/api/v1/agents/data-dictionary/execute`
- `/api/v1/agents/relationship-detector/execute`

Frontend pattern:

- `runDataProfiler(payload)`
- `runDataDictionary(payload)`
- `runRelationshipDetector(payload)`

### 4. Database Access and Query Safety

Standard:

- No f-string SQL in runtime code.
- All values must be parameterized.
- Dynamic table names or column names must come from a strict allowlist.
- Shared DB helpers must live in one module, not inside many routes.
- Use one consistent access style per service: either repository/service layer or clearly isolated query modules.

Why:

- This directly reduces SQL injection exposure and avoids hidden query drift.
- Centralizing query construction improves review quality and testability.

Team rule:

- Any SQL touching user input must be review-blocked unless parameters and identifier allowlists are visible.

### 5. Network and Transport Security

Standard:

- `verify=False` is prohibited.
- Internal service calls must validate TLS certificates.
- CORS origins must be explicit by environment.
- Vite `allowedHosts` must not contain wildcard-style bypass entries such as `"all"`.
- `vite.config.ts` must not act as the main source of backend endpoint configuration for the application.
- Reverse proxies in Vite are for local development only and must not substitute for explicit app runtime configuration.

Why:

- Disabling certificate validation maps to CWE-295 and permits impersonation of trusted services.
- Wildcard trust settings increase accidental exposure and weaken browser-side protection.
- Keeping endpoint configuration inside Vite config mixes build tooling with application configuration and makes production behavior less transparent.

### 6. Session Persistence Architecture

Standard:

- Do not store large evolving workflow documents as version-heavy JSON blobs in Postgres session tables when access patterns are document-oriented.
- Keep relational metastore entities in Postgres.
- Move agent session documents, intermediate workflow state, and replayable user edits to MongoDB.

Why:

- Session state in this codebase is document-centric, mutable, nested, and versioned.
- MongoDB is a better fit for partial reads, partial updates, flexible schema evolution, and large JSON workflow artifacts.

Recommended split:

- Postgres:
  - enterprise metastore
  - strongly relational reporting entities
  - lookup tables and referential workloads
- MongoDB:
  - `session_state`
  - `agent_inputs`
  - `agent_outputs`
  - user edit history
  - workflow snapshots

### 7. Vector Store Direction

Standard:

- Remove filesystem-bound LanceDB usage if MongoDB is adopted as the session platform.
- Prefer a managed vector capability aligned to the chosen document store, such as MongoDB Atlas Vector Search.

Why:

- One managed platform reduces operational sprawl.
- Local vector files are harder to secure, back up, and coordinate across environments.

Decision note:

- This is a platform decision, not just a code decision. Adopt it only if MongoDB is approved as a standard data platform.

### 8. Duplication and Naming Hygiene

Standard:

- No duplicate connector implementations across agents.
- Common connectors must be moved to a shared package.
- Avoid duplicate variables in the same scope and near-duplicate names that differ only by suffix or casing.
- Every new module must have a single clear owner responsibility.

Why:

- Duplication causes security fixes to drift and makes regression risk much higher.
- Variable duplication is a common source of wrong-config and wrong-payload bugs.

### 9. Logging and Data Exposure

Standard:

- Never log secrets, raw connection strings, tokens, or full session payloads.
- Use structured logging with `run_id`, `session_id`, `agent_name`, and `correlation_id`.
- Log status transitions, not full business payloads.

Why:

- Enterprise logs should support observability without becoming a second data leak surface.

### 10. CI/CD Guardrails

Standard:

- Every repo must enforce:
  - SAST scan
  - dependency audit
  - secret scan
  - lint
  - type check
  - unit tests for critical services
- Merge must fail on:
  - hardcoded credentials
  - `verify=False`
  - banned `localStorage` patterns
  - duplicate connector creation without approval

Why:

- Enterprise quality is sustained by guardrails, not by one-time cleanup.

## Target Architecture Decisions

### Approved Direction

1. Separate per-service env files and Key Vault-backed prod secrets.
2. Frontend `VITE_*` runtime config for API base URLs instead of hardcoded source values or Vite-config-driven endpoint behavior.
3. Per-agent endpoint contracts instead of one dynamic Function route.
4. Backend-persisted session recovery instead of browser `localStorage`.
5. MongoDB for session and workflow documents.
6. MongoDB vector capability instead of local LanceDB, if platform-approved.
7. Shared connector library instead of per-agent copies.

### Not Allowed Going Forward

- hardcoded passwords
- committed real `.env` files
- `verify=False`
- wildcard CORS for non-local environments
- hardcoded frontend API base URLs
- Vite config as the primary application endpoint registry
- generic UI fetch patterns that only pass agent names to shared execution APIs
- generic agent dispatch through one route for all new work
- storing workflow outputs in `localStorage`
- new copied connector modules

## Priority Remediation Plan

### P0

- remove hardcoded credentials
- remove hardcoded frontend API URLs and replace with explicit `VITE_*` env variables
- remove `verify=False`
- stop persisting sensitive workflow state in `localStorage`
- tighten CORS and host allowlists
- remove committed real env files and replace with `.env.example`

### P1

- split generic agent route into per-agent endpoints
- replace generic UI fetch-by-agent patterns with typed endpoint-specific API methods
- move DB access into shared repositories/helpers
- remove duplicated connector modules
- add secret scan and SAST gates in CI

### P2

- migrate session JSON storage from Postgres to MongoDB
- migrate LanceDB-backed vector usage to managed vector search on the approved platform

## Review Checklist for Every PR

- Are any secrets hardcoded?
- Is any sensitive state being stored in browser storage?
- Are all queries parameterized?
- Are TLS and certificate checks enabled?
- Is the endpoint explicit and bounded, not overly generic?
- Is config loaded from the correct service env source?
- Is any common logic being duplicated instead of reused?
- Are logs free of secrets and large raw payloads?

## References

- FIRST CVSS v3.1: https://www.first.org/cvss/v3-1/specification-document
- CWE-798: https://cwe.mitre.org/data/definitions/798.html
- CWE-89: https://cwe.mitre.org/data/definitions/89.html
- CWE-295: https://cwe.mitre.org/data/definitions/295.html
- CWE-922: https://cwe.mitre.org/data/definitions/922.html
- OWASP HTML5 Security Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html
