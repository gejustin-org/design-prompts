# AI-Maintained Component Library

**Extension:** Design specs are the source of truth. AI generates and maintains the actual React component library.

---

## The Extended Model

```
┌──────────────────────────────────────────────────────────────────┐
│                     DESIGN SPECS (Source of Truth)                │
│                                                                   │
│   tokens/          components/        patterns/                   │
│   colors.yaml      button.yaml        list-page.yaml             │
│   typography.yaml  card.yaml          data-table.yaml            │
│                    input.yaml                                     │
└──────────────────────────────────┬───────────────────────────────┘
                                   │
                                   ▼  AI generates
┌──────────────────────────────────────────────────────────────────┐
│                     COMPONENT LIBRARY (Generated Artifact)        │
│                     @handshake/rosetta-react                      │
│                                                                   │
│   src/                                                            │
│   ├── Button/                                                     │
│   │   ├── Button.tsx           # Generated                       │
│   │   ├── Button.test.tsx      # Generated                       │
│   │   ├── Button.stories.tsx   # Generated                       │
│   │   └── index.ts                                                │
│   ├── Card/                                                       │
│   ├── Input/                                                      │
│   └── ...                                                         │
└──────────────────────────────────┬───────────────────────────────┘
                                   │
                                   ▼  npm publish
┌──────────────────────────────────────────────────────────────────┐
│                     PUBLISHED PACKAGES                            │
│                                                                   │
│   @handshake/design-context    # Specs, tokens, AI context       │
│   @handshake/rosetta-react     # React components                │
│   @handshake/rosetta-native    # React Native (future)           │
└──────────────────────────────────────────────────────────────────┘
```

---

## How It Works

### 1. Component Spec (Human-Authored)

```yaml
# components/button.yaml

component: Button
description: Primary interactive element for actions

props:
  - name: children
    type: ReactNode
    required: true
    description: Button content
    
  - name: variant
    type: enum
    values: [primary, secondary, tertiary, destructive]
    default: primary
    
  - name: size
    type: enum
    values: [sm, md, lg]
    default: md
    
  - name: icon
    type: IconName
    optional: true
    position: leading
    
  - name: iconPosition
    type: enum
    values: [leading, trailing]
    default: leading
    when: icon is set
    
  - name: loading
    type: boolean
    default: false
    behavior: shows spinner, disables interaction
    
  - name: disabled
    type: boolean
    default: false
    
  - name: fullWidth
    type: boolean
    default: false
    
  - name: asChild
    type: boolean
    default: false
    description: Render as Slot for composition

states:
  - default
  - hover
  - active
  - focus
  - disabled
  - loading

styling:
  base:
    display: inline-flex
    alignItems: center
    justifyContent: center
    gap: spacing.sm
    fontWeight: 600
    borderRadius: radii.md
    transition: [background-color, border-color, box-shadow] 150ms ease
    cursor: pointer
    
  sizes:
    sm:
      height: 36px
      paddingX: spacing.md
      fontSize: typography.body-sm
    md:
      height: 48px
      paddingX: spacing.lg
      fontSize: typography.body-md
    lg:
      height: 60px
      paddingX: spacing.xl
      fontSize: typography.body-lg
      
  variants:
    primary:
      background: interactive.primary
      color: foreground.inverted
      border: none
      hover:
        background: interactive.primary-hover
      active:
        background: interactive.primary-active
      focus:
        ring: { width: 2, color: interactive.focus, offset: 2 }
        
    secondary:
      background: background.primary
      color: foreground.primary
      border: { width: 1, color: foreground.border }
      hover:
        background: background.tertiary
      active:
        background: background.quaternary
        
    destructive:
      background: status.negative.foreground
      color: foreground.inverted
      border: none

  disabled:
    background: background.disabled
    color: foreground.disabled
    cursor: not-allowed
    
  loading:
    extends: disabled
    content: spinner

accessibility:
  role: button
  focusable: true
  keyboardActivation: [Enter, Space]
  
tests:
  - renders with default props
  - renders each variant
  - renders each size
  - shows loading spinner when loading=true
  - is disabled when disabled=true or loading=true
  - calls onClick when clicked
  - does not call onClick when disabled
  - renders icon in correct position
  - supports asChild composition
  - has accessible focus state
```

### 2. AI Generation Step

