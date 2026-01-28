# Platform-Agnostic Design Architecture

**Vision:** Machine-readable design artifacts that serve as the single source of truth, generating any output — web, mobile, Figma, docs, emails.

---

## The Key Insight

Current approach mixes **WHAT** (design intent) with **HOW** (implementation):

```
rosetta.md contains:
- "Primary button uses teal #04191B" ← WHAT (intent)
- "<Button variant='primary'>" ← HOW (React-specific)
```

For true platform-agnostic design, we need to separate these:

```
Design Spec (WHAT)           Platform Renderers (HOW)
─────────────────            ────────────────────────
tokens.yaml          →       react-tailwind/
components.yaml      →       ios-swiftui/
patterns.yaml        →       android-compose/
layouts.yaml         →       figma/
behaviors.yaml       →       email/
content.yaml         →       docs/
```

**The spec is the truth. Renderers are implementations.**

---

## Revised Layer Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 0: SEMANTIC CONTENT                        │
│  What information exists. Data shapes. Content model.               │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 1: DESIGN TOKENS                           │
│  Colors, typography, spacing, shadows — platform-agnostic values    │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 2: COMPONENTS                              │
│  Abstract component definitions — props, variants, behaviors        │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 3: PATTERNS                                │
│  How components compose — abstract layouts, compositions            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 4: BEHAVIORS                               │
│  Interactions, flows, state machines — platform-agnostic logic      │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 5: PLATFORM RENDERERS                      │
│  Platform-specific implementation rules and mappings                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Layer 0: Semantic Content

**Purpose:** Define what information exists, independent of how it's displayed.

**Format:** YAML/JSON schema

```yaml
# content/job.yaml
entity: Job
description: A job posting from an employer

fields:
  - name: title
    type: string
    required: true
    maxLength: 100
    display: headline

  - name: company
    type: reference
    entity: Employer
    display: secondary

  - name: location
    type: Location
    display: metadata

  - name: salary
    type: SalaryRange
    display: metadata
    optional: true

  - name: description
    type: richtext
    display: body

  - name: requirements
    type: list<string>
    display: list

  - name: postedAt
    type: datetime
    display: timestamp

  - name: status
    type: enum
    values: [active, closed, draft]
    display: badge
```

**Why this matters:**
- Same content model → job card on web, job cell on mobile, job block in email
- AI knows what data exists without looking at code
- Enables consistent data display across all platforms

---

## Layer 1: Design Tokens

**Purpose:** Raw design values — no platform assumptions.

**Format:** YAML with semantic naming

```yaml
# tokens/colors.yaml
colors:
  # Semantic backgrounds
  background:
    primary: "#FFFFFF"
    secondary: "#F6F6F6"
    surface: "#FCFCFD"
    inverted: "#121212"
    
  # Semantic foregrounds
  foreground:
    primary: "#121212"
    secondary: "rgba(18, 18, 18, 0.7)"
    disabled: "rgba(18, 18, 18, 0.4)"
    inverted: "#FFFFFF"
    
  # Interactive
  interactive:
    primary: "#04191B"
    hover: "rgba(18, 18, 18, 0.06)"
    pressed: "rgba(18, 18, 18, 0.12)"
    focus: "#003BCC"
    
  # Status
  status:
    info:
      background: "#CCDBFF"
      foreground: "#003BCC"
    positive:
      background: "#A5E09F"
      foreground: "#327D0F"
    warning:
      background: "#FFD999"
      foreground: "#A36700"
    negative:
      background: "#FFC2C4"
      foreground: "#BB3643"
```

```yaml
# tokens/typography.yaml
typography:
  families:
    sans: "Noi Grotesk"
    brand: "Sansplomb"
    mono: "IBM Plex Mono"
    
  scale:
    heading-xl:
      size: 44
      weight: 500
      lineHeight: 1.0
      letterSpacing: -0.04
      
    heading-lg:
      size: 32
      weight: 500
      lineHeight: 1.2
      letterSpacing: -0.03
      
    body-lg:
      size: 17
      weight: 400
      lineHeight: 1.4
      letterSpacing: -0.02
      
    body-md:
      size: 15
      weight: 400
      lineHeight: 1.2
      letterSpacing: 0
```

