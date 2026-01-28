# Design Spec Generator — Project Plan

**Goal:** Build a tool that takes design specs (YAML) and generates publishable component libraries for web (React), mobile (Swift, Kotlin), and Figma.

**POC Scope:** React only, but architecture designed for multi-platform.

---

## Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          DESIGN SPECS                                │
│                                                                      │
│   tokens/              components/           patterns/               │
│   ├── colors.yaml      ├── button.yaml       ├── list-page.yaml    │
│   ├── typography.yaml  ├── card.yaml         └── ...               │
│   └── ...              └── ...                                      │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        DESIGN SPEC GENERATOR                         │
│                              (dspec)                                 │
│                                                                      │
│   ┌──────────┐    ┌──────────┐    ┌──────────────────────────────┐  │
│   │  Parser  │ →  │    IR    │ →  │        Generators            │  │
│   │  (YAML)  │    │ (Model)  │    │  ┌────────┐ ┌────────┐      │  │
│   └──────────┘    └──────────┘    │  │ React  │ │ Swift  │ ...  │  │
│                                   │  └────────┘ └────────┘      │  │
│                                   └──────────────────────────────┘  │
└─────────────────────────────────────┬───────────────────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              ▼                       ▼                       ▼
┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│   @org/rosetta-react │  │  @org/rosetta-ios   │  │  figma-tokens.json  │
│                      │  │                      │  │                      │
│   Button.tsx         │  │  Button.swift        │  │  (Figma plugin      │
│   Card.tsx           │  │  Card.swift          │  │   compatible)        │
│   tokens.css         │  │  Tokens.swift        │  │                      │
└─────────────────────┘  └─────────────────────┘  └─────────────────────┘
```

---

## Tool: `dspec`

### Name

`dspec` — Design Spec CLI

```bash
dspec generate --target react --output ./packages/react
dspec generate --target swift --output ./packages/ios
dspec generate --target figma --output ./dist/figma-tokens.json
dspec validate
dspec init
```

### Core Principles

1. **Specs are the source of truth** — Code is derived, not authored
2. **IR as the pivot** — All generators consume the same intermediate representation
3. **Generators are pluggable** — Add new targets without changing core
4. **Quality is built in** — Generated code passes lint, types, tests
5. **Human escape hatches** — Override files for edge cases

---

## Architecture

### Layer 1: Schema

JSON Schema definitions for valid specs.

```
schemas/
├── tokens.schema.json        # Color, typography, spacing schemas
├── component.schema.json     # Component definition schema
├── pattern.schema.json       # Pattern composition schema
└── behavior.schema.json      # Interaction/flow schema
```

Purpose:
- Validate specs before processing
- Enable IDE autocomplete for YAML authoring
- Document what's allowed

### Layer 2: Parser

Loads YAML specs, validates against schemas, produces typed objects.

```typescript
// parser/index.ts

interface ParseResult {
  tokens: TokenDefinitions;
  components: ComponentDefinition[];
  patterns: PatternDefinition[];
  behaviors: BehaviorDefinition[];
  errors: ValidationError[];
}

function parse(specDir: string): ParseResult {
  // 1. Discover all YAML files
  // 2. Load and parse
  // 3. Validate against schemas
  // 4. Resolve references (e.g., button references color tokens)
  // 5. Return typed result or errors
}
```

### Layer 3: Intermediate Representation (IR)

Normalized, platform-agnostic model. This is the key to portability.

```typescript
// ir/types.ts

interface DesignSystem {
  name: string;
  version: string;
  tokens: Tokens;
  components: Component[];
  patterns: Pattern[];
  behaviors: Behavior[];
}

interface Tokens {
  colors: Record<string, ColorValue>;
  typography: Record<string, TypographyValue>;
  spacing: Record<string, number>;
  radii: Record<string, number>;
  shadows: Record<string, ShadowValue>;
}

interface Component {
  name: string;
  description: string;
  props: Prop[];
  variants: Variant[];
  states: State[];
  styling: StylingRules;
  accessibility: AccessibilityRules;
  tests: TestCase[];
}

interface Prop {
  name: string;
  type: PropType;
  required: boolean;
  default?: unknown;
  description: string;
}

interface Variant {
  name: string;
  styling: StylingRules;
}

interface StylingRules {
  base: StyleProperties;
  states: Record<string, StyleProperties>;
}

