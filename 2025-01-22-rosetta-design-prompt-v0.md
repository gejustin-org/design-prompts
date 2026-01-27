# Rosetta Design Prompt v0 Implementation Plan

**Goal:** Prove AI-assisted design system development works end-to-end, then productionize.

**Template Sources:**
- `tmp/design-prompts/rosetta.md` - Rosetta-specific design prompt (rough draft)
- `tmp/design-prompts/enterprise.md`, `professional.md`, `saas.md` - designprompts.dev examples

---

# Phase 0: POC / Show & Tell

**Goal:** Stand up the minimum to demo the flow end-to-end. Use rough prompt, prove the concept.

**Time:** ~1-2 hours

## POC Setup

### Step 1: Copy rough prompt into place
```bash
mkdir -p .claude
cp tmp/design-prompts/rosetta.md .claude/rosetta-design-prompt.md
```

### Step 2: Configure MCPs (if not already)
```bash
# Chrome DevTools MCP for visual verification
claude mcp add chrome-devtools --scope user npx chrome-devtools-mcp@latest

# Storybook MCP for component discovery (if Storybook running)
claude mcp add storybook-mcp --transport http http://localhost:6006/mcp --scope project
```

### Step 3: Start services
```bash
yarn dev          # App at localhost:3000
yarn storybook    # Storybook at localhost:6006 (if available)
```

## POC Demo 1: Build a New Feature

**Scenario:** "Add an empty state to the jobs list page"

**Script:**
1. Load the design prompt: `Read .claude/rosetta-design-prompt.md`
2. Discover existing components: `mcp__storybook__list-all-components` (or grep hai-ui)
3. Write the empty state using Rosetta patterns
4. Navigate to page: `mcp__chrome-devtools__navigate_page`
5. Take snapshot to verify structure
6. Take screenshot to verify visual
7. Test dark mode
8. Iterate if needed

**Success criteria:**
- AI uses correct HAI-UI components (`<Icon>`, `<Button>`)
- AI uses semantic colors (`text-secondary-foreground`, not `text-gray-500`)
- Visual verification catches any issues before claiming done

## POC Demo 2: Migrate an Existing Component

**Scenario:** Find a component with arbitrary Tailwind colors, migrate to Rosetta

**Script:**
1. Find a migration target:
   ```bash
   grep -r "bg-gray-" src/ --include="*.tsx" | head -5
   ```
2. Load design prompt
3. Show AI the before state
4. Ask AI to migrate to Rosetta patterns
5. Verify visually - should look the same but use semantic tokens
6. Test dark mode - this is where arbitrary colors break

