# Tidy Feedback Frontend — Feature Specification

## 1. Overview

`itk-dev/tidy-feedback` is a combined **Drupal module** and **Symfony bundle** that provides a floating feedback widget and a local dashboard. It is installed on **staging sites only** — not in production environments.

The module serves two distinct purposes:

1. **Widget** — A floating button on every page that opens a form allowing testers (including the Product Owner) to submit feedback with a screenshot, description, and contextual data.
2. **Local dashboard** — A simple admin interface at `/tidy-feedback` where testers can review all feedback items submitted from that site.

This document describes the current state of the module and the changes needed to support forwarding feedback to the central [Tidy Feedback Backend](./feedback-backend-spec.md).

---

## 2. Current State

### 2.1 Technology Stack

| Layer | Technology |
|-------|-----------|
| Module type | Drupal module + Symfony bundle (single package) |
| Language | PHP 8.x |
| Templating | Twig (watered-down instance with `trans` filter and `path` function) |
| JavaScript | Vanilla JS (`assets/` compiled via Webpack Encore) |
| CSS | SCSS (compiled via Webpack) |
| Database | Doctrine ORM against a dedicated SQLite database (`TIDY_FEEDBACK_DATABASE_URL`) |
| Auth (dashboard) | Basic HTTP auth via `TIDY_FEEDBACK_USERS` env var (JSON map of username → password) |

### 2.2 Installation

The module is installed via Composer:

```bash
composer require itk-dev/tidy-feedback:dev-main
```

After installation, the database schema is created via:

```bash
# Drupal
drush tidy-feedback:doctrine:schema-update

# Symfony
bin/console tidy-feedback:doctrine:schema-update
```

### 2.3 Configuration

```env
# Required: dedicated SQLite database for feedback storage
TIDY_FEEDBACK_DATABASE_URL=pdo-sqlite://localhost//app/tidy-feedback.sqlite

# Required: basic auth users for the local dashboard (JSON map)
TIDY_FEEDBACK_USERS='{"admin": "password"}'
```

For Drupal, these are set in `settings.local.php` via `putenv()`.

### 2.4 Routes

| Route | Description |
|-------|-------------|
| `/tidy-feedback` | Local dashboard — lists all feedback items (requires basic auth) |
| `/tidy-feedback/{id}` | Single feedback item detail (requires basic auth) |
| `/tidy-feedback/{id}/image` | Raw screenshot binary (requires basic auth) |
| `/tidy-feedback/test` | Test page for development — renders the widget in isolation |
| `POST /tidy-feedback` | Ingestion endpoint — receives widget submissions |

### 2.5 Existing Data Model (`Item` entity)

The current `Item` entity stores the following fields (local SQLite):

| Field | Type | Description |
|-------|------|-------------|
| `id` | `integer` (auto-increment) | Local primary key |
| `subject` | `string` | Feedback subject (required) |
| `description` | `text, nullable` | User-provided description |
| `createdBy` | `string, nullable` | Reporter email or name |
| `whatDidYouDo` | `text, nullable` | Steps the user took |
| `data` | `json` | Full context blob including screenshot (base64), URL, referrer, navigator, window, region, and optionally full HTML document |
| `createdAt` | `datetime_immutable` | Submission timestamp |

The screenshot is stored as a base64 data URI inside the `data` JSON blob — not as a separate file. This is a known limitation; the backend stores screenshots as separate files on the filesystem.

### 2.6 Widget Behaviour

The widget is injected into every page via the Twig template and a small JavaScript file. Key behaviours:

- **Trigger** — A floating button (bottom-right corner by default) opens the feedback form overlay
- **Screenshot capture** — On form open, `html2canvas` (or similar) captures the visible viewport as a WebP image; the user can draw a region to highlight
- **Form fields** — Subject (required), email/name, description, "what did you do"
- **Submission** — `fetch()` POSTs a JSON payload to `form.action` (currently always the local `/tidy-feedback` endpoint)
- **Context data** — The payload includes URL, referrer, user agent, window dimensions, highlighted region coordinates, and optionally the full `document.documentElement.outerHTML`
- **Development helpers** — Query string parameters allow auto-opening the form (`tidy-feedback-show=form`) and pre-filling fields (`tidy-feedback[subject]=...`) for testing

### 2.7 Local Dashboard

The local dashboard at `/tidy-feedback` is a simple Twig-rendered list protected by HTTP basic auth. It shows all items submitted from this site, with links to detail views and the raw screenshot binary.

This dashboard is the **primary interface for testers and the PO** during a test session. It allows them to:
- Review their own submissions
- Verify the screenshot was captured correctly
- Confirm the right context data was recorded

It is **not** intended for triage or project management — that is the role of the central backend dashboard.