```typescript
// build/generate-components.ts

async function generateComponents() {
  const specs = await loadYaml('components/**/*.yaml');
  const tokens = await loadCompiledTokens();
  
  for (const spec of specs) {
    // AI generates the component
    const result = await ai.generate({
      task: 'generate-react-component',
      inputs: {
        spec,
        tokens,
        conventions: await loadConventions(),
        existingCode: await loadExisting(spec.component), // for updates
      },
      outputs: [
        `src/${spec.component}/${spec.component}.tsx`,
        `src/${spec.component}/${spec.component}.test.tsx`,
        `src/${spec.component}/${spec.component}.stories.tsx`,
        `src/${spec.component}/index.ts`,
      ],
    });
    
    // Validate generated code
    await typecheck(result);
    await lint(result);
    await runTests(result);
    
    // Write if passes
    await writeFiles(result);
  }
}
```

### 3. Generated Output

**Button.tsx:**

```tsx
// AUTO-GENERATED from components/button.yaml
// Spec version: 2.4.0
// Generated: 2026-01-28T14:30:00Z
// 
// DO NOT EDIT DIRECTLY - update the spec and regenerate
// Manual overrides: see Button.overrides.tsx

import * as React from 'react';
import { Slot } from '@radix-ui/react-slot';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';
import { Icon, type IconName } from '../Icon';
import { Spinner } from '../Spinner';

const buttonVariants = cva(
  // Base styles
  `inline-flex items-center justify-center gap-2 font-semibold
   rounded-lg transition-colors duration-150 ease-out
   focus-visible:outline-none focus-visible:ring-2 
   focus-visible:ring-interactive-focus focus-visible:ring-offset-2
   disabled:pointer-events-none disabled:bg-disabled disabled:text-disabled-foreground`,
  {
    variants: {
      variant: {
        primary: 
          'bg-interactive-primary text-inverted-foreground hover:bg-interactive-primary-hover active:bg-interactive-primary-active',
        secondary:
          'bg-primary text-primary-foreground border border-fg-border hover:bg-tertiary active:bg-quaternary',
        tertiary:
          'bg-transparent text-primary-foreground hover:bg-secondary active:bg-tertiary',
        destructive:
          'bg-status-negative-foreground text-inverted-foreground hover:bg-status-negative-hover',
      },
      size: {
        sm: 'h-9 px-4 text-sm',
        md: 'h-12 px-6 text-base',
        lg: 'h-15 px-8 text-lg',
      },
      fullWidth: {
        true: 'w-full',
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
  /** Button content */
  children: React.ReactNode;
  /** Icon to display */
  icon?: IconName;
  /** Icon position */
  iconPosition?: 'leading' | 'trailing';
  /** Show loading spinner */
  loading?: boolean;
  /** Render as child element (Slot) */
  asChild?: boolean;
}

/**
 * Button component for user actions.
 * 
 * @example
 * // Primary button
 * <Button>Save changes</Button>
 * 
 * @example
 * // With icon
 * <Button icon="plus">Add item</Button>
 * 
 * @example
 * // Loading state
 * <Button loading>Saving...</Button>
 * 
 * @example
 * // As link
 * <Button asChild>
 *   <a href="/somewhere">Go there</a>
 * </Button>
 */
export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  (
    {
      className,
      variant,
      size,
      fullWidth,
      icon,
      iconPosition = 'leading',
      loading = false,
      disabled,
      asChild = false,
      children,
      ...props
    },
    ref
  ) => {
    const Comp = asChild ? Slot : 'button';
    const isDisabled = disabled || loading;

    const iconElement = icon && !loading && (
      <Icon name={icon} className="h-4 w-4" aria-hidden />
    );
    
    const spinnerElement = loading && (
      <Spinner className="h-4 w-4" aria-hidden />
    );

    return (
      <Comp
        ref={ref}
        className={cn(buttonVariants({ variant, size, fullWidth }), className)}
        disabled={isDisabled}
        aria-disabled={isDisabled}
        aria-busy={loading}
        {...props}
      >
        {loading && spinnerElement}
        {!loading && iconPosition === 'leading' && iconElement}
        {children}
        {!loading && iconPosition === 'trailing' && iconElement}
      </Comp>
    );
  }
);

Button.displayName = 'Button';
```

**Button.test.tsx:**

