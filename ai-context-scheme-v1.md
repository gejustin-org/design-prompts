# AI Context Scheme for Handshake UI Development

**Problem:** Current design prompt tells AI what Rosetta *is* (atoms, tokens, components), but not how Handshake *uses* Rosetta. Result: AI generates valid Rosetta UI, but not authentic Handshake UI.

**Goal:** Design a context scheme so AI can build UIs the way a Handshake dev would.

---

## The Core Insight

Handshake devs don't just know Rosetta — they know:
1. **The vocabulary** — What components and tokens exist
2. **The grammar** — How components compose into patterns
3. **The idioms** — Handshake-specific conventions and preferences
4. **The examples** — What real pages look like
5. **The codebase** — Where things live, how to import, how data flows

Your current prompt covers #1. We need #2-5.

---

## Context Layers

```
┌─────────────────────────────────────────────────────────────────────┐
│                     LAYER 5: LIVE CODEBASE                          │
│  Dynamic context from the actual repo (MCP, grep, file reads)       │
└─────────────────────────────────────────────────────────────────────┘
                                  ▲
┌─────────────────────────────────────────────────────────────────────┐
│                     LAYER 4: EXAMPLES                                │
│  Real pages/components from the codebase with annotations           │
└─────────────────────────────────────────────────────────────────────┘
                                  ▲
┌─────────────────────────────────────────────────────────────────────┐
│                     LAYER 3: CONVENTIONS                             │
│  Handshake coding patterns, file structure, state management        │
└─────────────────────────────────────────────────────────────────────┘
                                  ▲
┌─────────────────────────────────────────────────────────────────────┐
│                     LAYER 2: PATTERNS                                │
│  UI compositions, page layouts, common workflows                    │
└─────────────────────────────────────────────────────────────────────┘
                                  ▲
┌─────────────────────────────────────────────────────────────────────┐
│                     LAYER 1: DESIGN SYSTEM (Current)                │
│  Rosetta tokens, components, visual rules                           │
└─────────────────────────────────────────────────────────────────────┘
```

**Key principle:** Each layer builds on the previous. Higher layers are more specific, lower layers are more foundational.

---

## Layer 1: Design System (What You Have)

**Purpose:** Define Rosetta vocabulary — tokens, components, styling rules.

**Current file:** `rosetta.md` (~43KB)

**What it covers:**
- Design philosophy and vibes
- Color tokens and semantic usage
- Typography system
- Spacing, borders, shadows
- Component styling (Button, Card, Input, etc.)
- Accessibility requirements

**What it lacks:**
- How these pieces combine
- Handshake-specific preferences within Rosetta
- Real-world examples

**Verdict:** Keep as-is. This layer is solid. It's the foundation.

---

## Layer 2: Patterns — The Missing Grammar

**Purpose:** Document how Rosetta components compose into higher-level UI patterns that Handshake uses repeatedly.

**This is your biggest gap.**

### Proposed Structure

```
patterns/
├── INDEX.md                    # Discovery document
├── page-layouts/               # Full page structures
│   ├── list-page.md            # Job listings, employer search, etc.
│   ├── detail-page.md          # Job detail, profile view
│   ├── form-page.md            # Multi-step forms, applications
│   ├── dashboard-page.md       # Analytics, admin views
│   └── settings-page.md        # Preferences, account settings
├── compositions/               # Reusable UI blocks
│   ├── data-table.md           # Tables with sorting, filtering, pagination
│   ├── card-grid.md            # Grid of cards (jobs, employers)
│   ├── search-filters.md       # Faceted search patterns
│   ├── empty-state.md          # No results, first-time user
│   ├── loading-state.md        # Skeletons, spinners, progressive
│   ├── error-state.md          # Error handling patterns
│   ├── form-sections.md        # Form grouping, validation display
│   └── action-bar.md           # Bulk actions, selection patterns
├── workflows/                  # Multi-page flows
│   ├── application-flow.md     # Student applying to job
│   ├── posting-flow.md         # Employer creating job post
│   └── onboarding-flow.md      # First-time user setup
└── interactions/               # Micro-interactions
    ├── modals.md               # When/how to use modals vs pages
    ├── toasts.md               # Success/error feedback
    ├── tooltips.md             # Help text patterns
    └── navigation.md           # Sidebar, breadcrumbs, tabs
```

### Pattern Document Format

Each pattern doc should follow this structure:

