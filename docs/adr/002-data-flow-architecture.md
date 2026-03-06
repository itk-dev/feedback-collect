# ADR-002: Data Flow Architecture — Dual Posting

- **Status:** Accepted
- **Created by:** Jesper Pedersen
- **Date:** 2026-03-04
- **Stakeholders:** Product Owner, Product Manager, development team

## Context and Drivers

The tidy-feedback widget runs on staging sites and stores feedback in a
local SQLite database. The new central backend needs to receive this
feedback for PM triage and GitHub export. Three approaches were
considered for how feedback reaches the backend.

The data flow decision affects:

- Whether widget code changes are needed (Phase 3 scope)
- Resilience when the central backend is unavailable
- How quickly the PM sees new feedback
- Whether the PO's local dashboard always has a complete record

### Functional drivers

- PM needs to see feedback promptly after submission
- PO's local dashboard must always show all submitted feedback
- Feedback must not be lost if the central backend is down

### Non-functional drivers

- Minimize widget code changes for existing installations
- Avoid polling overhead on staging sites
- Graceful degradation during backend outages

## Options Considered

### Option A: Push only

Widget POSTs directly to the central backend instead of the local
endpoint.

**Pros:**

- Real-time delivery
- Simple: one destination per submission

**Cons:**

- Feedback is lost if the backend is unreachable
- PO's local dashboard loses the submission entirely
- Requires widget JS changes (replace `form.action` target)

### Option B: Pull only

Central backend polls each staging site's existing GET endpoints
periodically.

**Pros:**

- Zero widget code changes
- Local storage is always the source of truth

**Cons:**

- Delayed delivery (polling interval)
- Bandwidth-heavy with many sites
- Requires the backend to know about and authenticate with each
  staging site's local dashboard
- Screenshot transfer over polling is expensive

### Option C: Dual posting (chosen)

Widget POSTs to the local endpoint first (guaranteed local storage),
then the backend (server-side) forwards to the central backend as
best-effort.

**Pros:**

- Local storage is always guaranteed
- Near-real-time delivery to the central backend
- PO's local dashboard always has the complete record
- Graceful degradation: if the backend is down, local copy is safe
- Forwarding can be retried asynchronously

**Cons:**

- Requires widget or server-side code changes for the forwarding step
- Slightly more complex than push-only
- Need to handle deduplication if retries succeed after initial failure

## Decision

Use **dual posting** (Option C).

The widget submits to the local endpoint as it does today. The local
tidy-feedback module then forwards the submission to the central
backend as a best-effort background operation. If forwarding fails, the
local copy is preserved and forwarding can be retried.

This approach ensures the PO never loses feedback due to backend
downtime, while still providing near-real-time delivery to the PM's
dashboard.

## Consequences

### Positive

- Zero data loss risk during backend outages
- PO's local dashboard is always complete and immediately available
- Existing installations continue working without configuration changes
  (forwarding only activates when `TIDY_FEEDBACK_REMOTE_URL` is set)

### Negative / Trade-offs

- Phase 3 (frontend widget changes) must implement the forwarding logic
  in the tidy-feedback module's server-side code
- Need deduplication logic in the central backend (idempotency key or
  source site + local ID uniqueness constraint)
- Slightly higher latency than direct push (local write + forward)

### Follow-up actions

- Design the forwarding mechanism in the tidy-feedback module
  (synchronous HTTP call vs. queued message)
- Add a `forwardedAt` or `remoteId` field to the local `Item` entity
  to track forwarding status
- Implement idempotency handling in the central backend's
  `POST /api/v1/feedback` endpoint