---

## 3. Required Changes

The following changes are needed to support forwarding feedback to the central backend while maintaining full backwards compatibility.

### 3.1 New Configuration Variables

```env
# When set, feedback is also POSTed to the central backend
TIDY_FEEDBACK_REMOTE_URL=https://feedback.example.com/api/v1/feedback

# API key for authenticating with the central backend
TIDY_FEEDBACK_API_KEY=tf_xxxxxxxxxxxxxxxx

# Whether to include the full HTML document in the payload (opt-in, default: false)
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

For Drupal, these are set via environment variables or `settings.local.php`.

### 3.2 JavaScript Changes (`widget.js`)

The `fetch()` call in the form submit handler must be updated to support an optional remote target:

**Current behaviour:**
```javascript
fetch(form.action, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
})
```

**New behaviour:**
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

When `remoteUrl` is set, the widget POSTs to the central backend instead of the local endpoint. The local endpoint continues to work unchanged when `remoteUrl` is not configured.

### 3.3 HTML Document Capture (Opt-in)

The current widget unconditionally sends `document.documentElement.outerHTML` in the context payload. This is significant payload overhead for every submission. The change makes it opt-in:

```javascript
data.context = {
    url: document.location.href,
    referrer: document.referrer,
    navigator: { userAgent: navigator.userAgent },
    window: { innerWidth: window.innerWidth, innerHeight: window.innerHeight },
    region: { /* ... */ },
};

if (config.captureDocument) {
    data.context.document = document.documentElement.outerHTML;
}
```

This change should be applied regardless of whether `remoteUrl` is configured — the opt-in benefits local storage too.

### 3.4 Submission Toast

After successful submission, the widget currently shows a generic success state. This must be updated to show:

- A toast message: "Feedback submitted"
- A link to the feedback item in the central backend: `{TIDY_FEEDBACK_REMOTE_BASE_URL}/dashboard/feedback/{id}` (the `id` is returned in the `201 Created` response body from the backend)

The link should only appear when `remoteUrl` is configured and the backend returns a valid item `id` in the response.

> **Note:** This link requires the user to have a central backend account to be useful. See cross-cutting gap in the user stories document. Consider whether the link should be conditional or point to the local dashboard item instead.

### 3.5 Backwards Compatibility

When `TIDY_FEEDBACK_REMOTE_URL` is **not** set, the widget must continue to work exactly as it does today:
- POSTs to the local endpoint
- Stores in local SQLite
- No API key header is added
- No external network calls are made

This is critical — existing installations must not require any configuration changes to continue working.

---

## 4. API Key Security

The API key configured via `TIDY_FEEDBACK_API_KEY` will be visible in the rendered page source (embedded in the widget's JavaScript config object). This is acceptable because:

- The key is **write-only** — it can only be used to submit feedback for that specific project
- It cannot be used to read existing feedback items from the backend
- The central backend enforces rate limiting per API key
- The widget is only installed on staging sites, limiting the exposure surface

A server-side proxy approach (where the key is never sent to the browser) would add significant complexity for minimal security gain in this context.

---

## 5. Open Questions

1. **Does the local dashboard need to indicate that feedback was forwarded to the central backend?** Currently the local `Item` entity has no field for this. Adding a `forwardedAt` or `remoteId` field would allow the local dashboard to show a "also sent to central backend" indicator and link.

2. **What happens if the central backend is unreachable?** The current spec does not include a local fallback (localStorage/IndexedDB retry). Should the widget silently store locally and retry, or immediately surface the error to the tester? This is mentioned as a future consideration in section 2.1 of the backend spec.

3. **Should the widget still POST to the local endpoint when `remoteUrl` is configured?** The current spec implies the widget replaces the local POST with a remote one. But there may be value in posting to both — keeping a local copy while also forwarding to the central backend. This would require no backend changes and would give the PO a local record even if the central backend has downtime.

4. **Who configures the `TIDY_FEEDBACK_API_KEY` per site?** This value is generated by the central backend when a project is created (displayed once, then hashed). The site developer installs it via environment variable. The flow for rotating or regenerating the key should be documented clearly in the backend's project edit page.

---

## 6. File Structure (Relevant to Changes)

```
tidy-feedback/
├── assets/
│   └── widget.js          # ← Primary change: fetch() update, toast update
├── src/
│   └── Entity/
│       └── Item.php       # ← Potential: add forwardedAt / remoteId fields
├── symfony/src/
│   └── DependencyInjection/
│       └── Configuration.php  # ← Add remote_url, api_key, capture_document config nodes
├── templates/
│   └── widget.html.twig   # ← Pass remoteUrl, apiKey, captureDocument to JS config
└── ...
```
