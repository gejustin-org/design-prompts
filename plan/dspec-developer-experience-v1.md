# dspec: Developer Experience

**Focus:** How do users actually use this thing?

---

## Installation Models

### Option A: Global CLI

```bash
npm install -g dspec

# Then use anywhere
dspec init
dspec generate
```

**Pros:** Simple, no per-project setup
**Cons:** Version drift, harder to pin in CI

### Option B: Project Dependency

```bash
npm install --save-dev dspec

# Use via npx or package.json scripts
npx dspec generate
```

**Pros:** Version pinned per-project, CI-friendly
**Cons:** Extra step per project

### Option C: Both (Recommended)

```bash
# Global for quick init
npm install -g dspec
dspec init my-design-system

# Project uses local version
cd my-design-system
npm install
npm run generate  # Uses local dspec
```

---

## Project Initialization

```bash
$ dspec init my-design-system

? What would you like to generate?
  ◉ React components
  ◯ Swift components
  ◯ Kotlin components
  ◉ Figma tokens
  ◉ Design context (for AI)

? Package scope (e.g., @myorg): @handshake

? Include example specs? Yes

Creating my-design-system...
  ✓ Created dspec.config.yaml
  ✓ Created specs/tokens/colors.yaml
  ✓ Created specs/tokens/typography.yaml
  ✓ Created specs/tokens/spacing.yaml
  ✓ Created specs/components/button.yaml (example)
  ✓ Created package.json
  ✓ Created .gitignore
  ✓ Created README.md

Next steps:
  cd my-design-system
  npm install
  npm run generate
```

---

## Project Structure

```
my-design-system/
├── package.json
├── dspec.config.yaml           # Main configuration
├── .env                        # AI API keys (gitignored)
├── .gitignore
├── README.md
│
├── specs/                      # Source of truth (human-authored)
│   ├── tokens/
│   │   ├── colors.yaml
│   │   ├── typography.yaml
│   │   ├── spacing.yaml
│   │   ├── radii.yaml
│   │   └── shadows.yaml
│   ├── components/
│   │   ├── button.yaml
│   │   ├── card.yaml
│   │   ├── input.yaml
│   │   └── ...
│   ├── patterns/               # Optional
│   │   └── ...
│   └── prose/                  # Optional: philosophy, voice
│       ├── philosophy.md
│       └── anti-patterns.md
│
├── overrides/                  # Human escape hatches
│   └── react/
│       └── Button.override.tsx
│
├── dist/                       # Generated output (gitignored or committed)
│   ├── react/
│   │   ├── package.json
│   │   ├── src/
│   │   │   ├── Button/
│   │   │   ├── Card/
│   │   │   └── ...
│   │   └── dist/              # Built output
│   ├── ios/
│   ├── figma/
│   └── context/
│       ├── context.json
│       └── context.md
│
└── node_modules/
```

---

## Configuration: `dspec.config.yaml`

```yaml
# dspec.config.yaml

# Schema version
version: 1

# Design system metadata
name: rosetta
description: Handshake Design System
version: 2.4.0

# Where specs live
specs:
  tokens: ./specs/tokens
  components: ./specs/components
  patterns: ./specs/patterns
  prose: ./specs/prose

# Output targets
targets:
  react:
    enabled: true
    output: ./dist/react
    package:
      name: "@handshake/rosetta-react"
      version: inherit  # Use design system version
    options:
      styling: tailwind-cva    # or: css-modules, styled-components
      typescript: true
      tests: vitest            # or: jest
      stories: true
      
  ios:
    enabled: false
    output: ./dist/ios
    package:
      name: RosettaUI
      
  figma:
    enabled: true
    output: ./dist/figma/tokens.json
    
  context:
    enabled: true
    output: ./dist/context
    formats:
      - json
      - markdown

# AI configuration
ai:
  provider: anthropic          # or: openai, local
  model: claude-sonnet-4-20250514
  
  # What to use AI for (vs static templates)
  generate:
    components: true           # AI generates component logic
    tests: true                # AI generates test implementations
    stories: true              # AI generates story examples
    documentation: true        # AI generates prose docs
    
  # Cost controls
  cache: true                  # Cache AI responses
  cacheDir: .dspec-cache
  maxTokensPerComponent: 4000

# Validation
validation:
  typescript: true
  eslint: true
  prettier: true
  tests: true                  # Run generated tests

# Overrides
overrides:
  dir: ./overrides
  strategy: merge              # or: replace
```

