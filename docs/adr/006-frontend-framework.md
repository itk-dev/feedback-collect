# ADR-006: Frontend Framework — Stimulus/Turbo over SPA Frameworks

- **Status:** Accepted
- **Created by:** Jesper Pedersen
- **Date:** 2026-03-04
- **Stakeholders:** Development team

## Context and Drivers

The dashboard needs interactive features: inline status updates, notes
without full page reloads, filter changes, and real-time indicators.
The question is whether to use a JavaScript SPA framework (React, Vue)
or server-rendered HTML enhanced with Symfony UX components.

### Functional drivers

- Status updates and note additions without full page reload
- Filter and sort operations on the feedback list
- Form interactions (project configuration, export dialogs)

### Non-functional drivers

- Team familiarity and maintenance burden
- Build complexity
- Alignment with Symfony ecosystem and ITK Dev conventions

## Options Considered

### Option A: React or Vue SPA

**Pros:**

- Rich client-side interactivity
- Large ecosystem of UI components

**Cons:**

- Separate build pipeline (Webpack/Vite)
- API-first architecture required (all data via JSON endpoints)
- Significantly more JavaScript to maintain
- Diverges from ITK Dev server-rendered conventions
- Doubles the codebase surface area (PHP backend + JS frontend)

### Option B: Stimulus + Turbo + Live Components (chosen)

**Pros:**

- Server-rendered HTML (Twig templates) as the primary output
- Stimulus controllers for targeted interactivity
- Turbo Frames for partial page updates without full reloads
- Live Components for real-time form interactions
- AssetMapper (no build step needed)
- Stays within the Symfony ecosystem
- Aligns with ITK Dev's server-rendered approach

**Cons:**

- Less suitable for highly complex client-side interactions
- Smaller ecosystem than React/Vue
- Some interactive patterns require more server round-trips

## Decision

Use **Stimulus**, **Turbo**, and **Symfony UX Live Components** for
all dashboard interactivity. Use **AssetMapper** for asset management
(no Webpack/Vite build step). Use **Bootstrap 5** for styling.

The dashboard's interactivity needs (status updates, notes, filters)
are well within what Turbo Frames and Stimulus controllers handle
efficiently. There is no need for a full SPA framework.

## Consequences

### Positive

- Single codebase (PHP + Twig + small Stimulus controllers)
- No JavaScript build pipeline
- Templates are the source of truth for UI
- Consistent with Symfony ecosystem and ITK Dev conventions
- Easier to maintain for a PHP-focused team

### Negative / Trade-offs

- If future requirements demand complex client-side state management,
  this approach may need to be reconsidered
- Some UI patterns require more creativity with Turbo Frames than
  they would with a client-side framework
