# Tidy Feedback Backend — Feature Specification & Implementation Plan

## 1. Problem Statement

The existing `itk-dev/tidy-feedback` package is a combined Drupal module / Symfony bundle that collects user feedback. It is installed on **staging sites only** (not production). It provides a floating feedback widget and a local dashboard at `/tidy-feedback` where testers and the Product Owner can review their own submissions, verify screenshots, and confirm context data was recorded correctly. Feedback is stored in a local SQLite database on the staging site itself.

There is no multi-site support, no central dashboard for project managers, and no way to aggregate or triage feedback across multiple sites or projects.

**Goal:** Build a dedicated Symfony backend application that acts as a centralized feedback collector for multiple staging sites/projects, each running the tidy-feedback widget. Provide a management dashboard for the Product Manager to view, filter, triage, and export feedback across all connected sites — with the ability to forward feedback as issues to GitHub or other project management tools.

The local tidy-feedback dashboard on each staging site remains the **testers' and PO's primary interface**. The central backend is the **PM's interface** for triage and routing.

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│  Staging environments (one per project)                   │
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  Site A       │  │  Site B       │  │  Site C       │  │
│  │  (Drupal)     │  │  (Symfony)    │  │  (Any CMS)    │  │
│  │  tidy-feedback│  │  tidy-feedback│  │  tidy-feedback│  │
│  │  widget +     │  │  widget +     │  │  widget +     │  │
│  │  local SQLite │  │  local SQLite │  │  local SQLite │  │
│  │  dashboard    │  │  dashboard    │  │  dashboard    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
│  ▲ PO/tester interface    │                 │            │
└──┼──────────────────────  │  ───────────────┼────────────┘
   │                        │                 │
   │  feedback forwarded to central backend   │
   │  (push POST, pull, or both — TBD)        │
   └────────────┬───────────┘─────────────────┘
                ▼
    ┌───────────────────────┐
    │  Tidy Feedback Backend │
    │  (Symfony application) │
    │                       │
    │  • REST API ingestion │
    │  • Dashboard (Twig +  │
    │    Stimulus/Turbo)    │
    │  • Site/project mgmt  │
    │  • User auth (login)  │
    │  • Notifications      │
    │  • Export to GitHub /  │
    │    issue trackers     │
    └───────────────────────┘
    ▲ PM interface
