# User Stories — Tidy Feedback End-to-End Flow

## Context

These stories cover the full feedback flow across three roles:

- **Product Owner / Customer** — Tests features on the staging site and submits feedback via the tidy-feedback widget
- **Product Manager** — Receives, reviews, and triages feedback in the central backend dashboard
- **Developer** — Receives GitHub issues derived from feedback and implements fixes

The tidy-feedback module is installed **on the staging site only** (not production). It provides:

- A **floating widget** available on every page for submitting feedback
- A **local dashboard** at `/tidy-feedback` (basic auth) where testers and the PO can review their own submissions, verify screenshots, and confirm context data was captured correctly

When the PO submits feedback, it is forwarded to the central backend for the PM to triage. The local dashboard is the PO's primary interface — the central backend dashboard is the PM's domain. How feedback reaches the backend (widget push, backend pull, or both) is an **open architectural decision** (see section 2.1 of the backend spec).

---

## User Story 1 — Product Owner / Customer

**As a Product Owner testing a new feature on the staging site, I want to submit feedback directly from the page I am reviewing, so that my observations are captured in context without leaving the page.**

### Scenario

The PO is reviewing a new feature on the staging site. Something doesn't work as expected. They open the tidy-feedback widget (floating button on the page), fill in a subject and description, and submit.

### Acceptance Criteria

- The tidy-feedback widget is available on the staging site without requiring the PO to log into any separate system
- The PO can fill in: subject (required), description, email, and "what did you do"
- A screenshot is automatically captured; the PO can optionally crop or replace it
- On submit, feedback is stored locally on the staging site and forwarded to the central backend
- A toast message confirms "Feedback submitted" and includes a direct link to the item in the **local dashboard** (`/tidy-feedback/{id}`) — accessible on the staging site without a central backend account
- The widget closes or resets after successful submission
- If the central backend is unreachable, feedback is still saved locally and the PO sees the submission confirmed — the local record is never lost due to a backend outage
- The PO can review all their submitted items in the local dashboard at `/tidy-feedback` on the staging site, including screenshots and context data
- The local dashboard does not require knowledge of the central backend — it works as a standalone review tool for testers

### Gaps / Open Questions

- **How feedback reaches the backend is undecided.** The push (widget POST), pull (backend polls), or dual (both simultaneously with graceful degradation) approach each have different implications for what the PO experiences when the backend is unavailable. The dual approach is the most resilient — feedback is always saved locally, and forwarding to the backend is best-effort.
- **The PO has no visibility into triage progress.** Once feedback leaves the staging site, there is no way to see whether it was triaged, exported to GitHub, or resolved (section 8.5 — public status page — is a future feature). This is an accepted v1 limitation but should be communicated clearly to testers.

---

## User Story 2 — Product Manager

**As a Product Manager, I want to receive, review, and triage incoming feedback in a central dashboard, so that I can classify issues and route them to the right developer via GitHub.**

### Scenario

The PM receives an email notification that new feedback has arrived from the staging site. They open the central backend dashboard, review the full item including screenshot and context, add an internal note, set the status to `in_progress`, and export it to GitHub as an issue.

> **Note:** The central backend receives feedback either via a push POST from the widget or by pulling from the staging site — this architectural decision is still open. From the PM's perspective, the experience is identical either way: feedback appears in the dashboard with a small delay at most.

### Acceptance Criteria

- PM receives a notification email shortly after the PO submits (sent asynchronously via Symfony Messenger)
- The email contains: subject, description, reporter email, page URL, and a direct link to the item in the dashboard
- On the feedback detail page, the PM can see: screenshot inline, all context data (browser, viewport, page URL), submission timestamp, and reporter info
- PM can assign a **category** (`bug`, `missing_feature`, `improvement`, `other`) — this classification is dashboard-only and is not collected by the widget
- PM can update **status** (`new` → `in_progress`) without a full page reload
- PM can add an internal note (visible to all dashboard users) via the notes section, which updates without reloading the full page
- The feedback list page supports filtering by project, status, category, and date range — the PM can find all `new` items across projects with a single filter combination
- PM can click "Export to issue tracker" on the detail page and select which GitHub tracker config to use when the project has multiple configured
- After export, a `FeedbackExport` record is created and the detail page shows: tracker name, issue reference (e.g. `itk-dev/my-site#42`), export timestamp, and a clickable link to the GitHub issue
- The feedback item is visibly marked as exported — both on the detail page (export history section) and in the feedback list (exported indicator)
- Workflow status and export state are independent — exporting does not automatically change the status; the PM controls both separately