```markdown
# [Pattern Name]

## When to Use
- Use case 1
- Use case 2
- NOT for: [anti-use-cases]

## Anatomy
[Diagram or description of the pattern structure]

## Components Used
- `<Button>` — for primary actions
- `<Card>` — for each item
- `<Pagination>` — for navigation
- etc.

## Layout Rules
- Container max-width
- Grid columns
- Spacing between elements
- Responsive behavior

## Handshake Preferences
- "We prefer X over Y because..."
- "Always include Z"
- "Never do W"

## Example Code
[Annotated, production-quality example]

## Variations
- Variation A: when [context]
- Variation B: when [context]

## Related Patterns
- [Link to related pattern]
```

### Source of Truth

**Where does this come from?**

1. **Mining the codebase** — Extract patterns from existing pages
2. **Design team input** — What patterns do designers think in?
3. **Code review history** — What feedback comes up repeatedly?
4. **Storybook** — If compositions are documented there, extract

**Maintenance:**
- Update when new patterns emerge
- Flag deprecated patterns
- Version alongside codebase

---

## Layer 3: Conventions — The Idioms

**Purpose:** Document Handshake-specific coding conventions that affect UI implementation.

### Proposed Structure

```
conventions/
├── INDEX.md                    # Discovery document
├── code-style.md               # Naming, imports, file organization
├── component-architecture.md   # How components are structured
├── state-management.md         # How data flows, where state lives
├── api-integration.md          # How to fetch data, handle loading/error
├── testing.md                  # What tests to write for UI
├── accessibility.md            # A11y patterns beyond Rosetta basics
└── performance.md              # Bundle considerations, lazy loading
```

### Key Content for Each

**code-style.md:**
```markdown
# Code Style for UI Components

## File Naming
- Components: `PascalCase.tsx`
- Hooks: `useCamelCase.ts`
- Utils: `camelCase.ts`
- Tests: `*.test.tsx` or `*.spec.tsx`

## Import Order
1. React/external libraries
2. Rosetta components
3. Internal components
4. Hooks
5. Utils
6. Types
7. Styles

## Component Structure
[Standard component template]

## Props Patterns
- Prefer composition over configuration
- Spread `...rest` to root element
- Use `className` merging pattern
```

**component-architecture.md:**
```markdown
# Component Architecture

## Atomic Design Mapping
- Rosetta provides atoms (Button, Input) and some molecules
- Pages assemble organisms from Rosetta + custom compositions
- Don't create new atoms — use Rosetta

## Directory Structure
src/
├── components/
│   ├── shared/           # Cross-feature components
│   ├── [feature]/        # Feature-specific components
│   └── layouts/          # Page layouts
├── pages/ or app/        # Route components
└── hooks/                # Shared hooks

## Co-location
- Component + styles + tests in same directory
- Export through index.ts
```

### Source of Truth

1. **Existing CONTRIBUTING.md or style guides**
2. **ESLint/Prettier configs** — Codified conventions
3. **Architecture decision records (ADRs)**
4. **Tribal knowledge** — Interview senior devs

---

## Layer 4: Examples — The Proof

**Purpose:** Show AI what real Handshake UI looks like with annotated examples.

### Approach: Curated Examples, Not Exhaustive

Don't dump the whole codebase. Select **5-10 exemplary pages/components** that demonstrate:
- Good pattern usage
- Handshake idioms
- Production quality

### Proposed Structure

```
examples/
├── INDEX.md                    # What examples exist and when to reference
├── job-listing-page/
│   ├── README.md               # Explanation and annotations
│   ├── page.tsx               # The actual code (snapshot)
│   └── screenshot.png         # Visual reference
├── job-detail-page/
│   └── ...
├── employer-search/
│   └── ...
├── application-form/
│   └── ...
└── profile-page/
    └── ...
```

### Example README Format

```markdown
# Job Listing Page

## What This Demonstrates
- List page layout pattern
- Search + filter composition
- Card grid with pagination
- Empty state handling
- Loading skeleton pattern

## Key Decisions
1. **Filters on left sidebar** — On desktop, filters are persistent. On mobile, filters are in a drawer.
2. **Card design** — Uses Rosetta Card with custom JobCard composition
3. **Pagination vs infinite scroll** — We chose pagination because [reason]

## Annotations
[Link to annotated code or inline comments]

## When to Reference This
- Building any list/search page
- Implementing filter UI
- Adding pagination

## Screenshot
[Visual showing the page]
```

### Source of Truth