```

The system consists of two distinct layers:

**Staging site layer** (`itk-dev/tidy-feedback` module/bundle) — Installed on each staging site. Provides the feedback widget for testers and POs, stores submissions in a local SQLite database, and exposes a local dashboard at `/tidy-feedback` for reviewing submissions. This is the primary interface for the people submitting feedback.

**Central backend application** (new Symfony app) — Aggregates feedback from all staging sites, stores it in PostgreSQL, and provides a dashboard for the Product Manager to triage, categorise, and export items to GitHub. This is the PM's domain — not the tester's.

How feedback moves from the staging layer to the central backend is an open architectural decision (see section 2.1).

### 2.1 Open Decision: Data Flow Architecture (Push vs Pull)

> **Status: To be decided.** The architecture diagram above shows a push (POST) model. An alternative pull (fetch/crawl) model has been proposed. This section documents the trade-offs to support the decision.

#### Option A: Push (POST) — widget sends feedback directly to central backend

The spec currently describes this approach. The widget is modified to POST feedback to a remote URL with an API key header. This requires changes to `itk-dev/tidy-feedback` (see section 4).

#### Option B: Pull (Fetch/Crawl) — central backend polls each site

The existing `itk-dev/tidy-feedback` already exposes GET endpoints:
- `GET /tidy-feedback` — returns all items as JSON (authenticated via basic HTTP auth)
- `GET /tidy-feedback/{id}` — single item with full data
- `GET /tidy-feedback/{id}/image` — screenshot as binary

The central backend could periodically call these endpoints on each registered site to collect feedback. This would require **zero widget changes** — section 4 of this spec becomes unnecessary.

#### Option C: Dual — widget posts locally AND to central backend (graceful degradation)

A hybrid of Options A and B. The widget always POSTs to the local endpoint first (guaranteed local storage), and also forwards the submission to the central backend if configured. If the central backend is unreachable, the widget still confirms success to the tester based on the local write — the central backend sync is best-effort. This requires the same widget changes as Option A but eliminates the "feedback lost on backend downtime" risk.

The local tidy-feedback dashboard continues to work as the tester's primary review tool regardless of backend availability. Historical items can be imported into the central backend via the pull-based migration tool (Phase 6) if needed.

#### Trade-off comparison

| Concern | Push (POST to central) | Pull (central polls sites) | Dual (local + central, best-effort) |
|---|---|---|---|
| **Real-time** | Immediate — feedback visible in dashboard within seconds | Delayed by poll interval (e.g. 5 minutes) | Immediate for central backend; local write always immediate |
| **Widget changes** | Required — ~20 lines of JS + env vars per site | None — widget works as-is | Same as push — ~20 lines of JS + env vars |
| **Bandwidth** | Efficient — each item sent once | Wasteful — full item list (incl. base64 screenshots) fetched every poll cycle. No `since` parameter exists on the GET endpoint. | Same as push — one POST per item to central backend |
| **Duplicate handling** | Not needed — each POST creates one item | Required — sites use auto-increment integer IDs (not globally unique). Need composite key (site + remote ID) and "last seen" tracking. | Not needed for new submissions; needed for migration of historical items only |
| **Notifications** | Immediate email on receipt | Delayed by poll interval | Immediate on receipt |
| **Network topology** | Central backend must be publicly reachable from user browsers | Central backend must be able to reach each site's admin endpoint (may be blocked by firewalls/VPNs) | Same as push — backend must be publicly reachable |
| **Central backend downtime** | Feedback lost unless local fallback added to widget | No impact — sites store locally, backend catches up when restored | **No impact** — local write always succeeds; central backend sync retried or imported later |
| **CORS** | Required — must configure allowed origins | Not needed — server-to-server calls | Required — same as push |
| **Scalability** | 1 HTTP request per feedback submission | N sites × polls/hour, each transferring full item lists including screenshots | 1 local write + 1 remote POST per submission |
| **Security model** | Hashed API key per project (visible in page source, write-only) | Basic auth credentials for each site stored in central backend | Same as push |
| **Existing API changes** | None to existing GET endpoints | Would benefit from adding `since` parameter and pagination to GET endpoint — which means modifying tidy-feedback anyway | None to existing GET endpoints |
| **Tester experience on backend downtime** | Error shown, or feedback silently lost | No visible impact | Success shown (local write succeeded); backend sync is invisible |

#### Key observations

1. **The existing GET endpoint returns ALL items every poll** — there is no filtering by date or ID. As sites accumulate feedback, each poll transfers increasingly large payloads including base64 screenshot data for every item. Adding a `since` parameter requires modifying tidy-feedback, which eliminates pull's main advantage (zero widget changes).

2. **Pull cannot deliver real-time notifications** — email notifications on new feedback (section 3.10) would be delayed by the poll interval. Push delivers them immediately.

3. **The widget changes for push are small** — approximately 20 lines of JavaScript and a few environment variables per site. The effort is days, not weeks.

4. **Push is the industry standard** for this class of system (Sentry, Hotjar, UserSnap, BugHerd all use POST/push).

5. **The local dashboard is the tester's primary interface** — testers and the PO review submissions locally at `/tidy-feedback` on the staging site, not in the central backend. This means the central backend is not in the critical path for the tester experience. If the central backend is unreachable, the tester's workflow is unaffected as long as the local write succeeds.

6. **Dual (Option C) gives the best tester experience** — the local write always succeeds and confirms the submission immediately. The central backend sync is invisible to the tester, making backend downtime a PM concern rather than a tester-facing failure.

#### Recommended approach

Use **dual posting (Option C) as the primary data flow**: the widget always POSTs to the local endpoint first, then forwards to the central backend as a best-effort fire-and-forget. The tester sees success based on the local write. The central backend receives real-time submissions when available.

Combine this with a **one-time pull-based migration tool** (Phase 6, step 22) using the existing GET endpoints to import any historical items that were collected before the central backend was set up.

> **Action needed:** Discuss with the development team and confirm the approach before implementation begins. If pure pull is chosen, section 4 (Frontend Widget Changes) should be removed and replaced with a polling/sync service specification.

---

## 3. Feature Specification — Backend Application

### 3.1 Data Model

#### Entity: `Project`

Represents a configured site/project that sends feedback to the backend.

| Field | Type | Description |
|-------|------|-------------|
| `id` | `uuid` | Primary key (UUIDv7) |
| `name` | `string(255)` | Human-readable project name |
| `slug` | `string(128)` | URL-safe identifier, unique |
| `description` | `text, nullable` | Optional description |
| `url` | `string(512)` | Base URL of the site |
| `apiKey` | `string(64)` | Hashed API key for authentication |
| `apiKeyPrefix` | `string(8)` | First 8 chars of key (for identification) |
| `isActive` | `boolean` | Whether the project accepts feedback |
| `captureHtmlDocument` | `boolean` | Whether the widget should send full HTML (default: false) |
| `notificationEmails` | `json` | Email addresses to notify on new feedback |
| `createdAt` | `datetime_immutable` | Creation timestamp |
| `updatedAt` | `datetime_immutable` | Last modification timestamp |

#### Entity: `FeedbackItem`

Stores individual feedback submissions. Closely mirrors the existing `Item` model but adds project association and richer metadata.

| Field | Type | Description |
|-------|------|-------------|
| `id` | `uuid` | Primary key (UUIDv7) |
| `project` | `ManyToOne → Project` | Which project this belongs to |
| `subject` | `string(255)` | Feedback subject (required) |
| `description` | `text, nullable` | User-provided description |
| `createdBy` | `string(255), nullable` | Email or name of reporter |
| `status` | `string(32)` | Enum: `new`, `in_progress`, `resolved`, `closed` |
| `category` | `string(64), nullable` | Enum: `bug`, `missing_feature`, `improvement`, `other`. Note: the current widget does not collect category — this is a **dashboard-only classification** assigned by triaging users, not submitted by the widget. |
| `pageUrl` | `string(2048), nullable` | URL of the page where feedback was given |
| `contextData` | `json` | Browser info, window size, region, etc. |
| `screenshotPath` | `string(512), nullable` | Path to stored screenshot file |
| `htmlDocumentPath` | `string(512), nullable` | Path to stored compressed HTML document (opt-in) |
| `rawData` | `json` | Full original POST payload for future-proofing |
| `exports` | `OneToMany → FeedbackExport` | Export records — see `FeedbackExport` entity below |
| `createdAt` | `datetime_immutable` | Submission timestamp |
| `updatedAt` | `datetime_immutable` | Last status change |

#### Entity: `FeedbackNote`

Internal notes/comments added by dashboard users to feedback items.

| Field | Type | Description |
|-------|------|-------------|
| `id` | `uuid` | Primary key (UUIDv7) |
| `feedbackItem` | `ManyToOne → FeedbackItem` | Parent feedback item |
| `author` | `ManyToOne → User` | User who wrote the note |
| `content` | `text` | Note content |
| `createdAt` | `datetime_immutable` | When the note was written |

#### Entity: `FeedbackExport`

Tracks individual exports of feedback items to external issue trackers. A feedback item can be exported to multiple trackers (e.g. GitHub for triage, then Leantime for planning). This replaces the previous design where `externalIssueUrl`/`externalIssueId` were single fields on `FeedbackItem` — that approach treated export as a terminal status (`exported`), but in practice an item can be "resolved" and also exported, or exported to multiple trackers.

| Field | Type | Description |
|-------|------|-------------|
| `id` | `uuid` | Primary key (UUIDv7) |
| `feedbackItem` | `ManyToOne → FeedbackItem` | The exported feedback item |
| `trackerConfig` | `ManyToOne → ExternalTrackerConfig` | Which tracker config was used |
| `externalIssueUrl` | `string(2048)` | URL of the created issue |
| `externalIssueId` | `string(255)` | External issue ID/reference (e.g. `owner/repo#42`) |
| `exportedBy` | `ManyToOne → User` | User who performed the export |
| `createdAt` | `datetime_immutable` | When the export was performed |

