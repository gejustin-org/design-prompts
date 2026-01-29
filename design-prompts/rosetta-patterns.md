# Rosetta Patterns

Patterns are reusable behavioral rules that govern how components compose and interact. If `rosetta.md` is the vocabulary (tokens, components, principles), this document is the grammar—how those pieces combine into consistent, purposeful UI.

Apply these patterns before reaching for custom solutions. When a pattern is marked **unresolved**, do not invent behavior—ask the user or wait for guidance.

---

## Interaction States

### Follow / Toggle Buttons
Toggle buttons (e.g. Follow) must reflect their current state, not the inverse action.
- **Do:** `Follow` → `Following` (describes current state)
- **Don't:** `Follow` → `Unfollow` (describes the destructive next action; creates anxiety)

### List & Row Hover States
Clickable rows must make interactivity discoverable on hover.
- **Do:** Apply a background color change (`--rosetta-interactive-hovered`) AND underline the primary text link to signal clickability.
- **Don't:** Rely on subtle color shifts alone—insufficient contrast between rest and hover makes rows feel static.

---

## Action Hierarchy

### One Primary Action Per Screen
Each screen has a single, clear primary action that guides the user's next step. Secondary and tertiary actions must be visually subordinate.
- **Do:** One `PrimaryButton` per page/view. Use `SecondaryButton` or text links for everything else.
- **Don't:** Place multiple primary buttons on the same page—they compete for attention and stall decision-making.

### Group Controls Together
Co-locate related controls instead of scattering them across a view. Proximity signals relationship.
- **Do:** Use a right chevron (`›`) to indicate navigable lists or sections. Keeps the action inline with its context.
- **Don't:** Use a separate "View all" link detached from the list—it fragments the interaction and adds a navigation hop.

---

## Content Constraints

### No Ornamentation
Avoid decorative elements, icons, or illustrations that don't serve a functional purpose. Every visual element must earn its place.
- Decorative elements increase visual debt and bundle size.
- If an icon doesn't aid comprehension or wayfinding, remove it.

### Character Count Limits
Body copy line length: **45–75 characters per line**. This is the optimal range for readability in paragraphs.
- **Do:** Constrain body text containers (e.g. `max-width: 65ch`) so lines stay within the readable range.
- **Don't:** Let body text run to full viewport width—lines over ~80 characters cause reading fatigue.

---

## Component Behavior

### Tooltips
Use tooltips only when a control's purpose is ambiguous from its label or icon alone.
- **Do:** Tooltip on an icon-only button to clarify its action.
- **Don't:** Default to tooltips as a crutch. If you need a tooltip, first ask: is the label or icon clear enough? Is this the right design or product decision?

### Tabs
Tabs switch between views within a single context. They must feel lightweight and immediate.
- **Do:** Left-align tabs. Use `linear` interpolation for the active indicator transition. Apply consistent padding between tab items.
- **Don't:** Stretch or pad tabs to fill the container width—it creates awkward whitespace and breaks scanning rhythm.

### More / Expand
Use inline expansion (`More`) to reveal additional content without navigating away. Preferred for content lists where the full set is nearby but not immediately needed.

---

## Unresolved Patterns

The following are identified in Rosetta but not yet specified. Do not invent behavior for these—ask or wait for guidance.

| Pattern | Open Question |
|:--------|:--------------|
| **Carousels** | Swipe/scroll physics, item count per viewport, peek behavior |
| **Skeleton vs. spinner loaders** | When to use each; skeleton shape fidelity |
| **Truncated text / "Show full copy"** | Ellipsis vs. full text; expand-in-place vs. popover |
| **Profile popover** | Trigger behavior, content hierarchy, dismissal |
| **Menus on mobile** | Full-screen vs. bottom sheet vs. inline expansion |
| **List vs. cards** | Decision criteria (data density, scanability, action count) |