```yaml
# tokens/spacing.yaml
spacing:
  base: 4  # px
  scale:
    xs: 4    # 1x
    sm: 8    # 2x
    md: 16   # 4x
    lg: 24   # 6x
    xl: 32   # 8x
    xxl: 48  # 12x

# tokens/radii.yaml
radii:
  none: 0
  sm: 4
  md: 8      # default for buttons, inputs
  lg: 12    # cards
  xl: 16    # large cards, modals
  full: 9999 # pills

# tokens/shadows.yaml
shadows:
  elevation-1: "0 1px 2px rgba(0,0,0,0.08)"
  elevation-2: "0 2px 20px rgba(0,0,0,0.04), 0 1px 2px rgba(0,0,0,0.08)"
  elevation-3: "0 6px 30px rgba(0,0,0,0.08), 0 1px 3px rgba(0,0,0,0.08)"
```

---

## Layer 2: Components

**Purpose:** Abstract component definitions — what they do, not how they're built.

**Format:** YAML with props, variants, states

```yaml
# components/button.yaml
component: Button
description: Primary interactive element for actions

props:
  - name: label
    type: string
    required: true
    
  - name: variant
    type: enum
    values: [primary, secondary, tertiary, destructive]
    default: primary
    
  - name: size
    type: enum
    values: [sm, md, lg]
    default: md
    
  - name: icon
    type: Icon
    position: [leading, trailing, only]
    optional: true
    
  - name: disabled
    type: boolean
    default: false
    
  - name: loading
    type: boolean
    default: false

states:
  - default
  - hover
  - pressed
  - focused
  - disabled
  - loading

sizing:
  sm: { height: 36, paddingX: 12, fontSize: body-sm }
  md: { height: 48, paddingX: 16, fontSize: body-md }
  lg: { height: 60, paddingX: 20, fontSize: body-lg }

styling:
  primary:
    default:
      background: interactive.primary
      foreground: foreground.inverted
      border: none
      radius: radii.md
    hover:
      background: "#062E32"  # or could reference a token
    pressed:
      background: "#073D42"
    disabled:
      background: background.disabled
      foreground: foreground.disabled
      
  secondary:
    default:
      background: background.primary
      foreground: foreground.primary
      border: { width: 1, color: foreground.border }
      radius: radii.md
    hover:
      background: background.tertiary

accessibility:
  role: button
  focusVisible: true
  focusRing: { width: 2, color: interactive.focus, offset: 2 }
  minimumTouchTarget: 44
```

```yaml
# components/card.yaml
component: Card
description: Container for grouped content

props:
  - name: variant
    type: enum
    values: [default, elevated, interactive]
    default: default
    
  - name: padding
    type: enum
    values: [none, sm, md, lg]
    default: md

slots:
  - name: header
    optional: true
  - name: body
    required: true
  - name: footer
    optional: true

styling:
  default:
    background: background.primary
    border: { width: 1, color: "#E2E5F3" }
    radius: radii.lg
    shadow: elevation-2
    
  interactive:
    extends: default
    hover:
      shadow: elevation-3
      cursor: pointer

accessibility:
  role: article  # or region with label
  interactive:
    role: button  # when clickable
```

---

## Layer 3: Patterns

**Purpose:** How components compose into higher-level UI structures.

**Format:** YAML describing structure, not code

```yaml
# patterns/list-page.yaml
pattern: ListPage
description: Page showing a searchable, filterable list of items

structure:
  - region: header
    contains:
      - element: title
        component: Heading
        level: 1
      - element: actions
        component: ButtonGroup
        position: trailing
        
  - region: toolbar
    contains:
      - element: search
        component: SearchInput
        width: flexible
      - element: filters
        component: FilterGroup
        collapsible: true
        
  - region: main
    layout: 
      type: grid
      columns: { sm: 1, md: 2, lg: 3 }
      gap: spacing.lg
    contains:
      - element: items
        component: Card
        repeats: true
        emptyState: EmptyState
        loadingState: SkeletonGrid
        
  - region: footer
    contains:
      - element: pagination
        component: Pagination
        position: center

responsive:
  mobile:
    - toolbar.filters: drawer
    - main.layout.columns: 1
  tablet:
    - main.layout.columns: 2
```

```yaml
# patterns/data-table.yaml
pattern: DataTable
description: Tabular data with sorting, filtering, and bulk actions

structure:
  - region: toolbar
    contains:
      - element: search
        component: SearchInput
        optional: true
      - element: filters
        component: FilterDropdown
        multiple: true
      - element: bulkActions
        component: ButtonGroup
        showWhen: hasSelection
        
  - region: table
    component: Table
    features:
      - sorting
      - selection
      - resizableColumns
      - stickyHeader
    columns:
      - type: checkbox
        width: 48
        sticky: left
      - type: data
        repeats: true
      - type: actions
        width: auto
        sticky: right
        
  - region: footer
    contains:
      - element: selectedCount
        showWhen: hasSelection
      - element: pagination
        component: Pagination

states:
  empty:
    component: EmptyState
    message: "No results found"
    action: clearFilters
    
  loading:
    component: TableSkeleton
    rows: 5
    
  error:
    component: ErrorState
    retry: true
```