---

## package.json Scripts

```json
{
  "name": "my-design-system",
  "scripts": {
    "generate": "dspec generate",
    "generate:react": "dspec generate --target react",
    "generate:static": "dspec generate --static-only",
    "validate": "dspec validate",
    "watch": "dspec generate --watch",
    "build": "dspec generate && npm run build:react",
    "build:react": "cd dist/react && npm run build",
    "publish:react": "cd dist/react && npm publish",
    "test": "dspec generate && cd dist/react && npm test"
  },
  "devDependencies": {
    "dspec": "^1.0.0"
  }
}
```

---

## AI Interaction Modes

### Mode 1: Fully Automated (Default)

AI generates, validates, outputs. No human in the loop.

```bash
$ npm run generate

Generating react target...
  ✓ Tokens (static)           12 files
  ✓ Button (ai)               4 files
  ✓ Card (ai)                 4 files
  ✓ Input (ai)                4 files
  
Validating...
  ✓ TypeScript
  ✓ ESLint
  ✓ Tests (24 passed)

Done. Generated 24 files in dist/react/
```

### Mode 2: Interactive Review

AI generates, human reviews before writing.

```bash
$ dspec generate --interactive

Generating Button...

┌─────────────────────────────────────────────────────────────────┐
│ Button.tsx (AI Generated)                                        │
├─────────────────────────────────────────────────────────────────┤
│ import * as React from 'react';                                  │
│ import { cva, type VariantProps } from 'class-variance-authority'│
│ ...                                                              │
│                                                                  │
│ [Showing first 30 lines. Press 'f' for full, 'd' for diff]      │
└─────────────────────────────────────────────────────────────────┘

? Action: (a)ccept, (r)eject, (e)dit prompt, (s)kip, (q)uit: a

✓ Button.tsx written

Generating Button.test.tsx...
```

### Mode 3: Prompt Editing

User can refine the prompt for specific components.

```bash
$ dspec generate --component Button --edit-prompt

Current prompt for Button component:
─────────────────────────────────────
You are generating a React component from a design spec...
[full prompt shown]
─────────────────────────────────────

? Edit prompt? (opens $EDITOR): y

[Editor opens, user modifies prompt]

Regenerating with updated prompt...
  ✓ Button.tsx generated

? Save prompt modification for future? y
  ✓ Saved to prompts/react/button.custom.md
```

### Mode 4: AI Chat

Interactive refinement via chat.

```bash
$ dspec chat --component Button

Starting chat for Button component...
Context: specs/components/button.yaml

You: The loading state should also show a text change to "Loading..."

AI: I'll update the Button component to change the text during loading state.
    Here's the modified implementation:
    
    ```tsx
    {loading ? (
      <>
        <Spinner className="h-4 w-4" />
        <span>Loading...</span>
      </>
    ) : (
      children
    )}
    ```
    
    Should I apply this change?

You: yes

✓ Button.tsx updated

You: /save
✓ Changes saved to dist/react/src/Button/Button.tsx

You: /exit
```

---

## Authoring Specs

### IDE Support

```bash
# Install VS Code extension
code --install-extension dspec.vscode-dspec
```

Features:
- YAML schema validation
- Autocomplete for token references (`$colors.`)
- Preview generated code
- Inline errors

### Schema Files