This design cleanly separates **workflow status** (`new` → `in_progress` → `resolved` → `closed`) from **export tracking**. A feedback item's status reflects its triage/resolution state, while exports are tracked independently.

#### Entity: `User`

Admin users who can access the dashboard. All users can see all projects (simple model to start; project-level scoping can be added later if needed).

| Field | Type | Description |
|-------|------|-------------|
| `id` | `uuid` | Primary key |
| `email` | `string(180)` | Login identifier, unique |
| `password` | `string` | Hashed password |
| `roles` | `json` | Symfony roles array |
| `name` | `string(255)` | Display name |
| `createdAt` | `datetime_immutable` | Account creation |

#### Entity: `ExternalTrackerConfig`

Configures a connection to an external issue tracker (GitHub, Jira, Leantime, etc.) for a project. A project can have **multiple** tracker configurations — e.g. one GitHub repo for bug reports and a Leantime project for feature requests. When exporting, the user chooses which tracker to send each item to.

| Field | Type | Description |
|-------|------|-------------|
| `id` | `uuid` | Primary key (UUIDv7) |
| `project` | `ManyToOne → Project` | Which project this belongs to |
| `name` | `string(255)` | Human-readable label (e.g. "GitHub – itk-dev/my-site", "Leantime – Redesign") |
| `type` | `string(32)` | Enum: `github`, `gitlab`, `jira`, `leantime` (extensible) |
| `baseUrl` | `string(512)` | API base URL (e.g. `https://api.github.com`) |
| `repository` | `string(255)` | Target repository or project identifier (e.g. `itk-dev/my-site`) |
| `authToken` | `string` | Encrypted API token/PAT for the tracker |
| `defaultLabels` | `json` | Labels to apply to created issues (e.g. `["feedback", "triage"]`) |
| `issueTemplate` | `text, nullable` | Custom issue body template (Twig-like placeholders) |
| `isActive` | `boolean` | Whether export is enabled |
| `createdAt` | `datetime_immutable` | Creation timestamp |
| `updatedAt` | `datetime_immutable` | Last modification timestamp |

### 3.2 API Endpoints (Feedback Ingestion)

All API endpoints are prefixed with `/api/v1`.

#### `POST /api/v1/feedback`

Receives feedback from the widget. Authenticated via API key in the `X-Api-Key` header.

**Request headers:**
- `Content-Type: application/json`
- `X-Api-Key: <project-api-key>`

**Request body** (matches current widget output):
```json
{
  "subject": "Button not working",
  "created_by": "user@example.com",
  "description": "The submit button does not respond.",
  "what_did_you_do": "Clicked the button on the form.",
  "image": "data:image/webp;base64,...",
  "context": {
    "url": "https://example.com/page",
    "referrer": "https://example.com/",
    "navigator": { "userAgent": "..." },
    "window": { "innerWidth": 1440, "innerHeight": 900 },
    "region": { "left": "300px", "top": "200px", "width": "400px", "height": "300px" }
  }
}
```

The `context.document` field (full HTML) is only included when the project has `captureHtmlDocument` enabled and the widget is configured accordingly.

**Response:** `201 Created` with JSON body containing the created item.

**Error responses:**
- `401 Unauthorized` — Missing or invalid API key
- `403 Forbidden` — Project is deactivated
- `400 Bad Request` — Validation errors
- `413 Payload Too Large` — Screenshot exceeds size limit

#### `GET /api/v1/feedback/{id}` (optional, future)

Retrieve a single feedback item. Authenticated via API key.

### 3.3 Screenshot & Document Handling

Screenshots are extracted from the `image` field (base64 data URI), decoded, and stored on the filesystem (configurable storage path, e.g. `var/screenshots/{project_slug}/{item_id}.webp`). The base64 data is **not** stored in the database to keep DB size manageable — only the file path is stored.

HTML document capture is **opt-in per project**. When enabled, the `document` HTML from the context payload is gzip-compressed and stored as a file alongside the screenshot. The project detail page in the dashboard exposes a toggle for this setting. The widget checks the project config and only sends the document when the project has it enabled (see section 4.4).

A configurable maximum payload size should be enforced (default: 10 MB).

### 3.4 Dashboard (Web Interface)

The dashboard is built with standard Symfony stack: Twig templates, Turbo Drive for SPA-like navigation, Turbo Frames for partial page updates, and Stimulus controllers for interactive behavior. No React, Vue, or other heavy JS frameworks.

#### Authentication

Standard Symfony Security with `form_login`. Users are managed via CLI commands initially, with optional admin UI later. All users can see all projects (simple access model; project-level scoping can be added later if needed).

#### Dashboard Pages

**3.4.1 — Overview / Home (`/dashboard`)**

> **Deferred for v1.** The initial home page redirects to the feedback list (`/dashboard/feedback`). A summary overview with statistics becomes meaningful after data accumulates and can be added in a later phase.

Future version would show:
- Total feedback count per project (with sparkline or simple bar)
- Recent feedback items across all projects (last 10)
- Quick-filter buttons for status (new, in_progress, resolved, closed)

**3.4.2 — Feedback List (`/dashboard/feedback`)**

The main working view. A filterable, sortable table of all feedback items with bulk selection support.

Filters (rendered as a Stimulus-powered filter bar that updates URL params):
- **Project** — dropdown/multi-select
- **Status** — checkbox group (new, in_progress, resolved, closed)
- **Exported** — filter by export state (exported / not exported)
- **Category** — checkbox group
- **Date range** — from/to date pickers
- **Search** — free text search on subject, description, created_by