```tsx
// AUTO-GENERATED from components/button.yaml

import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

describe('Button', () => {
  it('renders with default props', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByRole('button', { name: 'Click me' })).toBeInTheDocument();
  });

  describe('variants', () => {
    it.each(['primary', 'secondary', 'tertiary', 'destructive'] as const)(
      'renders %s variant',
      (variant) => {
        render(<Button variant={variant}>Button</Button>);
        expect(screen.getByRole('button')).toBeInTheDocument();
      }
    );
  });

  describe('sizes', () => {
    it.each(['sm', 'md', 'lg'] as const)('renders %s size', (size) => {
      render(<Button size={size}>Button</Button>);
      expect(screen.getByRole('button')).toBeInTheDocument();
    });
  });

  it('shows loading spinner when loading=true', () => {
    render(<Button loading>Loading</Button>);
    expect(screen.getByRole('button')).toHaveAttribute('aria-busy', 'true');
  });

  it('is disabled when disabled=true', () => {
    render(<Button disabled>Disabled</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });

  it('is disabled when loading=true', () => {
    render(<Button loading>Loading</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });

  it('calls onClick when clicked', async () => {
    const onClick = jest.fn();
    render(<Button onClick={onClick}>Click me</Button>);
    await userEvent.click(screen.getByRole('button'));
    expect(onClick).toHaveBeenCalledTimes(1);
  });

  it('does not call onClick when disabled', async () => {
    const onClick = jest.fn();
    render(<Button disabled onClick={onClick}>Click me</Button>);
    await userEvent.click(screen.getByRole('button'));
    expect(onClick).not.toHaveBeenCalled();
  });

  it('renders icon in leading position by default', () => {
    render(<Button icon="plus">Add</Button>);
    const button = screen.getByRole('button');
    // Icon should be before text
    expect(button.firstChild).toHaveAttribute('aria-hidden', 'true');
  });

  it('renders icon in trailing position', () => {
    render(<Button icon="arrow-right" iconPosition="trailing">Next</Button>);
    const button = screen.getByRole('button');
    // Icon should be after text
    expect(button.lastChild).toHaveAttribute('aria-hidden', 'true');
  });

  it('has accessible focus state', () => {
    render(<Button>Focus me</Button>);
    const button = screen.getByRole('button');
    button.focus();
    expect(button).toHaveFocus();
    // Visual focus ring tested via visual regression
  });
});
```

**Button.stories.tsx:**

```tsx
// AUTO-GENERATED from components/button.yaml

import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  tags: ['autodocs'],
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'tertiary', 'destructive'],
    },
    size: {
      control: 'select',
      options: ['sm', 'md', 'lg'],
    },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: {
    children: 'Primary Button',
    variant: 'primary',
  },
};

export const Secondary: Story = {
  args: {
    children: 'Secondary Button',
    variant: 'secondary',
  },
};

export const Tertiary: Story = {
  args: {
    children: 'Tertiary Button',
    variant: 'tertiary',
  },
};

export const Destructive: Story = {
  args: {
    children: 'Delete',
    variant: 'destructive',
  },
};

export const WithIcon: Story = {
  args: {
    children: 'Add Item',
    icon: 'plus',
  },
};

export const IconTrailing: Story = {
  args: {
    children: 'Next',
    icon: 'arrow-right',
    iconPosition: 'trailing',
  },
};

export const Loading: Story = {
  args: {
    children: 'Saving...',
    loading: true,
  },
};

export const Disabled: Story = {
  args: {
    children: 'Disabled',
    disabled: true,
  },
};

export const FullWidth: Story = {
  args: {
    children: 'Full Width Button',
    fullWidth: true,
  },
};

export const AllSizes: Story = {
  render: () => (
    <div className="flex items-center gap-4">
      <Button size="sm">Small</Button>
      <Button size="md">Medium</Button>
      <Button size="lg">Large</Button>
    </div>
  ),
};
```

---

## Maintenance: Spec Changes → Regenerate

### Scenario: Add new variant

1. **Update spec:**
```yaml
# components/button.yaml
variants:
  # ... existing ...
  - name: ghost
    background: transparent
    color: foreground.primary
    border: none
    hover:
      background: background.secondary
```

2. **Run generation:**
```bash
yarn generate:components
```

3. **AI sees diff:**
```
Spec changed: button.yaml
+ New variant: ghost

Updating:
- Button.tsx: add ghost to buttonVariants
- Button.test.tsx: add ghost variant test
- Button.stories.tsx: add Ghost story
```

4. **Output updated, PR created**

---

## Handling the "Last Mile"

Some things can't be fully specified. Options:

### Option A: Override Files

```
src/Button/
├── Button.tsx           # Generated
├── Button.test.tsx      # Generated
├── Button.overrides.tsx # Human-maintained, merged in
└── index.ts             # Exports combined
```

### Option B: Escape Hatches in Spec

```yaml
# components/button.yaml

customCode:
  beforeRender: |
    // Custom logic before render
    const analyticsId = useAnalyticsId();
    
  afterProps: |
    // Add custom props
    'data-analytics-id': analyticsId,
```

### Option C: Post-Generation Hooks