```bash
# Generate JSON schemas for IDE
dspec schemas --output ./schemas

# Creates:
#   schemas/tokens.schema.json
#   schemas/component.schema.json
#   schemas/pattern.schema.json
```

```yaml
# In specs/components/button.yaml
# yaml-language-server: $schema=../../schemas/component.schema.json

name: Button
# ... IDE now provides autocomplete
```

### Spec Templates

```bash
# Create new component from template
$ dspec new component dialog

? Component name: Dialog
? Has variants? Yes
? Variant names (comma-separated): default, alert, form
? Has sizes? No
? Include accessibility spec? Yes

Created specs/components/dialog.yaml

# Edit the generated spec
code specs/components/dialog.yaml
```

---

## Environment & API Keys

```bash
# .env (gitignored)
ANTHROPIC_API_KEY=sk-ant-...

# Or use environment
export ANTHROPIC_API_KEY=sk-ant-...

# Or configure in dspec.config.yaml (not recommended)
ai:
  apiKey: ${ANTHROPIC_API_KEY}  # Reference env var
```

### Local AI Option

```yaml
# dspec.config.yaml

ai:
  provider: local
  endpoint: http://localhost:11434  # Ollama
  model: codellama
```

---

## Consuming Generated Output

### As npm Dependency

```bash
# In consuming app
npm install @handshake/rosetta-react
```

```tsx
// In your app
import { Button, Card, Input } from '@handshake/rosetta-react';
import '@handshake/rosetta-react/styles.css';

function MyComponent() {
  return (
    <Card>
      <Input placeholder="Enter name" />
      <Button variant="primary">Submit</Button>
    </Card>
  );
}
```

### AI Context in Consuming App

```bash
# Install context package
npm install @handshake/rosetta-context
```

```markdown
<!-- .cursor/rules/design-system.mdc -->
---
description: Handshake Design System
globs: ["src/**/*.tsx"]
---

Follow the Rosetta design system.

@import node_modules/@handshake/rosetta-context/context.md
```

Or for Claude Code:

```markdown
<!-- CLAUDE.md -->

## Design System

When building UI, follow the Rosetta design system:

@file node_modules/@handshake/rosetta-context/context.md
```

---

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/generate.yml

name: Generate & Publish

on:
  push:
    branches: [main]
    paths:
      - 'specs/**'
      - 'dspec.config.yaml'

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          
      - name: Install
        run: npm ci
        
      - name: Generate
        run: npm run generate
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          
      - name: Validate
        run: npm run validate
        
      - name: Test
        run: npm test
        
      - name: Publish React
        if: github.ref == 'refs/heads/main'
        run: |
          cd dist/react
          npm publish
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### PR Preview

```yaml
# On PR, generate and comment with preview

- name: Generate Preview
  run: npm run generate -- --dry-run --output-summary
  
- name: Comment PR
  uses: actions/github-script@v6
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        body: `## dspec Preview\n\n${process.env.GENERATION_SUMMARY}`
      })
```

---

## Monorepo Integration

For projects where design system lives alongside app:

```
my-app/
├── apps/
│   └── web/
├── packages/
│   └── design-system/          # dspec project
│       ├── dspec.config.yaml
│       ├── specs/
│       └── dist/
│           └── react/          # Generated, used by apps/web
└── package.json
```

```json
// apps/web/package.json
{
  "dependencies": {
    "@myorg/design-system": "workspace:*"
  }
}
```

```yaml
# packages/design-system/dspec.config.yaml
targets:
  react:
    output: ./dist/react
    package:
      name: "@myorg/design-system"
```

---

## Override System

When AI doesn't get it right, humans can override.

### Override File

```tsx
// overrides/react/Button.override.tsx

// This gets merged with generated Button.tsx

// Add custom analytics
import { trackClick } from '@/lib/analytics';

// Extend the generated component
export const ButtonWithAnalytics = React.forwardRef((props, ref) => {
  const handleClick = (e) => {
    trackClick('button', props.variant);
    props.onClick?.(e);
  };
  
  return <Button {...props} ref={ref} onClick={handleClick} />;
});
```

### Override Sections

```tsx
// overrides/react/Button.sections.tsx