Sorting: Clickable column headers for subject, created_at, status, project.

Pagination: Standard Symfony paginator with Turbo Frame so only the table body reloads on page change.

Bulk actions (applied to selected items via checkboxes):
- **Change status** — Set status on multiple items at once

> **Deferred:** Bulk export to issue tracker is deferred to a later phase. Single-item export from the feedback detail page (section 3.9.2) covers the initial need. Bulk export adds significant UI complexity (cross-project tracker selection, progress tracking, partial failure handling) that is not justified until the single-export workflow is proven.

Implementation approach: Use a `LiveComponent` (Symfony UX) for the filter + table combination. The `LiveProp(writable: true, url: true)` feature maps filter state to URL query parameters, making filters bookmarkable and shareable. As the user changes filters, the component re-renders server-side and Turbo morphs the DOM.

**3.4.3 — Feedback Detail (`/dashboard/feedback/{id}`)**

Shows full details of a single feedback item:
- All metadata (subject, description, category, status, reporter, timestamps)
- Screenshot image (rendered inline, with link to full-size)
- Context data (collapsible JSON display)
- Page URL (clickable link)
- Status change controls (dropdown to update status, with Stimulus confirmation)
- Export history — list of `FeedbackExport` records with clickable links to created issues
- "Export to issue tracker" button with tracker selection (if the project has trackers configured)
- Internal notes section (add/view notes from dashboard users)

Status changes are handled via a small Stimulus controller that submits a PATCH request and updates the badge via Turbo Stream.

The notes section uses a Turbo Frame so adding a note doesn't reload the full page. A simple form (textarea + submit) appends to the notes list.

**3.4.4 — Project List (`/dashboard/projects`)**

Table of all configured projects with columns: name, URL, status (active/inactive), issue trackers (count of configured trackers), feedback count, last feedback received.

Each row links to the project detail/edit page. An "Add project" button opens the creation form.

**3.4.5 — Project Create/Edit (`/dashboard/projects/new`, `/dashboard/projects/{id}/edit`)**

Symfony Form for:
- Name, description, URL
- Active/inactive toggle
- HTML document capture toggle
- Notification email addresses (comma-separated or tag-style input)
- External issue tracker configurations — an embedded collection allowing add/edit/remove of multiple tracker connections (see section 3.9)

On **create**, the system generates a new API key and displays it **once** in a dismissable alert (the key is hashed and cannot be retrieved later). A "Regenerate API key" button is available on the edit page.

A Stimulus controller handles the "copy to clipboard" button for the API key.

**3.4.6 — Project Detail (`/dashboard/projects/{id}`)**

Shows project info, configuration snippet for the widget, and a filtered feedback list (same `LiveComponent` as 3.4.2 but pre-filtered to this project).

The configuration snippet shows example environment variables / config for setting up the widget on that project's site:

```
TIDY_FEEDBACK_REMOTE_URL=https://feedback.example.com/api/v1/feedback
TIDY_FEEDBACK_API_KEY=tf_xxxxxxxxxxxxxxxx
TIDY_FEEDBACK_CAPTURE_DOCUMENT=false
```

### 3.5 Technology Stack

| Layer | Technology |
|-------|-----------|
| Framework | Symfony 7.x |
| Database | PostgreSQL (via Doctrine ORM) |
| Templating | Twig |
| JS interactivity | Stimulus (via `symfony/stimulus-bundle`) |
| SPA navigation | Turbo Drive + Turbo Frames (`symfony/ux-turbo`) |
| Dynamic components | Symfony UX Live Components (for filter/table) |
| CSS | Bootstrap 5 (matches existing tidy-feedback styling) |
| Asset pipeline | AssetMapper (recommended over Webpack Encore for simplicity) |
| Authentication | Symfony Security (`form_login`) |
| API auth | Custom API key authenticator (Symfony Security) |
| Screenshot storage | Local filesystem (Flysystem abstraction for future S3 support) |
| Email notifications | Symfony Mailer |
| External trackers | GitHub API via `knplabs/github-api` (v1), adapter pattern for future trackers |
| Testing | PHPUnit + Symfony Panther for E2E |
| Dev tooling | Docker Compose, Taskfile |

### 3.6 Configuration

The backend application is configured via environment variables:

```env
DATABASE_URL=postgresql://user:pass@localhost:5432/tidy_feedback
APP_SECRET=your-app-secret
SCREENSHOT_STORAGE_PATH=%kernel.project_dir%/var/screenshots
MAX_PAYLOAD_SIZE=10485760
CORS_ALLOWED_ORIGINS=https://site-a.example.com,https://site-b.example.com

# Email notifications
MAILER_DSN=smtp://localhost
NOTIFICATION_FROM_ADDRESS=feedback@example.com
```

CORS must be configured to accept requests from all registered project URLs. This can be dynamically built from the `Project` entity or configured as a comma-separated list.

### 3.7 Security Considerations

- API keys are hashed (SHA-256 with prefix) before storage; only the prefix is stored in cleartext for identification
- CORS is restricted to configured project origins
- Rate limiting on the API endpoint (Symfony RateLimiter component)
- CSRF protection on all dashboard forms
- Screenshot uploads are validated (allowed MIME types, max size)
- The `document` HTML in context is stored compressed but never rendered raw in the dashboard (XSS prevention)
- External tracker tokens are encrypted at rest (Symfony Secrets or `paragonie/halite`)

### 3.8 Data Integrity & Operational Concerns

#### 3.8.1 Database Indexing Strategy

The feedback list page supports filtering by project, status, date range, and free-text search. The following indexes are required on `FeedbackItem`:

- `project_id` — filter by project
- `status` — filter by workflow status
- `createdAt` — sort by date, date range filter
- `project_id, status, createdAt` — composite index for the most common query pattern (filtered list sorted by date)
- GIN index on `subject` and `description` — PostgreSQL full-text search for the search filter

#### 3.8.2 Data Retention & Cleanup

