# ADR-001: Database Engine — MariaDB

- **Status:** Accepted
- **Created by:** Jesper Pedersen
- **Date:** 2026-03-05
- **Stakeholders:** Development team, operations

## Context and Drivers

The Tidy Feedback Backend stores feedback items with rich context data
(browser info, viewport dimensions, page URL, highlighted regions) as
JSON blobs. The dashboard needs full-text search across feedback
subjects, descriptions, and reporter names. The application also needs
reliable support for UUIDv7 primary keys.

The ITK Dev `symfony-6` Docker template bundles MariaDB
(`itkdev/mariadb:latest`) as the default database engine. Using a
different engine means deviating from the standard template and
maintaining a custom Docker Compose database service.

### Functional drivers

- Store and query JSON context data (`contextData`, `rawData` fields)
- Full-text search across multiple text columns
- UUIDv7 as primary key type

### Non-functional drivers

- Alignment with ITK Dev Docker template conventions
- Team familiarity with the database engine
- Operational simplicity

## Options Considered

### Option A: MariaDB (ITK Dev default, chosen)

**Pros:**

- Full compatibility with the `symfony-6` Docker template
- Standard ITK Dev healthcheck and configuration patterns
- Team familiarity from other ITK Dev projects
- `JSON_EXTRACT` and `JSON_VALUE` available for JSON querying
- FULLTEXT indexes available for search functionality
- Consistent with all other ITK Dev projects

**Cons:**

- JSON querying is less powerful than PostgreSQL (`JSON_EXTRACT` vs
  native operators and GIN indexing)
- FULLTEXT indexes behave differently from PostgreSQL full-text search
  (no stemming, no ranking by default)
- No native UUID type; Doctrine maps UUIDs to `BINARY(16)` or
  `CHAR(36)`

### Option B: PostgreSQL

**Pros:**

- Native `jsonb` type with GIN indexes for efficient JSON querying
- Built-in full-text search with `tsvector`/`tsquery` and GIN indexes
- Native `uuid` type

**Cons:**

- Requires replacing the MariaDB service in `docker-compose.yml`
- Different healthcheck configuration
- Server compose file needs PostgreSQL-compatible external DB setup
- Deviates from the ITK Dev standard template
- Team less familiar with PostgreSQL operations

## Decision

Use **MariaDB** as the database engine.

Full alignment with the ITK Dev `symfony-6` Docker template is
prioritized over the advanced JSON and full-text search features of
PostgreSQL. MariaDB's `JSON_EXTRACT` is sufficient for reading
structured context data, and FULLTEXT indexes cover the dashboard
search needs. Doctrine ORM abstracts most database differences, so
the application code remains portable.

## Consequences

### Positive

- Zero template customization needed for the database service
- Standard ITK Dev healthcheck and deployment patterns
- Team familiarity reduces operational risk
- Consistent with all other ITK Dev projects

### Negative / Trade-offs

- JSON field querying uses `JSON_EXTRACT` function syntax instead of
  native operators
- Full-text search uses FULLTEXT indexes (no built-in stemming or
  ranking)
- UUIDs stored as `BINARY(16)` via Doctrine rather than a native type

### Follow-up actions

- Use Doctrine's `json` column type for `contextData` and `rawData`
- Create FULLTEXT indexes on `subject` and `description` columns
- Use Doctrine's `UuidType` for UUIDv7 primary keys (maps to
  `BINARY(16)` on MariaDB)