---

## Layer 4: Behaviors

**Purpose:** Interactions and flows — how things behave, platform-agnostically.

**Format:** State machines, flow definitions

```yaml
# behaviors/form-submission.yaml
behavior: FormSubmission
description: Standard form submit flow

states:
  - idle
  - validating
  - submitting
  - success
  - error

transitions:
  idle:
    submit: validating
    
  validating:
    valid: submitting
    invalid: idle  # with errors displayed
    
  submitting:
    success: success
    failure: error
    
  success:
    reset: idle
    navigate: [destination]
    
  error:
    retry: submitting
    dismiss: idle

feedback:
  validating:
    button: { loading: true, disabled: true }
    
  submitting:
    button: { loading: true, disabled: true }
    overlay: optional  # prevent double-submit
    
  success:
    toast: { variant: positive, message: "[successMessage]" }
    
  error:
    banner: { variant: negative, message: "[errorMessage]" }
    # or inline errors on fields
```

```yaml
# behaviors/modal-confirmation.yaml
behavior: ModalConfirmation
description: Confirm before destructive action

trigger: userAction

flow:
  1. Show modal:
     component: Dialog
     title: "[confirmTitle]"
     body: "[confirmMessage]"
     actions:
       - label: Cancel
         variant: secondary
         action: dismiss
       - label: "[confirmAction]"
         variant: destructive
         action: confirm

  2. On confirm:
     state: loading
     execute: "[action]"
     
  3. On success:
     dismiss: modal
     feedback: toast.positive
     
  4. On error:
     feedback: banner.negative
     state: idle  # allow retry
```

```yaml
# behaviors/infinite-scroll.yaml
behavior: InfiniteScroll
description: Load more content as user scrolls

config:
  threshold: 200  # px from bottom
  pageSize: 20

states:
  - idle
  - loading
  - loadingMore
  - complete
  - error

triggers:
  scrollNearBottom:
    when: idle
    action: loadMore
    nextState: loadingMore
    
  loadSuccess:
    when: loadingMore
    hasMore: idle
    noMore: complete
    
  loadError:
    when: loadingMore
    nextState: error
    retryable: true

ui:
  loadingMore:
    component: Spinner
    position: bottom
    
  complete:
    component: EndOfList
    message: "You've seen it all"
    
  error:
    component: RetryButton
```

---

## Layer 5: Platform Renderers

**Purpose:** Platform-specific rules for turning abstract specs into implementations.

### React + Tailwind Renderer

```yaml
# renderers/react-tailwind/config.yaml
renderer: react-tailwind
description: React components with Tailwind CSS

mapping:
  tokens:
    colors: tailwind.config.js → theme.extend.colors
    typography: tailwind.config.js → theme.extend.fontSize
    spacing: tailwind.config.js → theme.extend.spacing
    
  components:
    Button: 
      import: "@/components/ui/button"
      propsMap:
        label: children
        variant: variant
        size: size
        
    Card:
      import: "@/components/ui/card"
      slots:
        header: CardHeader
        body: CardContent
        footer: CardFooter

conventions:
  fileNaming: PascalCase.tsx
  styling: className with cn() utility
  stateManagement: React hooks + React Query
  
templates:
  component: ./templates/component.tsx.hbs
  page: ./templates/page.tsx.hbs
```

### iOS SwiftUI Renderer

```yaml
# renderers/ios-swiftui/config.yaml
renderer: ios-swiftui
description: SwiftUI views for iOS/macOS

mapping:
  tokens:
    colors: Asset catalog + Color extension
    typography: Font extension with custom fonts
    spacing: CGFloat constants
    
  components:
    Button:
      native: Button with ButtonStyle
      propsMap:
        label: label (ViewBuilder)
        variant: custom ButtonStyle
        
    Card:
      native: VStack with background modifier
      styling: .background(RoundedRectangle)

conventions:
  fileNaming: PascalCase.swift
  stateManagement: @State, @ObservedObject, @EnvironmentObject
  
templates:
  view: ./templates/View.swift.hbs
```

### Figma Renderer

