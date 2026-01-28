# Compiled Design Context Architecture

**Model:** Source repo with design specs → compiled → published as package → consumed by apps and AI

---

## The Shape

```
┌──────────────────────────────────────────────────────────────────┐
│                     SOURCE REPO                                   │
│               design-system-source/                               │
│                                                                   │
│   tokens/     components/     patterns/     behaviors/            │
│      ↓            ↓              ↓             ↓                 │
│   YAML specs (human-authored, version-controlled)                │
└──────────────────────────────────┬───────────────────────────────┘
                                   │
                                   ▼ compile
┌──────────────────────────────────────────────────────────────────┐
│                     BUILD STEP                                    │
│                                                                   │
│   - Validate schemas                                              │
│   - Resolve references/inheritance                                │
│   - Flatten into distributable formats                           │
│   - Generate platform-specific outputs                           │
└──────────────────────────────────┬───────────────────────────────┘
                                   │
                                   ▼ publish
┌──────────────────────────────────────────────────────────────────┐
│                     PUBLISHED PACKAGE                             │
│               @handshake/design-context                           │
│                                                                   │
│   dist/                                                           │
│   ├── context.json        # Bundled, flattened specs             │
│   ├── context.md          # AI-optimized markdown                │
│   ├── tokens.css          # CSS custom properties                │
│   ├── tokens.ts           # TypeScript constants                 │
│   ├── tokens.swift        # Swift constants                      │
│   └── figma-tokens.json   # Figma-compatible format              │
└──────────────────────────────────┬───────────────────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
            ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
            │   Web App   │ │ Mobile App  │ │   Figma     │
            │             │ │             │ │             │
            │ imports     │ │ imports     │ │ imports     │
            │ tokens.css  │ │ tokens.swift│ │ figma-json  │
            │             │ │             │ │             │
            │ AI reads    │ │ AI reads    │ │             │
            │ context.md  │ │ context.md  │ │             │
            └─────────────┘ └─────────────┘ └─────────────┘
```

---

## Source Repo Structure

```
design-system-source/
├── package.json              # Build scripts, versioning
├── rosetta.config.yaml       # Build configuration
│
├── tokens/                   # Layer 1: Design tokens
│   ├── colors.yaml
│   ├── typography.yaml
│   ├── spacing.yaml
│   ├── radii.yaml
│   └── shadows.yaml
│
├── components/               # Layer 2: Component specs
│   ├── _base.yaml            # Shared component properties
│   ├── button.yaml
│   ├── card.yaml
│   ├── input.yaml
│   └── ...
│
├── patterns/                 # Layer 3: Composition patterns
│   ├── list-page.yaml
│   ├── detail-page.yaml
│   ├── data-table.yaml
│   └── ...
│
├── behaviors/                # Layer 4: Interaction specs
│   ├── form-submission.yaml
│   ├── modal-confirmation.yaml
│   └── ...
│
├── content/                  # Entity schemas (optional)
│   ├── job.yaml
│   ├── employer.yaml
│   └── ...
│
├── prose/                    # Human-readable context
│   ├── philosophy.md         # Design philosophy
│   ├── voice-and-tone.md     # Writing guidelines
│   └── anti-patterns.md      # What NOT to do
│
└── build/                    # Build scripts
    ├── compile.ts
    ├── validate.ts
    └── templates/
        ├── context.md.hbs
        ├── tokens.css.hbs
        └── ...
```

---

## Build Step

### What "Compile" Means

```typescript
// build/compile.ts

async function compile() {
  // 1. Load all YAML specs
  const tokens = await loadYaml('tokens/**/*.yaml');
  const components = await loadYaml('components/**/*.yaml');
  const patterns = await loadYaml('patterns/**/*.yaml');
  const behaviors = await loadYaml('behaviors/**/*.yaml');
  const prose = await loadMarkdown('prose/**/*.md');

  // 2. Validate against schemas
  validateTokens(tokens);
  validateComponents(components);
  // ... errors fail the build

  // 3. Resolve references
  // e.g., button.yaml references colors.yaml tokens
  const resolved = resolveReferences({ tokens, components, patterns, behaviors });

  // 4. Generate outputs
  await generateJSON(resolved, 'dist/context.json');
  await generateMarkdown(resolved, prose, 'dist/context.md');
  await generateCSS(tokens, 'dist/tokens.css');
  await generateTypeScript(tokens, 'dist/tokens.ts');
  await generateSwift(tokens, 'dist/tokens.swift');
  await generateFigmaTokens(tokens, 'dist/figma-tokens.json');
}
```