- **Screenshot cleanup:** A scheduled job (Symfony Scheduler or cron) should remove screenshot and HTML document files for feedback items older than a configurable retention period (default: 12 months). The `FeedbackItem` record is preserved but the file paths are set to null.
- **Project archival:** When a project is deactivated (`isActive = false`), it stops accepting new feedback but existing items remain accessible. There is no hard delete of projects — use deactivation instead.
- **Cascading behavior:** If a project must be deleted (rare), all associated `FeedbackItem`, `FeedbackExport`, `FeedbackNote`, and `ExternalTrackerConfig` records are cascade-deleted.

#### 3.8.3 Error Resilience

- **Screenshot storage failure:** If screenshot extraction or file storage fails (disk full, permission error, corrupt base64), the API should still accept and store the feedback item. The `screenshotPath` field is set to null and the failure is logged. The user's feedback should never be lost due to a screenshot processing error.
- **HTML document storage failure:** Same approach — store the feedback item, set `htmlDocumentPath` to null, log the error.

#### 3.8.4 Timezone Handling

All timestamps are stored in UTC in the database (matching the existing `itk-dev/tidy-feedback` behavior). The dashboard displays timestamps in the user's local timezone using browser-based formatting (e.g. `Intl.DateTimeFormat` via a small Stimulus controller or Twig's `timezone` filter with a user preference).

### 3.9 External Issue Tracker Integration

This feature allows dashboard users to export feedback items as issues in external trackers. **For v1, only the GitHub adapter will be implemented.** GitLab, Jira, and Leantime adapters are defined in the interface but deferred to later phases.

#### 3.9.1 Configuration

Each project can have **one or more** `ExternalTrackerConfig` entries (see data model). These are managed on the project edit page, where the user can add, edit, and remove tracker connections. Each tracker has its own type, repository target, API token, default labels, and an optional issue body template.

The issue body template uses Twig-like placeholders so the team can customize what an exported issue looks like:

```
**Feedback from {{ project.name }}**

**Subject:** {{ item.subject }}
**Reported by:** {{ item.createdBy ?? 'Anonymous' }}
**Page:** {{ item.pageUrl }}
**Category:** {{ item.category }}

**Description:**
{{ item.description }}

**What did they do:**
{{ item.rawData.what_did_you_do ?? 'Not provided' }}

**Browser:** {{ item.contextData.navigator.userAgent }}
**Viewport:** {{ item.contextData.window.innerWidth }}x{{ item.contextData.window.innerHeight }}

---
*Exported from Tidy Feedback*
```

A default template is used when no custom template is configured.

#### 3.9.2 Single Export (from Feedback Detail)

On the feedback detail page, an "Export to issue tracker" button appears when the project has at least one active `ExternalTrackerConfig`.

If the project has **multiple trackers configured**, clicking the button shows a dropdown or modal letting the user choose which tracker to export to (e.g. "GitHub – itk-dev/my-site" or "Leantime – Redesign project"). If only one tracker is configured, it exports directly.

After export, a `FeedbackExport` record is created linking the feedback item to the tracker and storing the issue URL and reference. A link to the created issue is displayed in the export history section on the detail page.

An item can be exported to **multiple trackers** (e.g. first to GitHub for triage, then to Leantime for planning). Each export creates a separate `FeedbackExport` record. The item's workflow status (`new`, `in_progress`, `resolved`, `closed`) is independent of its export state.

#### 3.9.3 Bulk Export (from Feedback List) — Deferred

> **Deferred to a later phase.** Single-item export from the feedback detail page covers the initial need. Bulk export is specified here for future reference.

When implemented, bulk export would:

1. Show a confirmation dialog where the user **selects which tracker** to export to (from the trackers configured on the relevant projects). If selected items span multiple projects, only trackers from projects that have them configured are shown — items without a matching tracker are flagged.
2. Create one issue per feedback item (not a single combined issue) — each gets its own issue in the tracker
3. Process exports sequentially with progress feedback (Turbo Stream updates or a simple progress indicator via Stimulus)
4. Create a `FeedbackExport` record for each successful export
5. Show a summary when done: how many succeeded, any failures (e.g. rate limit, auth error)

#### 3.9.4 Implementation: Adapter Pattern

The integration uses a service interface (`ExternalTrackerInterface`) with concrete implementations for each tracker type:

```php
interface ExternalTrackerInterface
{
    public function supports(string $type): bool;
    public function createIssue(ExternalTrackerConfig $config, FeedbackItem $item): ExternalIssueResult;
    public function testConnection(ExternalTrackerConfig $config): bool;
}
```

`ExternalIssueResult` is a simple value object containing `issueUrl`, `issueId`, and `success`.

Concrete adapters:
- `GitHubTrackerAdapter` **(v1)** — Uses `knplabs/github-api` or direct HTTP client to call GitHub REST API v3. Creates issue with title from subject, body from template, and applies default labels.
- `LeantimeTrackerAdapter` *(future)* — Uses Leantime REST API to create tickets. Prioritized after GitHub given team usage.
- `GitLabTrackerAdapter` *(future)* — Uses GitLab REST API. Similar to GitHub.
- `JiraTrackerAdapter` *(future)* — Uses Jira REST API. Maps feedback to Jira issue fields.

#### 3.9.5 Screenshot Attachment

When exporting to GitHub, the screenshot can optionally be uploaded as an image in the issue body (GitHub supports inline images via their upload API or by embedding base64 in the markdown — though the latter has size limits). A pragmatic approach: include a link back to the screenshot in the feedback backend dashboard rather than uploading the image, to avoid complexity with different tracker upload APIs.

### 3.10 Notifications

When new feedback is submitted, the backend sends email notifications to the addresses configured on the project (`notificationEmails` field). Notifications are sent asynchronously via Symfony Messenger to avoid slowing down the API response.

Each notification email contains:
- Feedback subject and description
- Reporter email (if provided)
- Page URL
- Direct link to the feedback item in the dashboard