```yaml
# renderers/figma/config.yaml
renderer: figma
description: Figma components and auto-layout frames

mapping:
  tokens:
    colors: Local styles → Color styles
    typography: Local styles → Text styles
    spacing: Auto-layout properties
    
  components:
    Button:
      type: component
      variants: variant, size, state
      autoLayout: horizontal
      
    Card:
      type: frame
      autoLayout: vertical
      slots: nested frames

output:
  format: figma-plugin-api  # or .fig file via REST API
```

---

## File Structure

```
design-system/
├── README.md                    # Overview and usage
├── content/                     # Layer 0: Semantic content
│   ├── job.yaml
│   ├── employer.yaml
│   ├── user.yaml
│   └── ...
├── tokens/                      # Layer 1: Design tokens
│   ├── colors.yaml
│   ├── typography.yaml
│   ├── spacing.yaml
│   ├── radii.yaml
│   └── shadows.yaml
├── components/                  # Layer 2: Component specs
│   ├── button.yaml
│   ├── card.yaml
│   ├── input.yaml
│   ├── dialog.yaml
│   └── ...
├── patterns/                    # Layer 3: Composition patterns
│   ├── list-page.yaml
│   ├── detail-page.yaml
│   ├── data-table.yaml
│   ├── form-section.yaml
│   └── ...
├── behaviors/                   # Layer 4: Interaction specs
│   ├── form-submission.yaml
│   ├── modal-confirmation.yaml
│   ├── infinite-scroll.yaml
│   └── ...
├── renderers/                   # Layer 5: Platform implementations
│   ├── react-tailwind/
│   │   ├── config.yaml
│   │   └── templates/
│   ├── ios-swiftui/
│   │   ├── config.yaml
│   │   └── templates/
│   ├── android-compose/
│   ├── figma/
│   └── email/
└── generated/                   # Output from renderers
    ├── web/
    ├── ios/
    ├── android/
    └── figma/
```

---

## How Generation Works

### Static Generation (Build-time)

```
1. Load specs (tokens, components, patterns)
2. Select renderer (react-tailwind)
3. Apply templates
4. Output: production code

design-system/components/button.yaml
        ↓
renderers/react-tailwind/templates/component.tsx.hbs
        ↓
generated/web/components/Button.tsx
```

### AI Generation (Runtime)

```
1. AI receives task: "Build a job listing page"
2. AI loads relevant specs:
   - content/job.yaml (what data exists)
   - patterns/list-page.yaml (structure)
   - components/card.yaml, button.yaml (pieces)
   - behaviors/infinite-scroll.yaml (interaction)
3. AI loads renderer rules:
   - renderers/react-tailwind/config.yaml
4. AI generates platform-specific code
```

### Figma Generation

```
1. Load component specs
2. Use Figma renderer
3. Output: Figma plugin commands or REST API calls
4. Result: Components in Figma that match code exactly
```

---

## Why YAML?

1. **Human readable** — designers and devs can edit
2. **Machine readable** — parseable by any tool
3. **Diffable** — meaningful PRs for design changes
4. **Portable** — no vendor lock-in
5. **AI-friendly** — structured enough for reliable parsing

Alternatives considered:
- JSON — less readable, no comments
- Design tokens format (W3C) — too narrow, just tokens
- Figma .fig — proprietary, binary
- Custom DSL — learning curve, tooling burden

---

## Migration Path

### From Current State

1. **Extract tokens** — Pull colors, typography, spacing from rosetta.md into YAML
2. **Define components** — Document Button, Card, Input abstractly
3. **Map to React** — Create react-tailwind renderer that matches current code
4. **Validate** — Generated code should match existing components

### Incremental Adoption

- Start with tokens (highest leverage, lowest risk)
- Add components one at a time
- Keep existing code working — generated code is additive
- Gradually replace hand-written with generated

---

## Open Questions

1. **Granularity:** How detailed should component specs be? Every prop, or just the key ones?

2. **Inheritance:** Can components extend others? (e.g., IconButton extends Button)

3. **Theming:** How do we handle theme variants (light/dark) in specs?

4. **Validation:** How do we ensure specs stay in sync with implementations?

5. **Design input:** How do designers author specs? Direct YAML? Visual tool that outputs YAML?

6. **Versioning:** How do we version specs independently from implementations?

---

## Next Steps

1. **Validate architecture** — Does this model work for Handshake's needs?
2. **Extract tokens first** — Lowest risk, immediate value
3. **Spec 3-5 core components** — Button, Card, Input, Dialog, Table
4. **Build one renderer** — React + Tailwind (matches current stack)
5. **Test generation** — Can we generate a page that matches existing?
6. **Add second renderer** — Proves platform-agnostic claim (Figma or mobile)
