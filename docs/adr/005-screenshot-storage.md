# ADR-005: Screenshot Storage — Filesystem via Flysystem

- **Status:** Accepted
- **Created by:** Jesper Pedersen
- **Date:** 2026-03-04
- **Stakeholders:** Development team, operations

## Context and Drivers

Each feedback submission includes a screenshot captured by the widget
(WebP format, typically 200 KB to 2 MB). The existing tidy-feedback
module stores screenshots as base64-encoded data URIs inside a JSON
blob in SQLite — a known limitation.

### Functional drivers

- Screenshots must be served inline on the dashboard detail page
- Screenshots are linked from GitHub issues (via URL)
- Screenshots vary in size; some full-page captures can exceed 1 MB

### Non-functional drivers

- Database size management
- Serving performance for binary assets
- Future migration path to object storage (S3)

## Options Considered

### Option A: Database BLOB storage

**Pros:**

- Single data store; backup includes screenshots automatically
- No filesystem management

**Cons:**

- Bloats database size significantly
- Slower queries when scanning tables
- Database backups become large
- Not suitable for CDN serving

### Option B: Filesystem with Flysystem abstraction (chosen)

**Pros:**

- Screenshots stored as files, database stores only the path/filename
- Efficient serving via Nginx for static files
- Flysystem abstraction allows switching to S3 or other storage
  adapters without code changes
- Database stays lean

**Cons:**

- Two storage systems to manage (database + filesystem)
- Filesystem backups needed separately from database
- Local development needs a writable directory

## Decision

Store screenshots on the **local filesystem** using the **Flysystem**
abstraction layer (`league/flysystem-bundle`).

Screenshots are extracted from the base64 data URI in the submission
payload, saved as files with a deterministic path based on the
feedback item UUID, and the file path is stored in the database. The
Flysystem adapter can be swapped to S3 in future without application
code changes.

## Consequences

### Positive

- Database stays small and fast
- Screenshots can be served efficiently by Nginx
- Future migration to S3 requires only configuration changes
- Clear separation of structured data and binary assets

### Negative / Trade-offs

- Filesystem must be backed up separately from the database
- Local development requires a writable storage directory
  (`.docker/data/` or `var/storage/`)
- Deleting a feedback item must also clean up the screenshot file

### Follow-up actions

- Configure Flysystem with a local adapter for development and
  document the storage directory in `.gitignore`
- Add a screenshot cleanup step to the feedback item deletion logic
- Document the storage path convention in the backend spec