A future enhancement could add webhook support or Slack integration, but email covers the initial need.

---

## 4. Feature Specification — Frontend Widget Changes

The existing `itk-dev/tidy-feedback` widget needs modifications to support remote feedback submission.

### 4.1 Configuration Changes

Add new environment variables / bundle configuration:

```env
# When set, feedback is POSTed to this URL instead of the local endpoint
TIDY_FEEDBACK_REMOTE_URL=https://feedback.example.com/api/v1/feedback

# API key for authenticating with the remote backend
TIDY_FEEDBACK_API_KEY=tf_xxxxxxxxxxxxxxxx

# Whether to include full HTML document in the payload (opt-in, default: false)
TIDY_FEEDBACK_CAPTURE_DOCUMENT=false
```

In Symfony bundle configuration:
```yaml
# config/packages/tidy_feedback.yaml
tidy_feedback:
    remote_url: '%env(TIDY_FEEDBACK_REMOTE_URL)%'
    api_key: '%env(TIDY_FEEDBACK_API_KEY)%'
    capture_document: '%env(bool:TIDY_FEEDBACK_CAPTURE_DOCUMENT)%'
```

In Drupal, these would be set via environment variables or `settings.local.php` (matching the existing pattern).

### 4.2 Widget Template Changes

The `widget.html.twig` template currently hardcodes the form action to `{{ path('tidy_feedback_create') }}`. This must be changed to use the remote URL when configured:

```twig
{% set form_action = remote_url ?? path('tidy_feedback_create') %}
{% set api_key = api_key ?? null %}

<form ... action="{{ form_action }}" method="post">
```

The `api_key`, `remote_url`, and `captureDocument` values need to be passed to the widget config JSON so the JavaScript can use them.

### 4.3 JavaScript Changes (`widget.js`)

The `fetch()` call in the form submit handler must be updated to:

1. Use the configured remote URL (from `config.remoteUrl`) if available, otherwise fall back to `form.action`
2. Include the API key header when posting to a remote backend
3. Handle CORS-related issues gracefully

Current code:
```javascript
fetch(form.action, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
})
```

Changed to:
```javascript
const url = config.remoteUrl || form.action;
const headers = { "Content-Type": "application/json" };
if (config.apiKey) {
    headers["X-Api-Key"] = config.apiKey;
}

fetch(url, {
    method: "POST",
    headers,
    body: JSON.stringify(data),
})
```

### 4.4 HTML Document Capture (Opt-in)

The current widget unconditionally sends `document.documentElement.outerHTML` in the context. This must become opt-in:

```javascript
data.context = {
    url: document.location.href,
    referrer: document.referrer,
    navigator: { userAgent: navigator.userAgent },
    window: { innerWidth: window.innerWidth, innerHeight: window.innerHeight },
    region: { /* ... */ },
};

// Only include full HTML when explicitly enabled
if (config.captureDocument) {
    data.context.document = document.documentElement.outerHTML;
}
```

This reduces payload size significantly for the majority of cases. When enabled, consider stripping `<script>` tags and inline styles to reduce size further.

### 4.5 Local Fallback

When `TIDY_FEEDBACK_REMOTE_URL` is not set, the widget should continue to work exactly as it does today — posting to the local endpoint and storing feedback in the local database. This ensures backwards compatibility.

### 4.6 API Key Security Note

The API key will be visible in the page source (embedded in the widget config). This is acceptable because:

- The key only grants write access to submit feedback for that specific project
- The key cannot be used to read existing feedback
- Rate limiting on the backend prevents abuse
- The alternative (server-side proxy) adds complexity without meaningful security gain for this use case

---

## 5. Implementation Plan

### Phase 1: Backend Foundation

**Goal:** Functioning API that can receive and store feedback.

1. **Project scaffolding**
   - Create new Symfony 7.x project
   - Configure Docker Compose (PHP 8.3, PostgreSQL, nginx)
   - Set up Taskfile for common commands
   - Configure AssetMapper + Stimulus + Turbo

2. **Data model & migrations**
   - Create `Project`, `FeedbackItem`, `FeedbackNote`, `FeedbackExport`, `User`, `ExternalTrackerConfig` entities with Doctrine ORM
   - Generate and run migrations
   - Add fixtures for development

3. **API key authentication**
   - Implement custom Symfony Security authenticator for `X-Api-Key` header
   - API key generation utility (CLI command)
   - Key hashing and prefix storage

4. **Feedback ingestion API**
   - `POST /api/v1/feedback` endpoint
   - Request validation (subject required, payload size limit)
   - Screenshot extraction and filesystem storage
   - HTML document storage (compressed, when present)
   - CORS configuration (nelmio/cors-bundle)
   - Rate limiting

5. **CLI commands**
   - `app:user:create` — Create dashboard user
   - `app:project:create` — Create project + generate API key
   - `app:project:list` — List projects

### Phase 2: Dashboard MVP

**Goal:** Working dashboard for viewing and managing feedback.

6. **Authentication & layout**
   - Login page with `form_login`
   - Base Twig layout with navigation sidebar
   - Bootstrap 5 integration via AssetMapper

7. **Project management pages**
   - Project list (Twig + Turbo Drive)
   - Project create/edit (Symfony Form + Stimulus for API key copy)
   - Project detail with configuration snippet
   - API key regeneration with confirmation dialog
   - HTML document capture toggle
   - Notification email configuration

8. **Feedback list page**
   - Live Component with filter controls (project, status, category, date range, search)
   - Sortable table columns
   - Pagination with Turbo Frames
   - URL-bound filter state (`LiveProp` with `url: true`)
   - Bulk selection via checkboxes
   - Bulk status change action

9. **Feedback detail page**
   - Full metadata display
   - Screenshot viewer (inline image)
   - Context data display (collapsible)
   - Status change control (Stimulus controller + PATCH endpoint)

### Phase 3: Frontend Widget Changes

**Goal:** Widget can post to remote backend.