### Output: context.json

Flattened, resolved, ready for programmatic use:

```json
{
  "version": "2.4.0",
  "tokens": {
    "colors": {
      "background": {
        "primary": "#FFFFFF",
        "secondary": "#F6F6F6"
      },
      "interactive": {
        "primary": "#04191B"
      }
    },
    "typography": { ... },
    "spacing": { ... }
  },
  "components": {
    "button": {
      "variants": ["primary", "secondary", "tertiary", "destructive"],
      "sizes": ["sm", "md", "lg"],
      "props": [ ... ],
      "styling": { ... }
    }
  },
  "patterns": {
    "list-page": {
      "structure": [ ... ],
      "slots": [ ... ]
    }
  },
  "behaviors": { ... }
}
```

### Output: context.md

AI-optimized markdown — this is what Claude/Cursor reads:

```markdown
# Handshake Design System (Rosetta) v2.4.0

## Design Philosophy

Professional clarity with human warmth...

[prose/philosophy.md content]

## Tokens

### Colors

| Token | Value | Usage |
|-------|-------|-------|
| `background.primary` | #FFFFFF | Primary canvas |
| `interactive.primary` | #04191B | Primary buttons |
...

## Components

### Button

**Variants:** primary, secondary, tertiary, destructive
**Sizes:** sm (36px), md (48px), lg (60px)

**Usage:**
- Primary: Main action on page (one per view)
- Secondary: Alternative actions
...

**Props:**
| Prop | Type | Default | Description |
...

## Patterns

### List Page

Use for searchable, filterable lists of items.

**Structure:**
- Header: title + actions
- Toolbar: search + filters
- Main: grid of cards
- Footer: pagination

**When to use:**
- Job listings
- Employer search
- Saved items

**Handshake preferences:**
- Filters in left sidebar on desktop, drawer on mobile
- Always include empty state
- Use skeleton loading, not spinners
...

## Anti-Patterns

[prose/anti-patterns.md content]
```

### Output: tokens.css

```css
/* Auto-generated from design-system-source v2.4.0 */

:root {
  /* Colors - Background */
  --rosetta-bg-primary: #FFFFFF;
  --rosetta-bg-secondary: #F6F6F6;
  --rosetta-bg-surface: #FCFCFD;
  
  /* Colors - Interactive */
  --rosetta-interactive-primary: #04191B;
  --rosetta-interactive-hover: rgba(18, 18, 18, 0.06);
  
  /* Typography */
  --rosetta-font-sans: "Noi Grotesk", system-ui, sans-serif;
  --rosetta-font-size-body-md: 15px;
  --rosetta-line-height-body-md: 1.2;
  
  /* Spacing */
  --rosetta-spacing-sm: 8px;
  --rosetta-spacing-md: 16px;
  --rosetta-spacing-lg: 24px;
  
  /* ... */
}
```

### Output: tokens.ts

```typescript
// Auto-generated from design-system-source v2.4.0

export const colors = {
  background: {
    primary: '#FFFFFF',
    secondary: '#F6F6F6',
  },
  interactive: {
    primary: '#04191B',
  },
} as const;

export const spacing = {
  sm: 8,
  md: 16,
  lg: 24,
} as const;

// Type-safe token access
export type ColorToken = keyof typeof colors.background | keyof typeof colors.interactive;
```

---

## Publish Step

```bash
# In CI after merge to main
yarn build                    # Compile specs to dist/
npm version patch             # Bump version
npm publish                   # Publish @handshake/design-context
```

Or publish to:
- GitHub Packages (private)
- Internal Artifactory
- CDN for non-npm consumers