interface StyleProperties {
  // Platform-agnostic style properties
  backgroundColor?: TokenReference | string;
  textColor?: TokenReference | string;
  fontSize?: TokenReference | number;
  padding?: SpacingValue;
  borderRadius?: TokenReference | number;
  // ... etc
}
```

The IR is:
- **Typed** — Full TypeScript types
- **Resolved** — Token references point to actual values
- **Normalized** — Consistent structure regardless of YAML quirks
- **Complete** — Everything a generator needs

### Layer 4: Generators

Each generator consumes IR and produces platform-specific output.

```typescript
// generators/base.ts

interface Generator {
  name: string;
  
  // Generate all files for this target
  generate(ir: DesignSystem, options: GeneratorOptions): GeneratedFile[];
  
  // What file types this generator produces
  outputTypes: string[];
}

interface GeneratedFile {
  path: string;
  content: string;
  type: 'source' | 'test' | 'story' | 'config';
}

interface GeneratorOptions {
  outputDir: string;
  packageName: string;
  formatting: FormattingOptions;
}
```

#### React Generator (POC Focus)

```typescript
// generators/react/index.ts

class ReactGenerator implements Generator {
  name = 'react';
  outputTypes = ['.tsx', '.ts', '.css', '.test.tsx', '.stories.tsx'];
  
  generate(ir: DesignSystem, options: GeneratorOptions): GeneratedFile[] {
    const files: GeneratedFile[] = [];
    
    // 1. Generate tokens
    files.push(this.generateTokensCSS(ir.tokens));
    files.push(this.generateTokensTS(ir.tokens));
    
    // 2. Generate components
    for (const component of ir.components) {
      files.push(this.generateComponent(component, ir.tokens));
      files.push(this.generateTests(component));
      files.push(this.generateStories(component));
      files.push(this.generateIndex(component));
    }
    
    // 3. Generate package files
    files.push(this.generatePackageJson(ir, options));
    files.push(this.generateMainExports(ir.components));
    
    return files;
  }
  
  private generateComponent(component: Component, tokens: Tokens): GeneratedFile {
    // Use templates + IR to generate React component
    // CVA for variants, Tailwind for styling, forwardRef, etc.
  }
}
```

#### Swift Generator (Future)

```typescript
// generators/swift/index.ts

class SwiftGenerator implements Generator {
  name = 'swift';
  outputTypes = ['.swift'];
  
  generate(ir: DesignSystem, options: GeneratorOptions): GeneratedFile[] {
    // Generate Swift structs, enums, ViewModifiers
  }
}
```

#### Figma Generator (Future)

```typescript
// generators/figma/index.ts

class FigmaGenerator implements Generator {
  name = 'figma';
  outputTypes = ['.json'];
  
  generate(ir: DesignSystem, options: GeneratorOptions): GeneratedFile[] {
    // Generate Figma Tokens plugin compatible JSON
  }
}
```

### Layer 5: Publisher

Packages generated files for distribution.

```typescript
// publisher/index.ts

interface PublishOptions {
  registry: 'npm' | 'github' | 'local';
  version: string;
  dryRun: boolean;
}

async function publish(generatedFiles: GeneratedFile[], options: PublishOptions) {
  // 1. Write files to temp directory
  // 2. Run build step (if needed)
  // 3. Run tests
  // 4. Publish to registry
}
```

---

## Spec Formats

### tokens/colors.yaml

```yaml
colors:
  background:
    primary:
      value: "#FFFFFF"
      description: Primary canvas, card surfaces
    secondary:
      value: "#F6F6F6"
      description: Secondary surfaces
    inverted:
      value: "#121212"
      description: Dark backgrounds
      
  foreground:
    primary:
      value: "#121212"
      description: Primary text
    secondary:
      value: "rgba(18, 18, 18, 0.7)"
      description: Secondary/muted text
      
  interactive:
    primary:
      value: "#04191B"
      description: Primary buttons, key actions
      
  status:
    positive:
      background: "#A5E09F"
      foreground: "#327D0F"
    negative:
      background: "#FFC2C4"
      foreground: "#BB3643"
```

### components/button.yaml

```yaml
name: Button
description: Primary interactive element for actions

props:
  - name: children
    type: node
    required: true
    
  - name: variant
    type: enum
    values: [primary, secondary, tertiary, destructive]
    default: primary
    
  - name: size
    type: enum
    values: [sm, md, lg]
    default: md
    
  - name: disabled
    type: boolean
    default: false
    
  - name: loading
    type: boolean
    default: false
    description: Shows spinner, disables interaction