// Override specific sections of generated code

export const sections = {
  // Replace the loading state implementation
  loadingState: `
    {loading && (
      <div className="absolute inset-0 flex items-center justify-center">
        <Spinner />
      </div>
    )}
  `,
  
  // Add to the component body
  beforeRender: `
    const analytics = useAnalytics();
  `,
};
```

### Configuration

```yaml
# dspec.config.yaml

overrides:
  dir: ./overrides
  strategy: merge    # merge: combine with generated
                     # replace: fully replace generated
                     # sections: replace specific sections
```

---

## Workflow Summary

### Initial Setup (Once)

```bash
# 1. Install dspec
npm install -g dspec

# 2. Create project
dspec init my-design-system
cd my-design-system

# 3. Configure AI
echo "ANTHROPIC_API_KEY=sk-ant-..." > .env

# 4. Install deps
npm install
```

### Daily Workflow

```bash
# 1. Edit specs
code specs/components/button.yaml

# 2. Generate (watch mode for iteration)
npm run watch

# 3. Review output
code dist/react/src/Button/Button.tsx

# 4. Commit specs (source of truth)
git add specs/
git commit -m "feat: add ghost variant to Button"

# 5. CI generates and publishes
git push
```

### Consuming Updates

```bash
# In consuming app
npm update @handshake/rosetta-react
```

---

## Commands Reference

```bash
# Project setup
dspec init [name]              # Create new project
dspec new component [name]     # Add component spec
dspec new token [category]     # Add token category

# Generation
dspec generate                 # Generate all enabled targets
dspec generate --target react  # Generate specific target
dspec generate --static-only   # Skip AI, templates only
dspec generate --component X   # Generate one component
dspec generate --watch         # Watch specs, regenerate
dspec generate --dry-run       # Preview without writing
dspec generate --interactive   # Review each file

# AI interaction
dspec chat --component X       # Interactive refinement
dspec prompt show              # Show current prompts
dspec prompt edit [component]  # Customize prompt

# Validation
dspec validate                 # Validate all specs
dspec validate --spec X.yaml   # Validate one spec
dspec test                     # Run generated tests

# Utilities
dspec schemas                  # Generate JSON schemas for IDE
dspec diff                     # Show spec vs generated diff
dspec version                  # Show version
dspec doctor                   # Check setup, API keys, etc.
```

---

## Error Handling

### Validation Errors

```bash
$ dspec validate

specs/components/button.yaml
  ✗ Line 15: Unknown token reference "$colors.primar"
    Did you mean "$colors.primary"?
    
  ✗ Line 28: Variant "destructiv" not in allowed values
    Allowed: primary, secondary, tertiary, destructive

2 errors found. Fix specs before generating.
```

### Generation Errors

```bash
$ dspec generate

Generating Button...
  ✗ AI generation failed: rate limit exceeded
  
Retrying in 60s... (attempt 2/3)
  ✓ Button.tsx generated

Validating Button.tsx...
  ✗ TypeScript error: Property 'onClick' does not exist
  
AI response was invalid. Options:
  1. (r)etry with different prompt
  2. (e)dit prompt and retry
  3. (s)kip this component
  4. (m)anual: open in editor

? Choice: r
  ✓ Button.tsx generated (attempt 2)
  ✓ Validation passed
```

---

## Summary: DX Principles

1. **Specs are simple YAML** — No DSL to learn
2. **Sensible defaults** — Works out of the box
3. **Progressive disclosure** — Simple commands, advanced options when needed
4. **IDE-first authoring** — Schemas, autocomplete, validation
5. **AI is assistive** — Human stays in control
6. **Escape hatches exist** — Override when AI fails
7. **CI-native** — Designed for automation
8. **Debuggable** — Clear errors, dry-run, verbose modes