10. **Configuration support**
    - Add `remote_url`, `api_key`, and `capture_document` to bundle/module configuration
    - Pass configuration through to Twig template and JavaScript config

11. **JavaScript changes**
    - Update `fetch()` call to support remote URL + API key header
    - Make HTML document capture opt-in
    - Error handling for CORS / network errors

12. **Testing & documentation**
    - Test local fallback (no remote URL configured)
    - Test remote posting (against backend API)
    - Update README with remote configuration instructions
    - Document API key setup flow

### Phase 4: Notifications & Notes

**Goal:** Proactive notification and internal collaboration on feedback.

13. **Email notifications**
    - Symfony Mailer + Messenger integration
    - Notification email template
    - Async sending via Messenger transport
    - Per-project notification email configuration

14. **Feedback notes**
    - Notes section on feedback detail page (Turbo Frame)
    - Add note form (textarea + submit)
    - Notes list with author and timestamp
    - Notes visible to all dashboard users

### Phase 5: External Issue Tracker Integration

**Goal:** Export feedback to GitHub as issues.

15. **Tracker configuration**
    - `ExternalTrackerConfig` entity and embedded collection form on project edit page
    - Add/edit/remove multiple tracker connections per project
    - Encrypted token storage
    - "Test connection" button per tracker (Stimulus controller that calls a backend endpoint)
    - Issue body template editor with placeholder reference

16. **GitHub adapter**
    - Implement `GitHubTrackerAdapter` (first and only v1 adapter)
    - Create issue via GitHub REST API with title, body from template, labels
    - Screenshot link in issue body (link back to dashboard)
    - Create `FeedbackExport` record linking item to tracker and issue

17. **Single export**
    - "Export to issue tracker" button on feedback detail page
    - Tracker selection dropdown when project has multiple trackers
    - Creates issue, creates `FeedbackExport` record, displays link
    - Export history section on detail page showing all exports for the item

> **Deferred to later phases:**
> - **Bulk export** — see section 3.9.3
> - **Additional adapters** — `LeantimeTrackerAdapter`, `GitLabTrackerAdapter`, `JiraTrackerAdapter`

### Phase 6: Polish & Production Readiness

**Goal:** Production-ready application.

18. **Data migration tool**
    - CLI command to import existing local tidy-feedback data into the central backend
    - Uses the existing `GET /tidy-feedback` API on each site (with basic auth) to fetch historical items — leveraging the existing API rather than reading SQLite files directly
    - Maps old `Item` entities to new `FeedbackItem` entities
    - Extracts screenshots from the `data` JSON blob and stores them on the filesystem
    - Associates imported items with a specified project
    - Tracks imported item IDs to avoid duplicates on re-run

19. **Operational concerns**
    - Health check endpoint (`/api/health`)
    - Logging and monitoring hooks
    - Backup strategy documentation
    - Deployment documentation (Docker / platform.sh / similar)
    - Screenshot cleanup job (Symfony Scheduler or cron) per retention policy (section 3.8.2)

20. **Testing**
    - Unit tests for API key auth, feedback ingestion, tracker adapters
    - Functional tests for dashboard (WebTestCase)
    - E2E tests with Panther for critical flows

> **Deferred to future phases:**
> - **Dashboard overview page** — summary statistics per project, sparklines, recent feedback
> - **CSV export** — export of filtered feedback for offline analysis
> - **Bulk export** — export multiple feedback items to issue trackers at once
> - **Additional tracker adapters** — Leantime, GitLab, Jira

---

## 6. File Structure (Backend Application)

```
tidy-feedback-backend/
├── assets/
│   ├── controllers/           # Stimulus controllers
│   │   ├── clipboard_controller.js
│   │   ├── confirm_controller.js
│   │   ├── filter_controller.js
│   │   ├── bulk_select_controller.js
│   │   └── collapsible_controller.js
│   ├── styles/
│   │   └── app.css
│   └── app.js
├── config/
│   ├── packages/
│   │   ├── doctrine.yaml
│   │   ├── security.yaml
│   │   ├── nelmio_cors.yaml
│   │   ├── rate_limiter.yaml
│   │   ├── messenger.yaml
│   │   └── ...
│   └── routes.yaml
├── src/
│   ├── Command/               # CLI commands
│   │   ├── CreateUserCommand.php
│   │   ├── CreateProjectCommand.php
│   │   └── ImportLegacyDataCommand.php
│   ├── Controller/
│   │   ├── Api/
│   │   │   └── FeedbackApiController.php
│   │   └── Dashboard/
│   │       ├── DashboardController.php
│   │       ├── FeedbackController.php
│   │       ├── ProjectController.php
│   │       └── ExportController.php
│   ├── Entity/
│   │   ├── Project.php
│   │   ├── FeedbackItem.php
│   │   ├── FeedbackNote.php
│   │   ├── FeedbackExport.php
│   │   ├── ExternalTrackerConfig.php
│   │   └── User.php
│   ├── Twig/Components/       # Live Components
│   │   └── FeedbackTable.php
│   ├── Repository/
│   ├── Security/
│   │   └── ApiKeyAuthenticator.php
│   ├── Service/
│   │   ├── ApiKeyManager.php
│   │   ├── FeedbackIngestionService.php
│   │   ├── ScreenshotStorageService.php
│   │   └── NotificationService.php
│   ├── ExternalTracker/       # Issue tracker integration
│   │   ├── ExternalTrackerInterface.php
│   │   ├── ExternalIssueResult.php
│   │   ├── ExternalTrackerRegistry.php
│   │   ├── Adapter/
│   │   │   └── GitHubTrackerAdapter.php  # v1 — other adapters added later
│   │   └── IssueBodyRenderer.php
│   ├── Message/               # Messenger messages
│   │   ├── SendFeedbackNotification.php
│   │   └── Handler/
│   │       └── SendFeedbackNotificationHandler.php
│   └── Enum/
│       ├── FeedbackStatus.php
│       ├── FeedbackCategory.php
│       └── TrackerType.php
├── templates/
│   ├── base.html.twig
│   ├── security/
│   │   └── login.html.twig
│   ├── emails/
│   │   └── new_feedback.html.twig
│   ├── dashboard/
│   │   ├── index.html.twig
│   │   ├── feedback/
│   │   │   ├── index.html.twig
│   │   │   ├── show.html.twig
│   │   │   └── _notes.html.twig
│   │   └── project/
│   │       ├── index.html.twig
│   │       ├── show.html.twig
│   │       ├── _form.html.twig
│   │       └── _tracker_form.html.twig
│   └── components/
│       └── FeedbackTable.html.twig
├── compose.yaml
├── Taskfile.yml
└── ...
```

