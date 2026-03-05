# ADR-004: Authentication — Form Login + API Key

- **Status:** Accepted
- **Created by:** Jesper Pedersen
- **Date:** 2026-03-04
- **Stakeholders:** Development team, Product Manager, site developers

## Context and Drivers

The application has two distinct entry points:

1. **Dashboard** — Web UI used by Product Managers to triage feedback
2. **API** — Receives feedback submissions from staging site widgets

Each entry point has different authentication requirements and threat
models.

### Functional drivers

- Dashboard users need session-based login with redirects
- API clients (widgets) need stateless authentication per request
- API keys must be scoped per project (one key per staging site)
- API keys are write-only (submit feedback only, no read access)

### Non-functional drivers

- Simplicity (avoid OAuth complexity for an internal tool)
- Security proportional to the threat model (staging-only widget)

## Options Considered

### Option A: Unified OAuth2 / JWT

**Pros:**

- Single auth mechanism for all entry points
- Industry standard

**Cons:**

- Significant complexity for an internal staging tool
- Widget would need OAuth token refresh logic
- Overkill for write-only API access

### Option B: Form login + API key (chosen)

**Pros:**

- Dashboard: standard Symfony `form_login` with session handling
- API: simple `X-Api-Key` header, stateless, no token refresh
- API keys scoped per project with write-only permissions
- Minimal widget code changes (add one header)

**Cons:**

- API key is visible in page source (embedded in widget JS config)
- Two auth mechanisms to maintain

## Decision

Use **Symfony `form_login`** for the dashboard and a **custom API key
authenticator** for the API.

API keys use a `tf_` prefix for identification, are stored hashed in
the database, and are displayed once at project creation time. Keys
are write-only — they can only submit feedback for their associated
project, not read existing items.

The API key visibility in page source is acceptable because:

- The key can only submit feedback (write-only)
- The widget runs on staging sites only, limiting exposure
- Rate limiting is enforced per API key

## Consequences

### Positive

- Simple, appropriate security for the use case
- No OAuth infrastructure needed
- Widget integration is minimal (one HTTP header)

### Negative / Trade-offs

- API key rotation requires updating the staging site's environment
  variable and redeploying
- Two authentication code paths to maintain and test
- API key visible in browser source on staging sites

### Follow-up actions

- Implement custom Symfony security authenticator for `X-Api-Key`
- Add rate limiting middleware per API key
- Document API key generation and rotation in the project admin UI
