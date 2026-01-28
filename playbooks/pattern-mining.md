# Pattern Mining Playbook

**Goal:** Extract UI patterns, conventions, and exemplary examples from an existing codebase to build AI context.

**Use with:** Claude Code, Cursor, or any AI coding assistant — run this in the Handshake repo.

---

## Phase 1: Reconnaissance — Understand the Landscape

### 1.1 Map the Component Architecture

```
Prompt:
---
Analyze the component structure in this codebase:

1. Where do shared/reusable components live? (e.g., src/components/, packages/ui/)
2. Where do page components live? (e.g., src/pages/, src/app/, routes/)
3. Where do feature-specific components live?
4. Is there a Rosetta/design system package? Where?

List the top-level directories and explain what each contains.
Then give me a count: how many components total, how many pages?
---
```

### 1.2 Identify the Tech Stack

```
Prompt:
---
What's the frontend tech stack?

Check package.json and codebase for:
- Framework (React, Next.js, Remix, etc.)
- Styling (styled-components, Tailwind, CSS modules, etc.)
- State management (Redux, Zustand, React Query, etc.)
- Routing approach
- Testing libraries
- Any design system packages (@joinhandshake/rosetta, etc.)

Summarize in a table: | Category | Tool | Notes |
---
```

### 1.3 Find the Entry Points

```
Prompt:
---
List all page/route components in this codebase.

For each, give me:
- File path
- Route URL (if determinable)
- Brief description (based on file name/content)

Group them by feature area if possible (jobs, profiles, employers, applications, etc.)
---
```

---

## Phase 2: Pattern Discovery — Find the Recurring Shapes

### 2.1 Identify Page Layout Patterns

```
Prompt:
---
Analyze the page components in this codebase and identify recurring layout patterns.

Look for:
- List/search pages (grids of cards, tables with filters)
- Detail pages (single item view with metadata, actions)
- Form pages (multi-step wizards, single forms)
- Dashboard pages (metrics, charts, summary cards)
- Settings pages (grouped preferences, toggles)

For each pattern type you find:
1. Name it
2. List 3-5 example pages that use it
3. Describe the common structure (header, sidebar, main content, etc.)
4. Note any variations

Output format:
## [Pattern Name]
**Examples:** [file paths]
**Structure:** [description]
**Variations:** [notes]
---
```

### 2.2 Identify UI Compositions (Molecules)

```
Prompt:
---
Find recurring UI compositions — components that combine multiple Rosetta primitives into reusable patterns.

Look for things like:
- Data tables (with sorting, filtering, pagination)
- Card grids (job cards, employer cards)
- Search + filter combinations
- Form sections (grouped inputs with labels, validation)
- Empty states
- Loading states / skeletons
- Error states
- Action bars (bulk actions, selection UI)
- Modal patterns (confirmation, forms, info)

For each composition:
1. Name it
2. List where it's used (3-5 examples)
3. What Rosetta components does it combine?
4. What props/configuration does it accept?
5. Any variations?

Focus on patterns used in 3+ places — those are the real patterns.
---
```

### 2.3 Identify Interaction Patterns

```
Prompt:
---
Analyze how this codebase handles common interactions:

1. **Navigation:** How do users move between pages? (sidebar, tabs, breadcrumbs)
2. **Modals vs Pages:** When are modals used vs full page navigation?
3. **Forms:** How is form state managed? Validation displayed? Submission handled?
4. **Data fetching:** How do components load data? Loading states? Error handling?
5. **Feedback:** How are success/error states communicated? (toasts, banners, inline)
6. **Bulk actions:** How do multi-select + bulk operations work?

For each, find 2-3 concrete examples and describe the pattern.
---
```

### 2.4 Find the Anti-Patterns

```
Prompt:
---
Look for inconsistencies and anti-patterns in the UI code:

1. Places where Rosetta components AREN'T used but should be
2. Hardcoded colors instead of design tokens
3. One-off styling that duplicates what Rosetta provides
4. Inconsistent patterns (same thing done differently in different places)
5. Components that should exist but don't (copy-pasted code)

List specific examples with file paths.
This tells us what to fix AND what to document as "don't do this."
---
```

---

## Phase 3: Convention Extraction — Capture the Idioms

### 3.1 Code Style Conventions

```
Prompt:
---
Analyze the coding conventions in this codebase:

1. **File naming:** How are components, hooks, utils named?
2. **Directory structure:** How are components organized? Co-located with tests?
3. **Import ordering:** Is there a consistent pattern?
4. **Component structure:** What's the typical order inside a component file?
   (imports, types, hooks, helpers, component, exports)
5. **Props patterns:** How are props typed? Spread patterns? Default values?
6. **Export patterns:** Default exports vs named exports?

Show me 2-3 example components that represent "the right way" to structure code here.
---
```

### 3.2 State Management Conventions

```
Prompt:
---
How does this codebase manage state?

1. **Local state:** When is useState used vs other patterns?
2. **Server state:** How is data fetching done? (React Query, SWR, custom hooks)
3. **Global state:** Is there Redux, Zustand, Context? When is each used?
4. **Form state:** How are forms managed? (React Hook Form, Formik, custom)
5. **URL state:** How is URL/query param state handled?

Find examples of each and explain the patterns.
---
```

### 3.3 Data Fetching Patterns

```
Prompt:
---
Analyze how pages fetch and display data:

1. Where does data fetching happen? (component level, page level, SSR)
2. How are loading states handled? (skeletons, spinners, suspense)
3. How are error states handled? (error boundaries, inline, toast)
4. How is data refreshed? (polling, refetch on focus, manual)
5. How is optimistic UI done, if at all?

Show me a "gold standard" example of a page that handles all states well.
---
```