```typescript
// build/post-generate.ts

export async function postGenerate(component: string, code: string) {
  if (component === 'Button') {
    // Add Handshake-specific analytics wrapper
    return injectAnalytics(code);
  }
  return code;
}
```

### Option D: Hybrid Ownership

```yaml
# components/button.yaml

ownership:
  generated:
    - types
    - variants
    - styling
    - stories
  human:
    - complex interactions
    - analytics integration
    - animation timing
```

---

## Quality Gates

### Before Merge

```yaml
# .github/workflows/generate.yml

steps:
  - name: Generate components
    run: yarn generate:components
    
  - name: TypeScript check
    run: yarn tsc --noEmit
    
  - name: Lint
    run: yarn lint
    
  - name: Unit tests
    run: yarn test
    
  - name: Visual regression
    run: yarn test:visual
    
  - name: Accessibility audit
    run: yarn test:a11y
    
  - name: Bundle size check
    run: yarn size-limit
```

### Snapshot Comparison

```typescript
// Tests compare generated output to previous version
expect(generatedButton).toMatchSnapshot();
```

If output changes unexpectedly, diff is visible in PR.

---

## The Full Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│  1. SPEC CHANGE (Human)                                          │
│     Edit components/button.yaml — add ghost variant              │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. PR CREATED                                                   │
│     CI triggers on components/*.yaml change                      │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. AI GENERATION (CI)                                           │
│     yarn generate:components                                     │
│     - Reads updated spec                                         │
│     - Generates/updates Button.tsx, .test.tsx, .stories.tsx     │
│     - Commits generated changes to PR                           │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. QUALITY GATES (CI)                                           │
│     - TypeScript: ✓                                              │
│     - Lint: ✓                                                    │
│     - Tests: ✓                                                   │
│     - Visual regression: ✓                                       │
│     - A11y: ✓                                                    │
│     - Bundle size: ✓                                             │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. HUMAN REVIEW                                                 │
│     - Review spec change (the intent)                            │
│     - Glance at generated diff (sanity check)                    │
│     - Approve if looks right                                     │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. MERGE + PUBLISH                                              │
│     - Merge to main                                              │
│     - Auto-publish @handshake/rosetta-react@2.5.0                │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. CONSUMPTION                                                  │
│     - Apps update dependency                                     │
│     - New ghost variant available                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Repo Structure (Full)

```
design-system/
├── package.json
├── rosetta.config.yaml
│
├── specs/                        # Source of truth
│   ├── tokens/
│   │   ├── colors.yaml
│   │   └── ...
│   ├── components/
│   │   ├── button.yaml
│   │   ├── card.yaml
│   │   └── ...
│   ├── patterns/
│   └── behaviors/
│
├── packages/
│   ├── design-context/           # Published: @handshake/design-context
│   │   ├── dist/
│   │   │   ├── context.json
│   │   │   ├── context.md
│   │   │   └── tokens.css
│   │   └── package.json
│   │
│   └── rosetta-react/            # Published: @handshake/rosetta-react
│       ├── src/
│       │   ├── Button/
│       │   │   ├── Button.tsx           # Generated
│       │   │   ├── Button.test.tsx      # Generated
│       │   │   ├── Button.stories.tsx   # Generated
│       │   │   └── index.ts
│       │   ├── Card/
│       │   └── ...
│       ├── package.json
│       └── tsconfig.json
│
├── build/
│   ├── generate-context.ts
│   ├── generate-components.ts
│   └── templates/
│
└── .github/
    └── workflows/
        ├── validate.yml
        ├── generate.yml
        └── publish.yml
```

---

## Why This Works

| Benefit | How |
|---------|-----|
| **Specs are readable** | YAML, not code — designers can contribute |
| **Code is consistent** | AI generates same patterns every time |
| **Quality is enforced** | Tests, types, visual regression in CI |
| **Changes are auditable** | Spec diff shows intent, code diff shows effect |
| **Humans stay in control** | Review spec changes, override when needed |
| **Scales** | Add component = add YAML file |

---

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| AI generates bad code | Quality gates catch it before merge |
| Spec can't express everything | Override files, custom code blocks |
| Generated code diverges from spec | Regenerate on every spec change |
| Developers edit generated files | Header comments, git hooks, CI checks |
| AI drift over time | Pin AI model version, snapshot tests |

---

## Starting Point

1. **Pick 3 components** — Button, Card, Input
2. **Write specs** — Full YAML definitions
3. **Generate once** — See if output matches current code
4. **Iterate on specs** — Refine until output is good
5. **Wire CI** — Automate the loop
6. **Expand** — Add more components