---

## 7. Key Decisions & Rationale

**Symfony (not Drupal) for the backend** — The backend is a focused application (API + dashboard), not a content management system. Symfony gives full control with less overhead. The frontend widget already supports both Drupal and Symfony, so this is consistent.

**Stimulus + Turbo + Live Components (not React)** — Aligns with the team's preference to avoid complex JS frameworks. Symfony UX Live Components handle the most interactive part (filter/table) entirely in PHP + Twig. Stimulus handles small interactive behaviors (clipboard, bulk select, collapsible sections, confirmations). Turbo Drive provides SPA-like navigation with zero JS.

**Bootstrap 5 for CSS** — Battle-tested and already used in tidy-feedback. Symfony UX Toolkit (Tailwind-based) is promising but still maturing; it can be evaluated for a future redesign.

**PostgreSQL** — Better JSON querying support than MySQL for the `contextData` and `rawData` fields. Strong full-text search capabilities for the search filter.

**API key authentication (not OAuth)** — Simple, fits the use case. The widget is a trusted first-party component installed by the site operator, not an arbitrary third-party. API keys keep the integration simple (one header).

**Screenshots on filesystem (not in database)** — The current widget can generate screenshots of several megabytes. Storing binary data in the database would quickly cause bloat. Flysystem abstraction allows migrating to S3/object storage later without code changes.

**UUIDv7 as primary keys** — Sortable by creation time, no sequential ID guessing, safe to expose in URLs.

**Adapter pattern for issue trackers** — Cleanly separates tracker-specific API logic from the core application. Adding a new tracker is one new class implementing the interface. Starting with GitHub since the team already uses it. Leantime is prioritized next given team usage.

**Multiple trackers per project with per-item selection** — A project might use GitHub for code bugs and Leantime for product planning. Rather than forcing a single tracker, the dashboard lets users choose where to export each feedback item. This adds minor UI complexity but provides significant workflow flexibility.

**Export tracking separate from workflow status** — Export state is tracked via `FeedbackExport` records (OneToMany from FeedbackItem) rather than as a status enum value. This allows an item to be "resolved" and also exported, or exported to multiple trackers. The workflow status (`new` → `in_progress` → `resolved` → `closed`) reflects triage/resolution, not export state.

**One issue per feedback item (not combined)** — Each feedback item maps to a distinct problem and should be trackable independently in the issue tracker.

**Link-back for screenshots (not upload)** — When exporting to issue trackers, the issue body contains a link back to the screenshot in the feedback backend dashboard rather than uploading the image to the tracker. This avoids dealing with different upload APIs across trackers and keeps the screenshot hosted in one place.

---

## 8. Future Features

Features that are out of scope for the initial build but worth planning for architecturally.

### 8.1 Bi-directional Sync with Source Sites

Currently, feedback flows one way: from sites into the central backend. A future enhancement would sync status and notes **back** to the individual tidy-feedback instances, so site administrators can track progress directly on the site where the feedback was submitted.

This would require:
- A polling or webhook mechanism where the frontend widget (or site) periodically checks the backend for updates on items it submitted
- A `GET /api/v1/feedback?since={timestamp}` endpoint returning updated items for a given project
- The tidy-feedback widget/module displaying status updates and notes from the central backend alongside the local feedback list
- Consideration of conflict resolution (e.g. if status is changed both locally and centrally)

This is a significant feature that would likely warrant its own specification, but the current data model and API key authentication already provide the foundation.

### 8.2 Webhook / Slack Notifications

Extend the notification system beyond email to support webhooks (generic HTTP POST) and Slack integration. This would use the same async Messenger approach, with configurable notification channels per project.

### 8.3 Feedback Status Sync from Issue Trackers

When an issue is closed in GitHub or resolved in Leantime, automatically update the corresponding feedback item's status in the backend. This could be implemented via:
- Webhook receivers for each tracker type (GitHub webhooks, Leantime webhooks)
- Or periodic polling of exported issues to check for status changes

### 8.4 Dashboard User Scoping

Add project-level access control so specific users only see feedback from their assigned projects. Useful when the backend serves multiple teams or organizations.

### 8.5 Public Feedback Status Page

An optional, unauthenticated page per project showing submitted feedback and its current status — so end-users who submitted feedback can see that their report is being addressed. This would require a separate "public token" per feedback item (sent in the confirmation response) to avoid exposing the full list.

### 8.6 Bulk Export to Issue Trackers

Bulk export of multiple feedback items at once from the feedback list page. See section 3.9.3 for the full specification. Deferred from v1 due to UI complexity.

### 8.7 Dashboard Overview with Statistics

Summary overview page with feedback counts per project, sparklines/charts, recent feedback, and quick-filter buttons. Deferred from v1 — the home page redirects to the feedback list until there is enough data to make statistics meaningful.

### 8.8 CSV Export

Export filtered feedback as CSV for offline analysis or reporting. Deferred from v1.

### 8.9 Additional Tracker Adapters

- `LeantimeTrackerAdapter` — prioritized after GitHub given team usage
- `GitLabTrackerAdapter`
- `JiraTrackerAdapter`

### 8.10 Notification Throttling

When a site submits many feedback items in a burst, the configured notification emails should receive a digest rather than individual emails per item. Could use Symfony Messenger's delay/batch capabilities.