### 3.4 Testing Conventions

```
Prompt:
---
How are UI components tested in this codebase?

1. What testing libraries are used? (Jest, RTL, Cypress, Playwright)
2. What's the typical test structure for a component?
3. What gets tested vs what doesn't?
4. Are there integration tests for pages/flows?
5. How is test data created? (fixtures, factories, mocks)

Find a well-tested component and use it as a reference example.
---
```

---

## Phase 4: Example Curation — Find the Exemplary

### 4.1 Identify Best-in-Class Pages

```
Prompt:
---
Find the 5 best-implemented pages in this codebase — the ones you'd point a new dev to as examples of "how we do things."

Criteria:
- Uses Rosetta components correctly
- Handles loading/error states
- Well-structured code
- Good accessibility
- Representative of common patterns

For each:
1. File path
2. What makes it good
3. What patterns it demonstrates
4. Any caveats or areas that could be better

These will become our annotated examples.
---
```

### 4.2 Identify Best-in-Class Components

```
Prompt:
---
Find the 5 best-implemented reusable components (not pages) — well-designed, well-tested compositions that others should copy.

Criteria:
- Clean API (good props design)
- Handles edge cases (empty, loading, error)
- Well-typed
- Accessible
- Has tests

For each:
1. File path
2. What it does
3. What makes it good
4. How it's used (example usage)
---
```

### 4.3 Document a Page End-to-End

```
Prompt:
---
Pick [JOB LISTING PAGE or specific page] and document it thoroughly:

1. **Purpose:** What does this page do?
2. **Route:** What URL serves this?
3. **Data flow:** What data does it fetch? From where? How?
4. **Components used:** What Rosetta components? What custom components?
5. **Layout:** How is it structured? (header, sidebar, main, etc.)
6. **States:** How does it handle loading, error, empty?
7. **Interactions:** What can users do? How is each action implemented?
8. **Responsive:** How does it adapt to mobile?

Then annotate the code — add comments explaining WHY, not just what.
---
```

---

## Phase 5: Output Generation — Create the Context Docs

### 5.1 Generate Pattern Doc

```
Prompt:
---
Based on our analysis, write a pattern document for [LIST PAGE / DATA TABLE / etc.]:

Use this format:

# [Pattern Name]

## When to Use
- [Use cases]
- NOT for: [anti-use-cases]

## Anatomy
[Describe structure]

## Components Used
- `<Component>` — purpose

## Layout Rules
- Container behavior
- Grid/spacing
- Responsive behavior

## Handshake Preferences
- "We prefer X over Y because..."
- "Always include Z"
- "Never do W"

## Example Code
[Production-quality example with annotations]

## Variations
- Variation A: when [context]

## Related Patterns
- [Links]

---
```

### 5.2 Generate Conventions Doc

```
Prompt:
---
Based on our analysis, write a conventions document for [CODE STYLE / STATE MANAGEMENT / etc.]:

Use this format:

# [Convention Area]

## Overview
[Brief description of what this covers]

## Rules

### [Rule 1]
- What to do
- Why
- Example

### [Rule 2]
...

## Examples

### Good ✓
[Code example]

### Bad ✗
[Code example]

## Exceptions
[When rules can be bent]

---
```

### 5.3 Generate Example Annotation

```
Prompt:
---
Create an annotated example for [PAGE/COMPONENT]:

# [Name]

## What This Demonstrates
- [Pattern 1]
- [Pattern 2]

## Key Decisions
1. **[Decision]** — [Why]

## Code (Annotated)
[Full code with // PATTERN: comments explaining key choices]

## Screenshot
[Describe or generate]

## When to Reference This
- When building [X]
- When implementing [Y]

---
```

---

## Phase 6: Validation — Test the Context

### 6.1 Test Generation Quality

```
Prompt:
---
Using only the context documents we created, build [A NEW FEATURE].

Do NOT look at existing code except through the documented patterns.

Let's see if the output matches Handshake style.

After generating, compare against existing similar features — does it fit?
---
```

### 6.2 Identify Gaps

```
Prompt:
---
What's missing from our documented patterns?

1. What questions would a dev have that aren't answered?
2. What patterns exist in code that we didn't document?
3. What edge cases aren't covered?

List the gaps so we can fill them.
---
```

---

## Quick Start: The 30-Minute Version

If you only have 30 minutes, run these in order:

```
1. Run 1.1 (component map) + 1.2 (tech stack) — understand the landscape

2. Run 2.1 (page layouts) — find the major page patterns

3. Run 4.1 (best pages) — identify exemplary code

4. Run 5.1 — generate ONE pattern doc for the most common pattern

That gives you: landscape understanding, key patterns identified, one documented pattern to test with.
```

---

## Output Checklist

After running this playbook, you should have:

- [ ] Component architecture map
- [ ] Tech stack summary
- [ ] 5+ identified page layout patterns
- [ ] 10+ identified UI compositions
- [ ] Interaction pattern documentation
- [ ] Anti-patterns list
- [ ] Code style conventions
- [ ] State management conventions
- [ ] Data fetching patterns
- [ ] 5 exemplary pages identified
- [ ] 5 exemplary components identified
- [ ] 1+ fully annotated example
- [ ] 2-3 pattern documents drafted
- [ ] Gap analysis

---

## Tips

1. **Start broad, then deep** — Map the landscape before diving into specifics
2. **Follow the repetition** — Patterns appear 3+ times. One-offs aren't patterns.
3. **Ask "why"** — Don't just document what, document why decisions were made
4. **Find the exemplars** — Best code teaches better than average code
5. **Test your output** — Generate something with your docs, see if it fits
6. **Iterate** — First pass won't be perfect. Refine based on gaps.