1. **Pick best-in-class pages from codebase**
2. **Snapshot the code** (with commit hash for versioning)
3. **Add annotations** explaining why, not just what
4. **Include visual** — screenshots help AI understand intent

### Maintenance

- Re-snapshot when significant changes happen
- Link to commit/PR that changed it
- Flag outdated examples

---

## Layer 5: Live Codebase — Dynamic Context

**Purpose:** Real-time access to the actual codebase for specific queries.

### Mechanisms

1. **Storybook MCP** — Component discovery
   - `list-all-components` — What exists?
   - `get-component-docs` — How to use it?
   - `get-story-urls` — Visual reference

2. **Codebase search** — Find real implementations
   - grep for patterns
   - Read existing pages as reference
   - Check how similar features were built

3. **Type definitions** — Understand data shapes
   - API response types
   - Component prop types

### When to Use Each

| Need | Mechanism |
|------|-----------|
| "What components exist?" | Storybook MCP |
| "How was X built?" | Codebase search |
| "What data does Y receive?" | Type definitions |
| "Does Z already exist?" | Codebase search |

---

## How Layers Link Together

### Cross-References

Each document should link to related docs:

```markdown
<!-- In patterns/compositions/data-table.md -->
## Related
- **Design System:** See `rosetta.md#tables` for base styling
- **Conventions:** See `conventions/api-integration.md` for data fetching
- **Examples:** See `examples/job-listing-page/` for real usage
```

### The Index Files

Each layer has an `INDEX.md` that:
1. Lists all documents in that layer
2. Provides brief descriptions
3. Suggests when to load each

This enables selective loading — AI reads INDEX first, then loads only what's relevant.

---

## Discovery Mechanism — How AI Knows What to Load

### Option A: Explicit Task Context

```markdown
# Task: Build a job listing page

## Required Context (Load These)
1. `rosetta.md` — Design system (always)
2. `patterns/page-layouts/list-page.md` — Page structure
3. `patterns/compositions/data-table.md` — Table pattern
4. `patterns/compositions/search-filters.md` — Filter pattern
5. `examples/job-listing-page/` — Reference implementation
6. `conventions/api-integration.md` — Data fetching
```

**Pros:** Precise, efficient, no wasted context
**Cons:** Requires knowing what's relevant upfront

### Option B: Hierarchical Loading

```
Start with: INDEX.md (master index)
    ↓
Load: rosetta.md (always)
    ↓
AI reads task → selects relevant patterns/INDEX.md entries
    ↓
Load: selected pattern docs
    ↓
AI checks: examples/INDEX.md for relevant examples
    ↓
Load: selected examples
    ↓
Query: Live codebase via MCP for specifics
```

**Pros:** AI can self-navigate
**Cons:** More back-and-forth, token overhead

### Option C: Pre-composed Context Bundles

Create task-specific bundles:

```
context-bundles/
├── build-list-page.md          # Aggregated context for list pages
├── build-detail-page.md        # Aggregated context for detail pages
├── build-form.md               # Aggregated context for forms
└── migrate-to-rosetta.md       # Context for migration tasks
```

Each bundle includes relevant excerpts from all layers, pre-composed for that task type.

**Pros:** Single file load, optimized for common tasks
**Cons:** Duplication, maintenance burden, less flexible

### Recommendation: Hybrid Approach

1. **Master INDEX** — AI always starts here
2. **Task detection** — INDEX suggests which context to load based on task type
3. **Lazy loading** — AI loads only what it needs
4. **Context budget** — Define max token budget, prioritize within it

---

## Efficient Context Strategy

### The Token Budget Problem

- rosetta.md alone is ~43KB
- Adding patterns, conventions, examples could explode
- AI has finite context window

### Solutions

1. **Layered Loading**
   - Always: rosetta.md core sections (tokens, components)
   - On-demand: Pattern docs relevant to task
   - Reference: Examples only when similar work exists

2. **Summarization**
   - Each doc has a TL;DR at top
   - AI can read TL;DR, then load full doc if needed

3. **Chunking**
   - Break large docs into focused sections
   - Reference by section: `rosetta.md#buttons`

4. **Caching/Prefetch**
   - For tools like Cursor/Claude Code, system prompts can include persistent context
   - Load design system once per session, not per request

### Practical Budget

| Layer | Estimated Size | Load Strategy |
|-------|----------------|---------------|
| Design System | ~40KB | Always (core sections ~15KB) |
| Patterns | ~5KB per doc | On-demand |
| Conventions | ~3KB per doc | On-demand |
| Examples | ~10KB per example | Reference only |
| Live Codebase | Dynamic | Query as needed |