---

## Consumption

### Web App (React)

```bash
npm install @handshake/design-context
```

**Use tokens:**
```tsx
// Import CSS tokens
import '@handshake/design-context/dist/tokens.css';

// Or TypeScript tokens
import { colors, spacing } from '@handshake/design-context';
```

**AI uses context:**
```
# .cursor/rules/rosetta.mdc or CLAUDE.md

Read the Rosetta design system context:
cat node_modules/@handshake/design-context/dist/context.md
```

Or reference directly:
```markdown
<!-- CLAUDE.md -->
## Design System

Import and follow the Rosetta design system:

@import node_modules/@handshake/design-context/dist/context.md
```

### Mobile App (iOS)

```swift
// Package.swift dependency
.package(url: "https://github.com/handshake/design-context-swift", from: "2.4.0")

// Generated from same source
import RosettaTokens

let buttonColor = RosettaTokens.colors.interactive.primary
```

### Figma

Figma Tokens plugin imports `figma-tokens.json`:
- Syncs colors, typography, spacing as Figma styles
- Updates when package version bumps

---

## End-to-End: Building a Feature

### 1. Developer starts

```bash
# Design context already installed
npm ls @handshake/design-context
# @handshake/design-context@2.4.0
```

### 2. AI has context

Developer opens Claude/Cursor. AI instructions include:

```markdown
When building UI, follow the Rosetta design system.
Context is at: node_modules/@handshake/design-context/dist/context.md

Before writing code:
1. Read context.md for patterns and components
2. Use tokens from tokens.css
3. Follow documented patterns
```

### 3. Developer prompts

> "Build a saved jobs page"

### 4. AI reads context.md

AI sees:
- `list-page` pattern exists
- Components available: Card, Button, SearchInput, Pagination
- Handshake preferences: filters in sidebar, skeleton loading, empty states required
- Token values for styling

### 5. AI generates code

Code matches Handshake conventions because it's reading the published context, not hallucinating.

### 6. Developer refines

- Wires up API
- Adds business logic
- Tweaks as needed

---

## Versioning

```
design-system-source repo:

v2.4.0 - Added new status colors
v2.3.1 - Fixed button hover states  
v2.3.0 - Added data-table pattern
v2.2.0 - New form validation behaviors
```

Consuming apps pin versions:
```json
{
  "dependencies": {
    "@handshake/design-context": "^2.4.0"
  }
}
```

Breaking changes = major version bump.

---

## CI/CD Flow

```yaml
# .github/workflows/publish.yml

name: Publish Design Context

on:
  push:
    branches: [main]

jobs:
  build-and-publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install
        run: npm ci
        
      - name: Validate
        run: npm run validate  # Schema validation
        
      - name: Build
        run: npm run build     # Compile to dist/
        
      - name: Test
        run: npm test          # Snapshot tests for outputs
        
      - name: Publish
        run: npm publish
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## What This Enables

| Capability | How |
|------------|-----|
| **Single source of truth** | All specs in one repo |
| **Multi-platform** | Same source → CSS, Swift, Figma, etc. |
| **Versioned** | Semver, breaking change control |
| **AI-ready** | context.md optimized for LLM consumption |
| **Type-safe** | Generated TypeScript types |
| **Diffable** | YAML specs = meaningful PRs |
| **Testable** | Validate schemas in CI |
| **Decoupled** | Source repo ≠ consuming apps |

---

## Open Questions

1. **Who authors specs?** Designers? Devs? Both via PR?

2. **How granular?** Just tokens? Full component specs? Patterns?

3. **context.md size** — How do we keep it under token limits? Chunking?

4. **Breaking changes** — What constitutes a breaking change in design context?

5. **Sync with code** — How do we ensure consuming apps actually follow the context?

---

## Next Steps

1. **Start minimal** — Tokens only, prove the compile→publish→consume loop
2. **Add components** — Abstract component specs
3. **Add patterns** — Composition documentation
4. **Add prose** — Philosophy, anti-patterns
5. **Test with AI** — Does context.md improve generation quality?