### Gaps / Open Questions

- **No bulk triage in v1.** If the PO submits many items during a test session, the PM must open each one individually to categorise and export. Bulk export is deferred (section 3.9.3). This will be felt immediately in practice during active testing periods.
- **Notification throttling is a future feature (section 8.10).** A heavy test session generates one email per item. The PM may want to configure notification emails carefully or expect noise during active testing.
- **Issue template preview is not available in v1.** When the PM configures a custom issue body template on the project edit page, there is no way to preview the rendered output before the first real export.

---

## User Story 3 — Developer

**As a Developer, I want to receive clearly described GitHub issues derived from real user feedback, so that I can understand, reproduce, and fix the problem with full context.**

### Scenario

The PM has exported a feedback item to GitHub. The developer picks up the issue in their normal GitHub workflow, works on a fix, and closes the issue. The PM then manually updates the status in the central backend dashboard to `resolved`.

### Acceptance Criteria

- The GitHub issue title matches the feedback subject
- The issue body (rendered from the configured template) includes: description, reporter info, page URL, category, browser and viewport info, and a link back to the screenshot in the central backend dashboard
- Default labels (e.g. `feedback`, `triage`) are applied automatically at creation time
- The developer can follow the link in the issue body to view the full feedback item in the dashboard (including the full-resolution screenshot and context data)
- The issue is created in the correct repository as configured in the project's `ExternalTrackerConfig`
- The central backend detail page shows the GitHub issue reference and link so the PM can monitor issue progress by following the link

### Gaps / Open Questions

- **No automatic status sync in v1.** When the developer closes the GitHub issue, nothing changes in the central backend automatically (section 8.3 — bi-directional sync — is a future feature). The PM must manually set the feedback item to `resolved` or `closed`. The team needs a shared convention to ensure this doesn't get forgotten.
- **Screenshot access requires a dashboard login.** The link in the GitHub issue points to the central backend dashboard, which is behind authentication. If the developer does not have a dashboard account, they cannot view the screenshot. The flat user model in v1 (all users see all projects) makes this easy to solve — developers should be given accounts — but this needs to be an explicit decision.
- **The developer has no way to push comments or status back to the feedback item.** Internal notes in the dashboard are PM-only unless the developer also has a dashboard account and knows to use it alongside GitHub.

---

## Cross-cutting Gaps

These span all three stories and warrant a team discussion before implementation begins:

1. **Data flow architecture is undecided.** Push (widget POST), pull (backend polls staging site), or dual (both, with graceful degradation) each affect resilience, real-time behaviour, and the scope of widget changes. This is the highest-priority open decision — it determines what gets built in Phase 3 of the backend spec. The dual approach is recommended: always write locally, forward to the backend as best-effort.

2. **Dashboard access model** — The spec has one flat user role. It is worth deciding explicitly: do POs get accounts? Do developers? The PO's primary interface is the local dashboard, so they likely don't need central backend access. Developers need access if they are expected to view screenshots linked from GitHub issues.

3. **The resolution loop has one manual gap** — After a developer closes a GitHub issue, nothing changes in the central backend automatically (section 8.3 — bi-directional sync — is a future feature). The PM must manually set the feedback item to `resolved` or `closed`. The team needs an agreed convention to prevent status drift.

4. **Staging-only installation must be enforced** — The widget is intended for staging, not production. Accidental production installation would expose the widget to real end users and generate noise in the central backend. This should be enforced by environment variable gating (e.g. only active when `APP_ENV=staging`) and documented clearly.
