# ADR-003: UUIDv7 for Primary Keys

- **Status:** Accepted
- **Created by:** Jesper Pedersen
- **Date:** 2026-03-04
- **Stakeholders:** Development team

## Context and Drivers

The application exposes feedback item IDs in URLs (dashboard links,
API responses, GitHub issue body links). The existing tidy-feedback
module uses auto-increment integers for local storage.

### Functional drivers

- IDs appear in URLs shared across systems (dashboard, GitHub issues,
  API responses)
- Items must be sortable by creation time without an additional query
- IDs must be safe to expose publicly without leaking information
  about system usage

### Non-functional drivers

- Database indexing performance
- Compatibility with Doctrine ORM

## Options Considered

### Option A: Auto-increment integers

**Pros:**

- Simple, compact, familiar
- Efficient indexing

**Cons:**

- Sequential IDs reveal system usage (total items, submission rate)
- Collision risk if data is ever merged from multiple sources

### Option B: UUIDv4 (random)

**Pros:**

- No information leakage
- Globally unique

**Cons:**

- Not sortable by creation time
- Poor B-tree index performance due to randomness
- Larger storage than integers

### Option C: UUIDv7 (time-ordered, chosen)

**Pros:**

- Sortable by creation time (embedded timestamp)
- No sequential ID guessing
- Safe to expose in URLs
- Better B-tree index performance than UUIDv4 due to time-ordering
- Doctrine maps to `BINARY(16)` on MariaDB (compact storage)

**Cons:**

- Larger storage than integers (16 bytes vs 4/8 bytes)
- Requires PHP library for generation (symfony/uid)

## Decision

Use **UUIDv7** for all entity primary keys.

The time-ordered property provides natural sorting without additional
columns, and the non-sequential nature prevents information leakage.
Doctrine's `UuidType` maps these to `BINARY(16)` on MariaDB,
providing compact and efficient storage.

## Consequences

### Positive

- Dashboard URLs do not reveal system metrics
- Natural chronological ordering from the ID itself
- Safe for use in API responses and external references

### Negative / Trade-offs

- Slightly larger storage per row compared to integers
- Requires `symfony/uid` component
- Different from the auto-increment pattern in the existing
  tidy-feedback module (deduplication must use source site + local
  ID, not the central UUID)