styling:
  base:
    display: inline-flex
    alignItems: center
    justifyContent: center
    gap: $spacing.sm
    fontWeight: 600
    borderRadius: $radii.md
    cursor: pointer
    transition:
      properties: [background-color, border-color]
      duration: 150ms
      
  sizes:
    sm:
      height: 36
      paddingX: $spacing.md
      fontSize: $typography.body-sm.size
    md:
      height: 48
      paddingX: $spacing.lg
      fontSize: $typography.body-md.size
    lg:
      height: 60
      paddingX: $spacing.xl
      fontSize: $typography.body-lg.size
      
  variants:
    primary:
      backgroundColor: $colors.interactive.primary
      textColor: $colors.foreground.inverted
      states:
        hover:
          backgroundColor: "#062E32"
        active:
          backgroundColor: "#073D42"
        disabled:
          backgroundColor: $colors.background.disabled
          textColor: $colors.foreground.disabled
          
    secondary:
      backgroundColor: $colors.background.primary
      textColor: $colors.foreground.primary
      border:
        width: 1
        color: $colors.foreground.border

accessibility:
  role: button
  focusRing:
    width: 2
    color: $colors.interactive.focus
    offset: 2
  ariaAttributes:
    - name: aria-disabled
      bind: disabled || loading
    - name: aria-busy
      bind: loading

tests:
  - name: renders with default props
    assert: renders
    
  - name: renders each variant
    for: [primary, secondary, tertiary, destructive]
    props:
      variant: $item
    assert: renders
    
  - name: shows loading state
    props:
      loading: true
    assert:
      - hasSpinner: true
      - isDisabled: true
      
  - name: calls onClick
    simulate: click
    assert: onClickCalled
    
  - name: does not call onClick when disabled
    props:
      disabled: true
    simulate: click
    assert: onClickNotCalled
```

---

## Generated Output (React POC)

### tokens.css

```css
/* Generated by dspec v1.0.0 */
/* DO NOT EDIT — regenerate with: dspec generate --target react */

:root {
  /* Colors - Background */
  --color-bg-primary: #FFFFFF;
  --color-bg-secondary: #F6F6F6;
  --color-bg-inverted: #121212;
  
  /* Colors - Foreground */
  --color-fg-primary: #121212;
  --color-fg-secondary: rgba(18, 18, 18, 0.7);
  
  /* Colors - Interactive */
  --color-interactive-primary: #04191B;
  
  /* Spacing */
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  
  /* Radii */
  --radius-md: 8px;
  
  /* Typography */
  --font-size-body-sm: 13px;
  --font-size-body-md: 15px;
  --font-size-body-lg: 17px;
}
```

### Button/Button.tsx

```tsx
/**
 * Generated by dspec v1.0.0
 * Source: components/button.yaml
 * DO NOT EDIT — regenerate with: dspec generate --target react
 */

import * as React from 'react';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '../utils/cn';
import { Spinner } from '../Spinner';

const buttonVariants = cva(
  'inline-flex items-center justify-center gap-2 font-semibold rounded-[var(--radius-md)] cursor-pointer transition-colors duration-150 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-[var(--color-interactive-focus)] focus-visible:ring-offset-2 disabled:cursor-not-allowed',
  {
    variants: {
      variant: {
        primary: 'bg-[var(--color-interactive-primary)] text-[var(--color-fg-inverted)] hover:bg-[#062E32] active:bg-[#073D42] disabled:bg-[var(--color-bg-disabled)] disabled:text-[var(--color-fg-disabled)]',
        secondary: 'bg-[var(--color-bg-primary)] text-[var(--color-fg-primary)] border border-[var(--color-fg-border)] hover:bg-[var(--color-bg-tertiary)]',
        tertiary: 'bg-transparent text-[var(--color-fg-primary)] hover:bg-[var(--color-bg-secondary)]',
        destructive: 'bg-[var(--color-status-negative-fg)] text-[var(--color-fg-inverted)]',
      },
      size: {
        sm: 'h-9 px-[var(--spacing-md)] text-[var(--font-size-body-sm)]',
        md: 'h-12 px-[var(--spacing-lg)] text-[var(--font-size-body-md)]',
        lg: 'h-[60px] px-[var(--spacing-xl)] text-[var(--font-size-body-lg)]',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  loading?: boolean;
}

export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, loading, disabled, children, ...props }, ref) => {
    const isDisabled = disabled || loading;
    
    return (
      <button
        ref={ref}
        className={cn(buttonVariants({ variant, size }), className)}
        disabled={isDisabled}
        aria-disabled={isDisabled}
        aria-busy={loading}
        {...props}
      >
        {loading && <Spinner className="h-4 w-4" />}
        {children}
      </button>
    );
  }
);