**Target:** Keep active context under 50KB for most tasks.

---

## File Structure (Full Proposal)

```
.claude/                              # Claude-specific context
├── design-system/
│   └── rosetta.md                    # Layer 1: What Rosetta is
├── patterns/
│   ├── INDEX.md                      # Pattern discovery
│   ├── page-layouts/
│   │   ├── list-page.md
│   │   ├── detail-page.md
│   │   ├── form-page.md
│   │   └── ...
│   ├── compositions/
│   │   ├── data-table.md
│   │   ├── card-grid.md
│   │   ├── search-filters.md
│   │   ├── empty-state.md
│   │   └── ...
│   ├── workflows/
│   │   └── ...
│   └── interactions/
│       └── ...
├── conventions/
│   ├── INDEX.md                      # Conventions discovery
│   ├── code-style.md
│   ├── component-architecture.md
│   ├── state-management.md
│   └── ...
├── examples/
│   ├── INDEX.md                      # Examples discovery
│   └── [feature]/
│       ├── README.md                 # Annotations
│       ├── code.tsx                  # Snapshot
│       └── screenshot.png            # Visual
├── context-bundles/                  # Optional: pre-composed contexts
│   ├── build-list-page.md
│   └── ...
└── INDEX.md                          # Master discovery document
```

---

## Index Document Format (Master)

```markdown
# Handshake UI Development Context

## How to Use This

1. **Always load:** `design-system/rosetta.md` (or core sections)
2. **Check task type:** Use table below to find relevant patterns
3. **Load patterns:** Read pattern INDEX, then specific docs
4. **Reference examples:** Check if similar work exists
5. **Query codebase:** Use MCP for specifics

## Task → Context Mapping

| Task Type | Patterns | Conventions | Examples |
|-----------|----------|-------------|----------|
| Build list/search page | list-page, data-table, search-filters | api-integration | job-listing-page |
| Build detail page | detail-page, action-bar | state-management | job-detail-page |
| Build form | form-page, form-sections | validation, api-integration | application-form |
| Add component | — | component-architecture, code-style | — |
| Migrate to Rosetta | — | code-style | — |

## Context Budget Guide

- **Quick task (<30 min):** rosetta.md core + 1 pattern
- **Medium task (1-2 hr):** rosetta.md full + 2-3 patterns + 1 example
- **Large feature:** Full context, multiple examples, MCP queries
```

---

## Creation Workflow — How to Build This

### Phase 1: Foundation (Week 1)

1. **Create file structure** — Empty directories with INDEX.md stubs
2. **Move rosetta.md** — Into `.claude/design-system/`
3. **Write master INDEX.md** — Discovery document
4. **Document 2-3 key patterns** — Start with most common (list-page, data-table)

### Phase 2: Pattern Mining (Week 2-3)

1. **Audit codebase** — Identify repeated patterns
2. **Document top 10 patterns** — Cover 80% of use cases
3. **Create 3-5 annotated examples** — Best-in-class pages
4. **Write core conventions** — code-style, component-architecture

### Phase 3: Integration (Week 4)

1. **Test with AI tools** — Does context improve output?
2. **Create context bundles** — For common task types
3. **Set up MCP integration** — Storybook, codebase search
4. **Write Cursor rules / Claude skills** — Automate context loading

### Phase 4: Maintenance (Ongoing)

1. **Update patterns when codebase evolves**
2. **Refresh examples periodically**
3. **Collect feedback** — What context is missing?
4. **Prune unused context** — Remove what doesn't help

---

## Success Metrics

1. **Quality:** AI-generated UI passes code review first try
2. **Consistency:** Output matches existing patterns
3. **Efficiency:** Context loads under 50KB for most tasks
4. **Developer validation:** "This looks like us"

---

## Next Steps

1. **Validate this scheme** — Does it match your mental model?
2. **Prioritize patterns** — Which 5 patterns are most needed?
3. **Identify example pages** — Which existing pages are exemplary?
4. **Define conventions** — What tribal knowledge needs documenting?

---

## Open Questions

1. **Versioning:** How do we keep context in sync with codebase?
2. **Multiple codebases:** Does Handshake have separate repos that need different context?
3. **Team adoption:** How do we get designers/devs contributing to patterns?
4. **Tooling:** Cursor rules vs Claude skills vs MCP — which integrations matter most?