**Success criteria:**
- AI correctly identifies arbitrary colors
- AI maps to correct semantic tokens
- Dark mode works after migration (didn't before)

## POC Definition of Done

- [ ] Design prompt file in place (rough is fine)
- [ ] Chrome DevTools MCP working
- [ ] Demo 1: New feature built with Rosetta patterns, verified visually
- [ ] Demo 2: Existing component migrated, dark mode works
- [ ] Recorded or live demo ready for show & tell

---

# Phase 1: Productionize

**Goal:** Polish the prompt, add tooling, make it repeatable for HAI migrations.

**Architecture:**
- Main design prompt document (`.claude/rosetta-design-prompt.md`) follows the `<role>` + `<design-system>` structure
- Cursor rule references the prompt for quick lookups
- Claude skill enforces component discovery + visual verification workflow
- Playbook documents multi-MCP usage (Storybook MCP + Chrome DevTools MCP)

**MCP Strategy:**
- **Storybook MCP** → Component discovery ("What exists?")
- **Chrome DevTools MCP** → Visual verification ("Does it look right?")

**Definition of Done:**
- [ ] Rosetta design prompt polished with full template structure
- [ ] Cursor rule references the prompt
- [ ] Claude skill loads the prompt and enforces visual verification
- [ ] Playbook documents multi-MCP workflow
- [ ] Demo: HAI engineer can use tooling to migrate a page

---

## Task 1: Polish Rosetta Design Prompt Document

**Files:**
- Polish: `.claude/rosetta-design-prompt.md` (created in Phase 0)

**Step 1: Enhance the design prompt with full template structure**

Take the rough `rosetta.md` and enhance with:

```markdown
<role>
You are an expert frontend engineer building within the Rosetta design system. Your goal is to write frontend code that is visually consistent, accessible, and uses existing components correctly.

Before writing any code:
- Check if a Rosetta/HAI-UI component exists for what you need
- Understand the semantic color tokens (never use arbitrary Tailwind colors)
- Review existing patterns in the codebase for similar UI

When writing code:
- Use HAI-UI components from `@/components/hai-ui` - never raw HTML for interactive elements
- Use semantic color classes (`bg-primary`, `text-secondary-foreground`) - never `bg-gray-100`
- Use the `<Icon>` component - never import from lucide-react directly
- Handle loading and error states for all data-fetching components
- Verify visually using browser tools before claiming work is complete

Always aim to:
- Preserve accessibility (focus states, contrast, semantic HTML)
- Match the Rosetta aesthetic (clean, accessible, intentionally minimal)
- Leave the codebase more consistent than you found it
</role>

<design-system>
# Design Style: Rosetta

## 1. Design Philosophy

### Core Principle

**Professional clarity through semantic consistency.** Rosetta is a design system for work tools - it prioritizes accessibility, theme-awareness, and composability over decoration. Every color is semantic, every component is intentional, every interaction is accessible.

### The Visual Vibe

**Clean. Accessible. Intentionally minimal. Theme-aware.**

Rosetta feels like a well-designed productivity tool - the kind where you don't notice the UI because it never gets in your way. It's not playful, not decorative, not trendy. It's the design equivalent of good prose: clear, purposeful, and invisible when it's working.

**Emotional Keywords:**
- *Professional* — This is a tool for getting work done, not a portfolio piece
- *Accessible* — WCAG compliance is non-negotiable, not an afterthought
- *Minimal* — Every element earns its place; nothing is decorative
- *Consistent* — The same patterns appear everywhere; no surprises
- *Theme-aware* — Light and dark modes are first-class citizens

### What Rosetta is NOT

- **Not playful or whimsical** — No rounded avatars as CTAs, no emoji in UI chrome
- **Not decorative** — No gradients for decoration, no shadows for drama
- **Not trendy** — No glassmorphism, no neo-brutalism, no skeuomorphism
- **Not cold** — The lime accent and rounded corners add warmth
- **Not arbitrary** — Every color has a semantic meaning; no `bg-gray-100`

### The DNA of Rosetta

#### 1. The Lime Signature

The lime green accent (`#d3fb52` in dark mode, `#121212` primary in light) is Rosetta's brand marker. It appears sparingly but distinctly:

**Where it appears:**
- Primary CTAs in dark mode
- Interactive highlights
- Active states and selections
- Brand moments

**Why it works:** In a sea of blue SaaS products, lime is memorable. It's energetic without being aggressive, distinctive without being distracting.

#### 2. Semantic Color System

Colors in Rosetta have meaning, not just values. You don't pick "gray-100" - you pick "secondary background" and the system handles light/dark modes automatically.

**The hierarchy:**
- `surface` → `primary` → `secondary` → `tertiary` (increasing emphasis)
- `interactive-*` for anything clickable
- `status-*` for feedback (positive, negative, warning, info)

#### 3. Layered Interaction States

Hover and active states don't swap colors - they layer translucent overlays on the base color. This creates consistent interaction feedback across all color contexts.

**The pattern:** `bg-{base}-with-{overlay}`
- `bg-primary-with-interactive-hover`
- `bg-surface-with-interactive-active`

#### 4. Component-First Architecture

Rosetta isn't just tokens - it's components. When you need a button, you use `<Button>`. When you need a modal, you use `<Dialog>`. The components encode the patterns so you don't have to remember them.

#### 5. Accessibility by Default

Every Rosetta component ships with:
- Proper focus states (visible rings, not just outlines)
- WCAG AA contrast ratios
- Semantic HTML and ARIA where needed
- Keyboard navigation support

### Sensory Description

If Rosetta were a physical space, it would be:
- A clean, well-lit coworking space
- White walls with one accent wall in muted lime
- Ergonomic furniture that disappears when you're working
- Natural light, no fluorescents
- The quiet hum of focused productivity

If it were music, it would be:
- Lo-fi beats for concentration
- Consistent rhythm, no surprises
- Background music that helps you focus, not music you listen to
- Professional enough for a video call, interesting enough to not be elevator music

---

## 2. Design Token System

### Color Strategy

**Semantic over arbitrary.** Every color has a purpose. Never use Tailwind's arbitrary colors (`bg-gray-100`, `text-blue-500`).

#### Background Colors

| Token | Light | Dark | When to Use |
|-------|-------|------|-------------|
| `bg-surface` | `#fcfcfd` | `#121212` | Page background, base layer |
| `bg-primary` | `#ffffff` | `#2f2f2f` | Cards, panels, elevated surfaces |
| `bg-secondary` | `rgba(18,18,18,0.06)` | `#1d1d1f` | Subtle differentiation, table rows, hover states |
| `bg-tertiary` | `rgba(18,18,18,0.12)` | `#363636` | Selected states, active items, emphasized areas |
| `bg-disabled` | `rgba(18,18,18,0.09)` | `#2b2b2b` | Disabled elements (pair with `cursor-not-allowed`) |
| `bg-inverted` | `#2f2f2f` | `#fbfbfb` | Inverted sections, toasts, dark CTAs in light mode |

#### Foreground Colors

| Token | When to Use |
|-------|-------------|
| `text-primary-foreground` | Primary text, headings, important content |
| `text-secondary-foreground` | Secondary text, descriptions, less important content |
| `text-tertiary-foreground` | Hints, placeholders, least important content |
| `text-inverted-foreground` | Text on inverted/dark backgrounds |
| `text-disabled-foreground` | Disabled text (pair with `bg-disabled`) |
| `border-fg-border` | **The only border color** - use for all borders and dividers |

#### Interactive Colors

| Token | When to Use |
|-------|-------------|
| `bg-interactive-primary` | Primary buttons, main CTAs |
| `bg-interactive-secondary` | Secondary interactive elements (lime in dark mode) |
| `bg-interactive-positive` | Success actions, confirmations, "yes" actions |
| `bg-interactive-negative` | Destructive actions, errors, "delete" actions |
| `bg-interactive-warning` | Warning actions, caution states |
| `bg-interactive-info` | Info actions, focus rings, links |

#### Status Colors (for Banners, Tags, Badges)

| Token | When to Use |
|-------|-------------|
| `bg-status-positive` | Success messages, completed states |
| `bg-status-warning` | Warning messages, needs attention |
| `bg-status-negative` | Error messages, failed states |
| `bg-status-info` | Informational messages, neutral alerts |
| `bg-status-neutral` | Default/neutral state |

#### Layered Colors (Hover/Active States)

Pattern: `bg-{base}-with-{overlay}`

```tsx
// Button with hover state
<button className="bg-primary hover:bg-primary-with-interactive-hover">

// Selectable card
<div className="bg-surface aria-selected:bg-surface-with-interactive-active">

// Inverted button hover
<button className="bg-inverted hover:bg-inverted-with-interactive-inverted-hover">
```

**Available bases:** `surface`, `primary`, `secondary`, `tertiary`, `inverted`, `interactive-primary`, `interactive-secondary`

**Available overlays:** `interactive-hover`, `interactive-active`, `interactive-inverted-hover`, `interactive-inverted-active`

---

### Typography System

#### Font Families

| Font | Class | Usage |
|------|-------|-------|
| **Noi Grotesk** | `font-sans` (default) | Body text, headings, UI elements - the workhorse |
| **IBM Plex Mono** | `font-mono` | Code blocks, technical content, data displays |
| **Sans Plomb** | `font-accent` | Brand/marketing typography only - never in product UI |

#### Header Classes

Use these for all headings. Never use arbitrary `text-4xl font-bold`.

| Class | Size | Weight | Use For |
|-------|------|--------|---------|
| `text-header-xl` | 44px | 500 | Page titles, hero headings |
| `text-header-lg` | 32px | 500 | Section headers |
| `text-header-md` | 28px | 500 | Subsection headers |
| `text-header-sm` | 24px | 500 | Card titles, dialog titles |

#### Body Classes

| Class | Size | Weight | Use For |
|-------|------|--------|---------|
| `text-body-xl` | 19px | 400 | Large body text, introductions |
| `text-body-lg` | 17px | 400 | Default body text |
| `text-body-md` | 15px | 400 | Secondary text, descriptions |
| `text-body-sm` | 13px | 400 | Captions, metadata, helper text |

**Medium weight variants:** Add `-medium` suffix for 500 weight (`text-body-lg-medium`)

#### Typography Signature

- **Tight letter-spacing** on headers: -0.02em to -0.04em (built into classes)
- **Relaxed line-height** on body: 120-140% (built into classes)
- **Never use arbitrary sizes** like `text-[17px]` - use the system

---

### Spacing & Sizing

#### Component Size Scale

All interactive components follow the same size scale:

| Size | Height | Use For |
|------|--------|---------|
| `xs` | 28px (`h-7`) | Dense UIs, inline actions, compact tables |
| `sm` | 32px (`h-8`) | Secondary buttons, compact inputs |
| `md` | 40px (`h-10`) | **Default** - most buttons and inputs |
| `lg` | 48px (`h-12`) | Primary CTAs, prominent inputs, touch-friendly |

#### Border Radius

| Class | Pixels | Use For |
|-------|--------|---------|
| `rounded-sm` | 2px | Checkboxes |
| `rounded-md` | 6px | Tooltips, small elements |
| `rounded-lg` | 8px | **Default** - buttons, inputs, cards |
| `rounded-xl` | 12px | Dialogs, popovers, large containers |
| `rounded-full` | 9999px | Pills, avatars, circular icons |

---

### Shadows

Rosetta uses minimal shadows. Depth comes from background color differences, not dramatic shadows.

| Token | Use For |
|-------|---------|
| `shadow-light` | Light elevation, subtle lift |
| `shadow-card` | Cards and panels |
| `shadow-lg` | Dialogs, popovers, elevated overlays |

---

## 3. Component Catalog

### The Golden Rule

**MANDATORY: Use Rosetta components, never raw HTML for interactive elements.**

| Instead of... | Use... | Import from |
|---------------|--------|-------------|
| `<button>` | `<Button>` | `@/components/hai-ui` |
| `<input>` | `<Input>` | `@/components/hai-ui` |
| `<select>` | `<Select>` | `@/components/hai-ui` |
| `<textarea>` | `<TextArea>` | `@/components/hai-ui` |
| Custom modal | `<Dialog>` | `@/components/hai-ui` |
| Custom tooltip | `<Tooltip>` | `@/components/hai-ui` |
| Custom dropdown | `<DropdownMenu>` | `@/components/hai-ui` |
| `<a>` for internal links | `<Link>` | `@/components/hai-ui` |
| Icons from lucide-react | `<Icon name="...">` | `@/components/hai-ui` |

### Button

```tsx
import { Button } from '@/components/hai-ui';

// Variants
<Button variant="primary">Save</Button>           // Main CTA - dark bg
<Button variant="secondary">Cancel</Button>       // Alternative - light bg, bordered
<Button variant="tertiary">Learn more</Button>    // Low emphasis - no bg
<Button variant="primary-destructive">Delete</Button>    // Danger - red bg
<Button variant="secondary-destructive">Remove</Button>  // Danger alt - light bg, red text

// Sizes
<Button size="xs">Tiny</Button>
<Button size="sm">Small</Button>
<Button size="md">Medium (default)</Button>
<Button size="lg">Large</Button>

// With icon - ALWAYS use the icon prop
<Button icon="plus">Add item</Button>  // ✅ Correct
<Button><Icon name="plus" /> Add</Button>  // ❌ Wrong
```

**When to use which variant:**

| Variant | Use When |
|---------|----------|
| `primary` | Main action on page/modal (usually one per view) |
| `secondary` | Alternative actions, cancel buttons |
| `tertiary` | Low-emphasis actions, "learn more" links |
| `primary-destructive` | Irreversible destructive actions (delete account) |
| `secondary-destructive` | Reversible destructive actions (remove from list) |

### Dialog

```tsx
import {
  Dialog, DialogContent, DialogHeader,
  DialogTitle, DialogBody, DialogFooter
} from '@/components/hai-ui';

<Dialog open={isOpen} onOpenChange={setIsOpen}>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>Confirm action</DialogTitle>
    </DialogHeader>
    <DialogBody>
      <p className="text-body-md text-secondary-foreground">
        Are you sure you want to proceed?
      </p>
    </DialogBody>
    <DialogFooter>
      <Button variant="secondary" onClick={() => setIsOpen(false)}>
        Cancel
      </Button>
      <Button variant="primary" onClick={handleConfirm}>
        Confirm
      </Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

### Banner

```tsx
import { Banner } from '@/components/hai-ui';

<Banner variant="neutral">Information message</Banner>
<Banner variant="warning">Warning: check your input</Banner>
<Banner variant="negative">Error: something went wrong</Banner>
```

**Banner vs Toast:**
- **Banner** - Persistent, inline with content, user should address before proceeding
- **Toast** - Temporary, floating, acknowledgment of completed action

### Tag

```tsx
import { Tag } from '@/components/hai-ui';

<Tag variant="default">Default</Tag>
<Tag variant="success">Completed</Tag>
<Tag variant="warning">Pending</Tag>
<Tag variant="danger">Failed</Tag>
<Tag variant="info">In Progress</Tag>
<Tag variant="neutral">Draft</Tag>
```

### Icon

```tsx
import { Icon } from '@/components/hai-ui';

// Standard usage
<Icon name="plus" />
<Icon name="check" filled />  // Filled variant
<Icon name="spinner" />       // Auto-animates with spin

// ❌ NEVER import from lucide-react directly
import { Plus } from 'lucide-react';  // Wrong!
```

### Form Components

```tsx
import {
  Input, Label, TextArea, Select,
  SelectTrigger, SelectValue, SelectContent, SelectItem,
  Checkbox, Switch
} from '@/components/hai-ui';

// Input with label
<div className="space-y-2">
  <Label htmlFor="email">Email</Label>
  <Input id="email" type="email" placeholder="you@example.com" />
</div>

// Select
<Select>
  <SelectTrigger>
    <SelectValue placeholder="Choose option" />
  </SelectTrigger>
  <SelectContent>
    <SelectItem value="a">Option A</SelectItem>
    <SelectItem value="b">Option B</SelectItem>
  </SelectContent>
</Select>

// Checkbox with label
<div className="flex items-center gap-2">
  <Checkbox id="terms" />
  <Label htmlFor="terms">I agree to the terms</Label>
</div>

// Switch
<Switch checked={enabled} onCheckedChange={setEnabled} />
```

---

## 4. The Bold Factor (Signature Elements)

These elements define Rosetta and prevent generic output:

1. **Semantic colors only** — No arbitrary Tailwind colors anywhere. Every color has a name and purpose.

2. **Layered interaction states** — Hover/active use `bg-{base}-with-{overlay}` pattern, not color swaps.

3. **Lime accent in dark mode** — The distinctive `#d3fb52` lime appears for primary CTAs and interactive highlights.

4. **0.5px borders** — Subtle, refined borders using `border-fg-border`. Never heavy.

5. **Tight letter-spacing on headers** — -0.02em to -0.04em gives a polished, not loose, feel.

6. **8px radius default** — `rounded-lg` for most elements. Rounded but not bubbly.

7. **Component-first** — You don't style buttons, you use `<Button>`. The patterns are encoded.

8. **Theme-aware everything** — Light/dark mode via CSS variables, not conditional classes.

---

## 5. Effects & Animation

### Motion Philosophy

**Functional, not decorative.** Animations communicate state changes and provide feedback. They're never just for show.

### Transition Defaults

- Standard interactive: `transition-colors duration-150`
- Elevation changes: `transition-shadow duration-200`
- Transform changes: `transition-transform duration-200`

### Hover Effects

- **Buttons:** Background color shift via layered overlay, no lift
- **Cards:** Subtle shadow increase if interactive
- **Links:** Color shift to `interactive-info`

### Loading States

```tsx
// Spinner icon (auto-animates)
<Icon name="spinner" />

// Button loading state
<Button disabled>
  <Icon name="spinner" className="mr-2" />
  Saving...
</Button>

// Skeleton loading
<div className="animate-pulse">
  <div className="h-4 bg-secondary rounded w-3/4 mb-2" />
  <div className="h-4 bg-secondary rounded w-1/2" />
</div>
```

---

## 6. Responsive Strategy

### Breakpoint Philosophy

Mobile layouts maintain the Rosetta feel through consistent typography and spacing. Structure simplifies, but aesthetic remains.

### Touch Targets

- **Minimum 44px height** for all interactive elements on mobile
- Buttons expand to full width on small screens: `w-full sm:w-auto`

### Typography Scaling

- Headers reduce gracefully but maintain hierarchy
- Body text never goes below 13px (`text-body-sm`)

### Common Patterns

```tsx
// Responsive button group
<div className="flex flex-col sm:flex-row gap-3">
  <Button variant="secondary" className="w-full sm:w-auto">Cancel</Button>
  <Button variant="primary" className="w-full sm:w-auto">Save</Button>
</div>

// Responsive grid
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
  {items.map(item => <Card key={item.id} />)}
</div>
```

---

## 7. Accessibility & Best Practices

### Color Contrast

- All text meets WCAG AA minimum (4.5:1 for normal text)
- Interactive colors tested for sufficient contrast
- Don't rely on color alone to convey meaning

### Focus States

- Visible focus rings on all interactive elements
- `focus-visible:ring-2 focus-visible:ring-interactive-info`
- Never remove focus indicators

### Semantic HTML

- Use proper heading hierarchy (h1 → h2 → h3)
- Use `<button>` for actions, `<a>` for navigation
- Use native form elements with proper labels

### Motion

- Respect `prefers-reduced-motion` for animations
- No flashing or rapid movements
- Transitions are subtle (150-200ms)

---

## 8. Composition Patterns

### Destructive Action Confirmation

```tsx
<Dialog open={showDelete} onOpenChange={setShowDelete}>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>Delete item?</DialogTitle>
    </DialogHeader>
    <DialogBody>
      <p className="text-body-md text-secondary-foreground">
        This action cannot be undone. The item will be permanently removed.
      </p>
    </DialogBody>
    <DialogFooter>
      <Button variant="secondary" onClick={() => setShowDelete(false)}>
        Cancel
      </Button>
      <Button variant="primary-destructive" onClick={handleDelete}>
        Delete
      </Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

### Form with Validation Error

```tsx
<div className="space-y-4">
  <div className="space-y-2">
    <Label htmlFor="name">Name</Label>
    <Input
      id="name"
      value={name}
      onChange={e => setName(e.target.value)}
      aria-invalid={!!errors.name}
    />
    {errors.name && (
      <p className="text-body-sm text-interactive-negative">
        {errors.name}
      </p>
    )}
  </div>
  <Button type="submit" variant="primary">Submit</Button>
</div>
```

### Page Header

```tsx
<div className="flex items-center justify-between mb-6">
  <div>
    <h1 className="text-header-lg text-primary-foreground">Page Title</h1>
    <p className="text-body-md text-secondary-foreground mt-1">
      Brief description of the page
    </p>
  </div>
  <Button variant="primary" icon="plus">Add new</Button>
</div>
```

### Empty State

```tsx
<div className="flex flex-col items-center justify-center py-12 text-center">
  <Icon name="inbox" className="h-12 w-12 text-tertiary-foreground mb-4" />
  <h3 className="text-header-sm text-primary-foreground mb-2">No items yet</h3>
  <p className="text-body-md text-secondary-foreground mb-4 max-w-sm">
    Get started by creating your first item.
  </p>
  <Button variant="primary" icon="plus">Create item</Button>
</div>
```

---

## 9. Migration Recipes

### Custom Button → Rosetta Button

```tsx
// ❌ Before
<button
  className="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded"
  onClick={handleClick}
>
  Submit
</button>

// ✅ After
<Button variant="primary" onClick={handleClick}>
  Submit
</Button>
```

### Custom Modal → Rosetta Dialog

```tsx
// ❌ Before
<div className="fixed inset-0 bg-black/50 flex items-center justify-center">
  <div className="bg-white rounded-lg p-6 max-w-md">
    <h2 className="text-xl font-bold mb-4">Title</h2>
    <p>Content</p>
    <div className="mt-4 flex justify-end gap-2">
      <button onClick={onClose}>Cancel</button>
      <button onClick={onConfirm}>Confirm</button>
    </div>
  </div>
</div>

// ✅ After
<Dialog open={isOpen} onOpenChange={setIsOpen}>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>Title</DialogTitle>
    </DialogHeader>
    <DialogBody>Content</DialogBody>
    <DialogFooter>
      <Button variant="secondary" onClick={() => setIsOpen(false)}>Cancel</Button>
      <Button variant="primary" onClick={onConfirm}>Confirm</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

### Arbitrary Colors → Semantic Tokens

```tsx
// ❌ Before
<div className="bg-gray-100 text-gray-700 border border-gray-300">
<p className="text-red-500">Error message</p>
<button className="bg-green-500 text-white">Success</button>

// ✅ After
<div className="bg-secondary text-secondary-foreground border border-fg-border">
<p className="text-interactive-negative">Error message</p>
<Button variant="primary">Success</Button>
```

### Custom Form → Rosetta Form

```tsx
// ❌ Before
<label className="block text-sm font-medium text-gray-700">Email</label>
<input
  type="email"
  className="mt-1 block w-full border border-gray-300 rounded-md px-3 py-2"
/>

// ✅ After
<div className="space-y-2">
  <Label htmlFor="email">Email</Label>
  <Input id="email" type="email" />
</div>
```

---

## 10. Anti-Patterns (What NOT to Do)

### Color Anti-Patterns

```tsx
// ❌ Arbitrary Tailwind colors
<div className="bg-gray-100">
<p className="text-blue-600">
<button className="bg-green-500">

// ❌ Hardcoded hex values
<div style={{ backgroundColor: '#f5f5f5' }}>

// ❌ rgba without tokens
<div className="bg-[rgba(0,0,0,0.1)]">
```

### Component Anti-Patterns

```tsx
// ❌ Raw HTML when Rosetta component exists
<button onClick={...}>Click</button>
<input type="text" />
<select>...</select>

// ❌ Importing icons directly
import { Plus } from 'lucide-react';

// ❌ Custom modal implementation
<div className="fixed inset-0 ...">

// ❌ Icon as button child instead of prop
<Button><Icon name="plus" /> Add</Button>
```

### Layout Anti-Patterns

```tsx
// ❌ Fixed pixel values for responsive layouts
<div className="w-[847px]">

// ❌ Missing loading state
const MyComponent = () => {
  const { data } = useQuery(...);
  return <div>{data.title}</div>; // Crashes if data undefined
};

// ❌ Missing error state
// (always handle both loading AND error)
```

---

## 11. Verification Checklist

Before considering FE work complete, verify:

**Colors:**
- [ ] No `bg-gray-*`, `text-blue-*`, or other arbitrary colors
- [ ] Dark mode works (toggle theme, verify no color issues)

**Components:**
- [ ] All buttons use `<Button>`, not `<button>`
- [ ] All inputs use `<Input>`, not `<input>`
- [ ] All modals use `<Dialog>`
- [ ] Icons use `<Icon name="...">`, not lucide-react imports

**States:**
- [ ] Loading states exist for data-fetching components
- [ ] Error states exist for data-fetching components
- [ ] Focus states are visible (tab through the UI)

**Typography:**
- [ ] Headers use `text-header-*` classes
- [ ] Body text uses `text-body-*` classes
- [ ] No arbitrary text sizes

**Visual Verification:**
- [ ] Viewed in browser, not just in code
- [ ] Tested in both light and dark mode
- [ ] Tested on mobile viewport width
</design-system>
```

**Step 2: Verify file created**

Run: `head -50 .claude/rosetta-design-prompt.md`
Expected: First 50 lines showing `<role>` section

**Step 3: Commit**

```bash
git add .claude/rosetta-design-prompt.md
git commit -m "feat: add Rosetta Design Prompt v0

Follows designprompts.dev template structure with:
- Role preamble for FE development context
- Design philosophy and signature elements
- Complete token reference with usage guidance
- Component catalog with HAI-UI mappings
- Composition patterns and migration recipes
- Anti-patterns and verification checklist

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 2: Create Claude Skill for FE Development

**Files:**
- Create: `.claude/skills/rosetta-frontend/skill.md`

**Step 1: Create skills directory**

```bash
mkdir -p .claude/skills/rosetta-frontend
```

**Step 2: Create the skill file**

```markdown
# Rosetta Frontend Development

## Overview

Use this skill when building frontend components, pages, or features. Loads the Rosetta Design Prompt and enforces visual verification.

**Announce at start:** "Using Rosetta Frontend skill - will follow design system and verify visually."

## Before Writing Code

1. Read `.claude/rosetta-design-prompt.md` for design context

2. **Discover existing components via Storybook MCP:**
   ```
   mcp__storybook__list-all-components
   ```
   - Check if a component already exists for your use case
   - Review props/variants via `get-component-docs`

3. Check `src/components/hai-ui/` for implementation details

4. Identify which Rosetta components and patterns to use

## While Writing Code

Follow the Rosetta Design Prompt:
- Use semantic color tokens, never arbitrary colors
- Use HAI-UI components, never raw HTML for interactive elements
- Use `<Icon>` component, never direct lucide-react imports
- Use typography classes (`text-header-*`, `text-body-*`)
- Handle loading and error states

## After Writing Code

**MANDATORY: Visual Verification (Multi-MCP Workflow)**

Before claiming work is complete:

### Step 1: Verify in Storybook (if new/modified component)

```
# Get story URLs for the component
mcp__storybook__get-story-urls
component: Button

# Navigate to Storybook to verify component in isolation
mcp__chrome-devtools__navigate_page
url: http://localhost:6006/?path=/story/button--primary
```

### Step 2: Verify in App Context

1. **Navigate to the page**
   ```
   mcp__chrome-devtools__navigate_page
   url: http://localhost:3000/your-page
   ```

2. **Take a snapshot** to verify structure
   ```
   mcp__chrome-devtools__take_snapshot
   ```

3. **Verify against Rosetta checklist:**
   - [ ] Component renders without console errors
   - [ ] No arbitrary Tailwind colors visible
   - [ ] HAI-UI components used correctly
   - [ ] Loading state works (if applicable)
   - [ ] Error state works (if applicable)

4. **Test dark mode**
   ```
   mcp__chrome-devtools__evaluate_script
   function: () => document.documentElement.setAttribute('data-theme', 'dark')
   ```

5. **Take screenshot** if needed
   ```
   mcp__chrome-devtools__take_screenshot
   ```

## Quick Reference

### Color Classes
- Backgrounds: `bg-surface`, `bg-primary`, `bg-secondary`, `bg-tertiary`, `bg-inverted`
- Text: `text-primary-foreground`, `text-secondary-foreground`, `text-tertiary-foreground`
- Interactive: `bg-interactive-primary`, `bg-interactive-negative`, `bg-interactive-positive`
- Borders: `border-fg-border`
- Hover: `bg-{base}-with-interactive-hover`

### Typography Classes
- Headers: `text-header-xl`, `text-header-lg`, `text-header-md`, `text-header-sm`
- Body: `text-body-xl`, `text-body-lg`, `text-body-md`, `text-body-sm`
- Add `-medium` for 500 weight

### Component Imports
```tsx
import {
  Button, Dialog, Banner, Tag, Icon,
  Input, Label, Select, Checkbox, Switch
} from '@/components/hai-ui';
```
```

**Step 3: Commit**

```bash
git add .claude/skills/rosetta-frontend/
git commit -m "feat: add Rosetta Frontend Claude skill

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 3: Create Enhanced Cursor Rule

**Files:**
- Create: `.cursor/rules/rosetta-design-system.mdc`

**Step 1: Create the Cursor rule**

```markdown
---
description: Rosetta Design System
globs: ["src/**/*.tsx", "src/**/*.ts"]
alwaysApply: true
---

## Rosetta Design System

**MANDATORY:** All frontend code must follow the Rosetta Design Prompt.

Full design prompt: `.claude/rosetta-design-prompt.md`

### Quick Reference

#### Colors (NEVER use arbitrary Tailwind colors)

**Backgrounds:** `bg-surface`, `bg-primary`, `bg-secondary`, `bg-tertiary`, `bg-inverted`
**Text:** `text-primary-foreground`, `text-secondary-foreground`, `text-tertiary-foreground`
**Interactive:** `bg-interactive-primary`, `bg-interactive-negative`, `bg-interactive-positive`
**Status:** `bg-status-positive`, `bg-status-warning`, `bg-status-negative`, `bg-status-info`
**Borders:** `border-fg-border` (the only border color)
**Hover/Active:** `bg-{base}-with-interactive-hover`, `bg-{base}-with-interactive-active`

#### Components (NEVER use raw HTML)

| Raw HTML | Rosetta Component |
|----------|-------------------|
| `<button>` | `<Button>` |
| `<input>` | `<Input>` |
| `<select>` | `<Select>` |
| `<textarea>` | `<TextArea>` |
| Modal/dialog | `<Dialog>` |
| Tooltip | `<Tooltip>` |
| Dropdown | `<DropdownMenu>` |

Import from: `@/components/hai-ui`

#### Icons

Always use `<Icon name="..." />` from `@/components/hai-ui`.
Never import from `lucide-react` directly.

#### Typography

Headers: `text-header-xl`, `text-header-lg`, `text-header-md`, `text-header-sm`
Body: `text-body-xl`, `text-body-lg`, `text-body-md`, `text-body-sm`
Add `-medium` for 500 weight.

### Before Completing Work

Verify:
- [ ] No `bg-gray-*`, `text-blue-*`, or other arbitrary colors
- [ ] No raw `<button>`, `<input>`, `<select>` elements
- [ ] No direct lucide-react imports
- [ ] Loading and error states handled
- [ ] Works in dark mode
```

**Step 2: Commit**

```bash
git add .cursor/rules/rosetta-design-system.mdc
git commit -m "feat: add Rosetta Design System Cursor rule

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 4: Create Visual Verification Playbook

**Files:**
- Create: `.claude/playbooks/visual-verification.md`

**Step 1: Create playbooks directory and file**

```bash
mkdir -p .claude/playbooks
```

```markdown
# Visual Verification Playbook

How to use Storybook MCP + Chrome DevTools MCP for AI-assisted frontend development.

## Architecture: Multi-MCP Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                    AI Coding Assistant                       │
│              (Claude Code / Cursor)                         │
└────────────────────────┬────────────────────────────────────┘
                         │ MCP Protocol
         ┌───────────────┴───────────────┐
         ▼                               ▼
┌─────────────────┐            ┌─────────────────┐
│   Storybook     │            │  Chrome DevTools│
│      MCP        │            │      MCP        │
│                 │            │                 │
│ - list-all-     │            │ - navigate      │
│   components    │            │ - screenshot    │
│ - get-docs      │            │ - snapshot      │
│ - get-story-urls│            │ - interact      │
└────────┬────────┘            └────────┬────────┘
         │                              │
         ▼                              ▼
┌─────────────────┐            ┌─────────────────┐
│    Storybook    │            │   Dev Server    │
│  localhost:6006 │            │  localhost:3000 │
└─────────────────┘            └─────────────────┘
```

**Storybook MCP** → Component discovery ("What exists?")
**Chrome DevTools MCP** → Visual verification ("Does it look right?")

## Prerequisites

### Setup MCP Servers

```bash
# 1. Install Storybook MCP addon (one-time)
npx storybook@latest add @storybook/addon-mcp

# 2. Configure Claude Code
claude mcp add storybook-mcp --transport http http://localhost:6006/mcp --scope project
claude mcp add chrome-devtools --scope user npx chrome-devtools-mcp@latest
```

### Running Services

- Storybook: `yarn storybook` (default: http://localhost:6006)
- Dev server: `yarn dev` (default: http://localhost:3000)
- Chrome browser open

## Workflow 1: Discover Components Before Writing Code

Before implementing, check what already exists:

### List all components
```
mcp__storybook__list-all-components
```

Returns catalog of all HAI-UI components with their stories.

### Get component documentation
```
mcp__storybook__get-component-docs
component: Button
```

Returns props, variants, usage examples.

### Get story URLs for visual reference
```
mcp__storybook__get-story-urls
component: Button
```

Navigate to these to see the component in isolation.

## Workflow 2: Verify New/Modified Components in Storybook

After writing component code:

### Navigate to the story
```
mcp__chrome-devtools__navigate_page
url: http://localhost:6006/?path=/story/hai-ui-button--primary
```

### Take screenshot of component
```
mcp__chrome-devtools__take_screenshot
```

### Test different variants
Navigate to each story variant to verify all states render correctly.

## Workflow 3: Verify in Application Context

After component works in isolation:

### 1. Navigate to the Page

```
mcp__chrome-devtools__navigate_page
url: http://localhost:3000/your-page
```

### 2. Take a Snapshot

See the page structure and content:

```
mcp__chrome-devtools__take_snapshot
```

Returns an accessibility tree - useful for verifying:
- Component hierarchy
- Text content
- Interactive elements

### 3. Take a Screenshot

Get a visual of the rendered page:

```
mcp__chrome-devtools__take_screenshot
```

### 4. Interact with Elements

Click a button:
```
mcp__chrome-devtools__click
uid: <element-uid-from-snapshot>
```

Fill an input:
```
mcp__chrome-devtools__fill
uid: <element-uid-from-snapshot>
value: "test input"
```

### 5. Check Console for Errors

```
mcp__chrome-devtools__list_console_messages
```

Look for:
- React errors
- TypeScript runtime errors
- Network failures

### 6. Test Dark Mode

```
mcp__chrome-devtools__evaluate_script
function: () => {
  document.documentElement.setAttribute('data-theme', 'dark');
}
```

Then take another screenshot to verify.

## Closed-Loop Development Pattern

The complete AI-assisted FE development loop:

```
1. Discover (Storybook MCP)
   └→ "What components exist?"
   └→ "What are the props/variants?"

2. Write Code
   └→ Use existing components
   └→ Follow Rosetta patterns

3. Verify in Storybook (Chrome DevTools MCP)
   └→ Component renders correctly in isolation
   └→ All variants work

4. Verify in App (Chrome DevTools MCP)
   └→ Works in real page context
   └→ Dark mode works
   └→ No console errors

5. Iterate
   └→ If issues found, fix and re-verify
   └→ Don't claim complete until verified
```

## Rosetta Verification Checklist

After visual verification, confirm:

- [ ] Page loads without console errors
- [ ] UI matches Rosetta aesthetic (no arbitrary colors)
- [ ] Interactive elements work
- [ ] Dark mode renders correctly
- [ ] Loading states display properly
- [ ] Error states display properly (if testable)

## Common Issues

### Storybook MCP not connecting
- Verify Storybook is running: `yarn storybook`
- Check MCP endpoint: `curl http://localhost:6006/mcp`
- Ensure addon is installed

### Page not loading
- Check dev server: `yarn dev`
- Check URL is correct
- Check for build errors

### Elements not clickable
- Element might be covered
- Element might be disabled
- Take snapshot to see current state

### Dark mode not working
- Verify `data-theme` attribute is set
- Check if component uses semantic colors (not arbitrary)
```

**Step 2: Commit**

```bash
git add .claude/playbooks/
git commit -m "feat: add visual verification playbook

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 5: Test the Demo

**Files:**
- None (verification only)

**Step 1: Verify all files exist**

```bash
ls -la .claude/rosetta-design-prompt.md .claude/skills/rosetta-frontend/skill.md .cursor/rules/rosetta-design-system.mdc .claude/playbooks/visual-verification.md
```

Expected: All 4 files exist

**Step 2: Demo test**

Test prompt:
> "Using the Rosetta design system, create a confirmation dialog that asks 'Delete this item?' with Cancel and Delete buttons."

Expected behavior:
- Claude reads the design prompt
- Uses `<Dialog>`, `<Button>` components
- Uses correct variants (`secondary` for Cancel, `primary-destructive` for Delete)
- Uses semantic colors
- Offers to verify visually

**Step 3: Final commit**

```bash
git add -A
git commit -m "feat: complete Rosetta Design Prompt v0 demo

Includes:
- Design prompt following designprompts.dev template
- Claude skill for FE development with visual verification
- Cursor rule for quick reference
- Visual verification playbook

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Summary

### Phase 0: POC / Show & Tell (~1-2 hours)

| Step | Action |
|------|--------|
| Setup | Copy rough prompt, configure MCPs, start services |
| Demo 1 | Build new feature (empty state), verify visually |
| Demo 2 | Migrate existing component, verify dark mode works |
| Output | Live/recorded demo for show & tell |

### Phase 1: Productionize

| Deliverable | File | Purpose |
|-------------|------|---------|
| Design Prompt | `.claude/rosetta-design-prompt.md` | Full `<role>` + `<design-system>` structure |
| Claude Skill | `.claude/skills/rosetta-frontend/skill.md` | Component discovery + visual verification workflow |
| Cursor Rule | `.cursor/rules/rosetta-design-system.mdc` | Quick reference for code completion |
| Playbook | `.claude/playbooks/visual-verification.md` | Multi-MCP guide (Storybook + Chrome DevTools) |

**Key improvements from template:**
- Philosophy section with emotional keywords and sensory description
- "What Rosetta is NOT" anti-patterns
- "The Bold Factor" signature elements
- Usage guidance in token tables (not just values)
- Complete composition patterns
- Migration recipes with before/after code

**MCP Integration:**
- Storybook MCP for component discovery before writing code
- Chrome DevTools MCP for visual verification after writing code
- Closed-loop workflow: Discover → Write → Verify → Iterate

**Setup Commands:**
```bash
# Chrome DevTools MCP (required)
claude mcp add chrome-devtools --scope user npx chrome-devtools-mcp@latest

# Storybook MCP (optional, if Storybook addon installed)
npx storybook@latest add @storybook/addon-mcp
claude mcp add storybook-mcp --transport http http://localhost:6006/mcp --scope project
```