Button.displayName = 'Button';
```

### Button/Button.test.tsx

```tsx
/**
 * Generated by dspec v1.0.0
 * Source: components/button.yaml
 */

import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

describe('Button', () => {
  it('renders with default props', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByRole('button')).toBeInTheDocument();
  });

  it.each(['primary', 'secondary', 'tertiary', 'destructive'] as const)(
    'renders %s variant',
    (variant) => {
      render(<Button variant={variant}>Button</Button>);
      expect(screen.getByRole('button')).toBeInTheDocument();
    }
  );

  it('shows loading state', () => {
    render(<Button loading>Loading</Button>);
    const button = screen.getByRole('button');
    expect(button).toHaveAttribute('aria-busy', 'true');
    expect(button).toBeDisabled();
  });

  it('calls onClick', async () => {
    const onClick = jest.fn();
    render(<Button onClick={onClick}>Click</Button>);
    await userEvent.click(screen.getByRole('button'));
    expect(onClick).toHaveBeenCalled();
  });

  it('does not call onClick when disabled', async () => {
    const onClick = jest.fn();
    render(<Button disabled onClick={onClick}>Click</Button>);
    await userEvent.click(screen.getByRole('button'));
    expect(onClick).not.toHaveBeenCalled();
  });
});
```

---

## Project Phases

### Phase 1: Foundation (Week 1-2)

**Goal:** Core architecture, token generation, one component.

| Task | Description | Output |
|------|-------------|--------|
| 1.1 | Project setup | Repo, TypeScript, build system |
| 1.2 | Schema definitions | JSON schemas for tokens, components |
| 1.3 | Parser implementation | YAML → validated typed objects |
| 1.4 | IR types | Full type definitions for intermediate representation |
| 1.5 | Token generator (React) | tokens.yaml → tokens.css + tokens.ts |
| 1.6 | Button generator (React) | button.yaml → Button.tsx + tests + stories |
| 1.7 | CLI scaffold | `dspec generate --target react` |

**Milestone:** Generate Button component from YAML spec.

### Phase 2: Component Library (Week 3-4)

**Goal:** Generate full component set, publishable package.

| Task | Description | Output |
|------|-------------|--------|
| 2.1 | Add components | Card, Input, Dialog, Select, Table |
| 2.2 | Package generation | package.json, exports, types |
| 2.3 | Storybook generation | Stories for all components |
| 2.4 | Test generation | Full test coverage |
| 2.5 | CI integration | GitHub Actions for validate + generate |
| 2.6 | Publishing | npm publish workflow |

**Milestone:** Publishable @org/rosetta-react package.

### Phase 3: Patterns & Behaviors (Week 5-6)

**Goal:** Higher-level compositions and interactions.

| Task | Description | Output |
|------|-------------|--------|
| 3.1 | Pattern schema | Define pattern composition format |
| 3.2 | Pattern generator | patterns/list-page.yaml → composition docs |
| 3.3 | Behavior schema | Define interaction/flow format |
| 3.4 | Context.md generation | AI-optimized documentation |
| 3.5 | Full design-context package | Tokens + components + patterns + prose |

**Milestone:** Full design context for AI consumption.

### Phase 4: Multi-Platform (Week 7-8)

**Goal:** Add Swift and Figma generators.

| Task | Description | Output |
|------|-------------|--------|
| 4.1 | Swift token generator | tokens.yaml → Tokens.swift |
| 4.2 | Swift component generator | button.yaml → Button.swift (SwiftUI) |
| 4.3 | Figma token generator | tokens.yaml → figma-tokens.json |
| 4.4 | Generator plugin system | Easy to add new targets |

**Milestone:** Same specs → React + Swift + Figma outputs.

### Phase 5: Production Hardening (Week 9-10)

**Goal:** Ready for real usage.

| Task | Description | Output |
|------|-------------|--------|
| 5.1 | Override system | Escape hatches for edge cases |
| 5.2 | Incremental generation | Only regenerate changed components |
| 5.3 | Visual regression | Chromatic/Percy integration |
| 5.4 | Documentation | Full docs for spec authoring |
| 5.5 | Handshake migration | Import existing Rosetta into specs |

**Milestone:** Production-ready tool, Handshake using it.

---

## Tech Stack

### Core

- **Language:** TypeScript
- **Runtime:** Node.js
- **Build:** tsup or unbuild
- **CLI:** Commander or oclif
- **YAML:** yaml package
- **Schema validation:** Ajv
- **Templating:** Handlebars or EJS (for complex generation)

### React Generator

- **Styling:** CVA (class-variance-authority) + Tailwind
- **Components:** Radix UI primitives where applicable
- **Testing:** Vitest + Testing Library
- **Storybook:** Auto-generated stories

### Swift Generator (Future)

- **UI Framework:** SwiftUI
- **Package:** Swift Package Manager

### Figma Generator (Future)

- **Format:** Figma Tokens plugin JSON
- **API:** Figma REST API for direct sync (optional)

---

## File Structure

```
dspec/
├── package.json
├── tsconfig.json
├── README.md
│
├── src/
│   ├── cli/
│   │   ├── index.ts           # CLI entry point
│   │   ├── commands/
│   │   │   ├── generate.ts
│   │   │   ├── validate.ts
│   │   │   └── init.ts
│   │   └── utils/
│   │
│   ├── schemas/
│   │   ├── tokens.schema.json
│   │   ├── component.schema.json
│   │   └── index.ts
│   │
│   ├── parser/
│   │   ├── index.ts
│   │   ├── tokens.ts
│   │   ├── components.ts
│   │   └── resolver.ts        # Resolve $references
│   │
│   ├── ir/
│   │   ├── types.ts           # IR type definitions
│   │   └── transform.ts       # Parsed → IR
│   │
│   ├── generators/
│   │   ├── base.ts            # Generator interface
│   │   ├── react/
│   │   │   ├── index.ts
│   │   │   ├── tokens.ts
│   │   │   ├── component.ts
│   │   │   ├── tests.ts
│   │   │   ├── stories.ts
│   │   │   └── templates/
│   │   │       ├── component.hbs
│   │   │       ├── test.hbs
│   │   │       └── story.hbs
│   │   ├── swift/             # Future
│   │   └── figma/             # Future
│   │
│   └── publisher/
│       ├── index.ts
│       └── npm.ts
│
├── test/
│   ├── fixtures/
│   │   ├── tokens/
│   │   └── components/
│   ├── parser.test.ts
│   ├── generators/
│   │   └── react.test.ts
│   └── e2e/
│       └── generate.test.ts
│
└── examples/
    └── basic/
        ├── specs/
        │   ├── tokens/
        │   └── components/
        └── expected-output/
```

---

## CLI Interface

```bash
# Initialize new spec project
dspec init --name my-design-system

# Validate specs
dspec validate
dspec validate --spec tokens/colors.yaml

# Generate for a target
dspec generate --target react --output ./packages/react
dspec generate --target swift --output ./packages/ios
dspec generate --target figma --output ./dist/figma.json

# Generate all targets
dspec generate --all

# Watch mode (regenerate on spec changes)
dspec generate --target react --watch

# Publish (after generate)
dspec publish --target react --registry npm
```

---

## Success Criteria

### POC (Phase 1-2)

- [ ] `dspec generate --target react` produces working Button, Card, Input
- [ ] Generated components pass TypeScript
- [ ] Generated tests pass
- [ ] Generated package is publishable to npm
- [ ] Round-trip: edit spec → regenerate → updated component

### MVP (Phase 3-4)

- [ ] Full component set (10+ components)
- [ ] Patterns and behaviors documented
- [ ] context.md generated for AI consumption
- [ ] Swift generator produces working code
- [ ] Figma tokens sync works

### Production (Phase 5)

- [ ] Handshake using tool for Rosetta
- [ ] Override system handles edge cases
- [ ] CI/CD fully automated
- [ ] Documentation complete

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Specs can't express everything | Override files, custom code blocks |
| Generated code quality issues | Quality gates, snapshot tests |
| Multi-platform consistency | IR is the contract, test across platforms |
| Adoption resistance | Start with tokens (low risk), expand |
| Maintenance burden | Automate everything, good docs |

---

## Open Questions

1. **Repo structure:** Monorepo (specs + generators + output) or separate?
2. **AI in the loop:** Use AI for generation or pure templates?
3. **Existing Rosetta:** Migrate specs from existing code or start fresh?
4. **Component complexity:** How much logic can specs express?
5. **Versioning:** How do spec versions map to package versions?

---

## Next Steps

1. **Create repo:** `dspec` or similar
2. **Set up project:** TypeScript, build, CLI scaffold
3. **Define schemas:** Start with tokens, then components
4. **Build parser:** YAML → validated objects
5. **Build IR:** Transform to intermediate representation
6. **Build React generator:** Tokens first, then Button
7. **Test end-to-end:** Spec → working component
