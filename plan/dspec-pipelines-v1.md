# dspec: Pipeline Definition

**Question:** How do users define what "react" or any target pipeline actually does?

---

## The Model

A **pipeline** is a sequence of generators that transform specs into output files.

```
┌─────────────┐     ┌────────────────────────────────────────────────────┐
│   SPECS     │ ──▶ │                    PIPELINE                         │
│             │     │                                                      │
│ • tokens    │     │  ┌─────────┐   ┌─────────┐   ┌─────────┐           │
│ • components│     │  │ Step 1  │ → │ Step 2  │ → │ Step 3  │ → ...     │
│ • patterns  │     │  │ tokens  │   │ types   │   │component│           │
│             │     │  └─────────┘   └─────────┘   └─────────┘           │
└─────────────┘     └──────────────────────────────┬─────────────────────┘
                                                   │
                                                   ▼
                                    ┌─────────────────────────────┐
                                    │          OUTPUT              │
                                    │                              │
                                    │  dist/react/                 │
                                    │  ├── tokens.css              │
                                    │  ├── types.ts                │
                                    │  ├── Button/Button.tsx       │
                                    │  └── ...                     │
                                    └─────────────────────────────┘
```

---

## Built-in vs Custom

dspec ships with **built-in pipelines** for common targets:
- `react` — React + TypeScript + Tailwind/CVA
- `react-native` — React Native
- `swift` — SwiftUI
- `kotlin` — Jetpack Compose
- `figma` — Figma Tokens format
- `context` — AI-optimized documentation

Users can:
1. **Use built-ins as-is** — Zero config
2. **Extend built-ins** — Override specific steps
3. **Define custom pipelines** — Full control

---

## Pipeline Definition

### Location

```
my-design-system/
├── dspec.config.yaml         # References pipelines
├── pipelines/                 # Custom pipelines
│   ├── react.yaml            # Override built-in
│   └── my-custom.yaml        # Fully custom
└── ...
```

### Structure

```yaml
# pipelines/react.yaml

name: react
description: React components with TypeScript and Tailwind CSS

# What specs this pipeline consumes
inputs:
  - tokens      # required
  - components  # required
  - patterns    # optional
  - prose       # optional

# Output configuration
output:
  dir: ./dist/react
  clean: true   # Delete before generating

# Package.json for the generated package
package:
  name: "{{ config.package.name }}"
  version: "{{ config.package.version }}"
  main: ./dist/index.js
  types: ./dist/index.d.ts
  dependencies:
    class-variance-authority: "^0.7.0"
  peerDependencies:
    react: "^18.0.0"

# The actual generation steps
steps:
  # ─────────────────────────────────────────────────────────────
  # TOKENS
  # ─────────────────────────────────────────────────────────────
  
  - name: tokens-css
    type: static
    description: Generate CSS custom properties
    input: tokens
    template: react/tokens.css.hbs
    output: src/tokens.css
    
  - name: tokens-ts
    type: static
    description: Generate TypeScript token constants
    input: tokens
    template: react/tokens.ts.hbs
    output: src/tokens.ts

  # ─────────────────────────────────────────────────────────────
  # COMPONENTS
  # ─────────────────────────────────────────────────────────────
  
  - name: component
    type: ai
    description: Generate React component
    input: component          # Runs once per component
    prompt: react/component.md
    output: "src/{{ component.name }}/{{ component.name }}.tsx"
    validate:
      - typescript
      - eslint
      
  - name: component-types
    type: static
    description: Generate component props interface
    input: component
    template: react/component-types.ts.hbs
    output: "src/{{ component.name }}/types.ts"
    
  - name: component-test
    type: ai
    description: Generate component tests
    input: component
    prompt: react/component-test.md
    output: "src/{{ component.name }}/{{ component.name }}.test.tsx"
    context:
      # Pass the generated component code to the test prompt
      componentCode: "{{ steps.component.output }}"
      
  - name: component-story
    type: ai
    description: Generate Storybook stories
    input: component
    prompt: react/component-story.md
    output: "src/{{ component.name }}/{{ component.name }}.stories.tsx"
    
  - name: component-index
    type: static
    description: Generate component barrel export
    input: component
    template: react/component-index.ts.hbs
    output: "src/{{ component.name }}/index.ts"

  # ─────────────────────────────────────────────────────────────
  # PACKAGE
  # ─────────────────────────────────────────────────────────────
  
  - name: main-index
    type: static
    description: Generate main package exports
    input: components         # All components
    template: react/index.ts.hbs
    output: src/index.ts
    
  - name: package-json
    type: static
    description: Generate package.json
    input: meta
    template: react/package.json.hbs
    output: package.json
    
  - name: tsconfig
    type: static
    description: Generate TypeScript config
    template: react/tsconfig.json.hbs
    output: tsconfig.json

# Post-generation hooks
hooks:
  afterGenerate:
    - npm install
    - npm run build
  afterValidate:
    - npm test
```

---

## Step Types

### Static Step

Uses Handlebars templates. Deterministic, fast.

```yaml
- name: tokens-css
  type: static
  template: react/tokens.css.hbs
  input: tokens
  output: src/tokens.css
```

Template:
```handlebars
{{! templates/react/tokens.css.hbs }}

/* Generated by dspec — DO NOT EDIT */

:root {
{{#each tokens.colors}}
  {{#each this}}
  --color-{{@../key}}-{{@key}}: {{value}};
  {{/each}}
{{/each}}

{{#each tokens.spacing}}
  --spacing-{{@key}}: {{this}}px;
{{/each}}
}
```

### AI Step

Uses prompts + LLM. For semantic generation.

```yaml
- name: component
  type: ai
  prompt: react/component.md
  input: component
  output: "src/{{ component.name }}/{{ component.name }}.tsx"
  options:
    model: claude-sonnet-4-20250514   # Override default
    temperature: 0.2
    maxTokens: 4000
  validate:
    - typescript
    - eslint
  retry:
    attempts: 3
    onFailure: skip  # or: error, manual
```

Prompt file:
```markdown
{{! prompts/react/component.md }}

You are generating a React component from a design specification.

## Component Specification

```yaml
{{ yaml component }}
```

## Available Tokens

```yaml
{{ yaml tokens }}
```

## Requirements

1. Use TypeScript with strict types
2. Use `forwardRef` for ref forwarding
3. Use `cva` from class-variance-authority for variants
4. Use CSS variables for token references: `var(--color-...)`
5. Implement all states: {{ component.states | join ", " }}
6. Include accessibility attributes from spec

## Output

Return ONLY the TypeScript code. No explanations.
The component should be production-ready.
```

### Script Step

Run arbitrary code for complex transformations.

```yaml
- name: custom-transform
  type: script
  script: scripts/custom-transform.ts
  input: components
  output: src/custom/
```

```typescript
// scripts/custom-transform.ts
import { StepContext, GeneratedFile } from 'dspec';

export default async function(ctx: StepContext): Promise<GeneratedFile[]> {
  const { components, tokens } = ctx.inputs;
  
  // Custom logic
  const files: GeneratedFile[] = [];
  
  for (const component of components) {
    files.push({
      path: `src/custom/${component.name}.tsx`,
      content: generateCustom(component, tokens),
    });
  }
  
  return files;
}
```

---

## Extending Built-in Pipelines

### Override Specific Steps

```yaml
# pipelines/react.yaml

extends: builtin:react    # Start with built-in

# Override specific steps
steps:
  # Replace the component step with custom prompt
  - name: component
    prompt: ./prompts/my-component.md   # Custom prompt
    
  # Add a new step
  - name: analytics-wrapper
    type: static
    template: ./templates/analytics.hbs
    output: "src/{{ component.name }}/analytics.tsx"
    after: component   # Run after component step
    
  # Disable a step
  - name: component-story
    enabled: false     # Don't generate stories
```

### Add Steps

```yaml
extends: builtin:react

steps:
  # Add custom documentation step
  - name: component-docs
    type: ai
    prompt: ./prompts/docs.md
    output: "src/{{ component.name }}/README.md"
    after: component
```

### Change Templates

```yaml
extends: builtin:react

# Override template directory
templates:
  dir: ./my-templates/react/
  
# Now uses ./my-templates/react/tokens.css.hbs instead of built-in
```

---

## Custom Pipelines from Scratch

```yaml
# pipelines/my-custom.yaml

name: my-custom
description: Custom output format

inputs:
  - tokens
  - components

output:
  dir: ./dist/custom

steps:
  - name: manifest
    type: static
    template: custom/manifest.json.hbs
    output: manifest.json
    
  - name: styles
    type: script
    script: scripts/generate-styles.ts
    output: styles/
    
  - name: components
    type: ai
    prompt: custom/component.md
    input: component
    output: "components/{{ component.name }}.js"
```

---

## Template System

### Directory Structure

```
templates/
├── react/                    # Built-in React templates
│   ├── tokens.css.hbs
│   ├── tokens.ts.hbs
│   ├── component-types.ts.hbs
│   ├── component-index.ts.hbs
│   ├── index.ts.hbs
│   └── package.json.hbs
├── swift/                    # Built-in Swift templates
│   └── ...
└── custom/                   # User's custom templates
    └── ...
```

### Template Helpers

```handlebars
{{! Built-in helpers }}

{{! Convert to camelCase }}
{{ camelCase component.name }}

{{! Convert to kebab-case }}
{{ kebabCase component.name }}

{{! Convert to PascalCase }}
{{ pascalCase component.name }}

{{! YAML dump }}
{{ yaml component }}

{{! JSON dump }}
{{ json component }}

{{! Join array }}
{{ join component.variants ", " }}

{{! Conditional }}
{{#if component.hasIcon}}
import { Icon } from '../Icon';
{{/if}}

{{! Loop }}
{{#each component.variants}}
  '{{ name }}': '{{ classes }}',
{{/each}}

{{! Token reference }}
{{ tokenRef "colors.interactive.primary" }}
→ var(--color-interactive-primary)
```

### Custom Helpers

```typescript
// dspec.config.yaml or templates/helpers.ts

export const helpers = {
  // Custom helper
  rosettaClass(token: string) {
    return `rosetta-${token.replace(/\./g, '-')}`;
  },
  
  // Use in template: {{ rosettaClass "colors.primary" }}
  // Output: rosetta-colors-primary
};
```

---

## Prompt System

### Directory Structure

```
prompts/
├── react/
│   ├── component.md          # Component generation
│   ├── component-test.md     # Test generation
│   ├── component-story.md    # Story generation
│   └── _partials/
│       ├── requirements.md   # Shared requirements
│       └── examples.md       # Shared examples
├── swift/
│   └── ...
└── shared/
    ├── accessibility.md      # Cross-platform a11y knowledge
    └── patterns.md           # Common patterns
```

### Prompt Composition

```markdown
{{! prompts/react/component.md }}

# Component Generation

{{ include "shared/role.md" }}

## Specification

```yaml
{{ yaml component }}
```

## Tokens

```yaml
{{ yaml tokens }}
```

{{ include "react/_partials/requirements.md" }}

{{ include "shared/accessibility.md" }}

## Output

Generate the component:
```

### Prompt Variables

```yaml
# In pipeline step
- name: component
  type: ai
  prompt: react/component.md
  input: component
  variables:
    # Extra variables passed to prompt template
    projectName: "{{ config.name }}"
    styleSystem: tailwind
    testFramework: vitest
```

```markdown
{{! In prompt }}
Project: {{ projectName }}
Style system: {{ styleSystem }}
```

---

## Configuration Reference

### dspec.config.yaml

```yaml
version: 1

name: my-design-system
version: 1.0.0

specs:
  tokens: ./specs/tokens
  components: ./specs/components

# Pipeline targets
targets:
  react:
    enabled: true
    # Use built-in with overrides
    pipeline: builtin:react
    output: ./dist/react
    package:
      name: "@myorg/design-system"
      version: inherit
    options:
      styling: tailwind-cva
      tests: vitest
      stories: true
      
  ios:
    enabled: true
    pipeline: builtin:swift
    output: ./dist/ios
    
  custom:
    enabled: true
    # Use custom pipeline
    pipeline: ./pipelines/custom.yaml
    output: ./dist/custom

# AI configuration (applies to all pipelines)
ai:
  provider: anthropic
  model: claude-sonnet-4-20250514
  temperature: 0.2
  cache: true

# Template overrides
templates:
  dir: ./templates           # Custom templates
  helpers: ./templates/helpers.ts

# Prompt overrides  
prompts:
  dir: ./prompts             # Custom prompts

# Global validation
validation:
  typescript: true
  eslint: true
  prettier: true
```

---

## CLI Pipeline Commands

```bash
# List available pipelines
dspec pipelines list

# Show pipeline definition
dspec pipelines show react

# Validate pipeline definition
dspec pipelines validate ./pipelines/custom.yaml

# Run specific pipeline
dspec generate --target react

# Run specific step
dspec generate --target react --step tokens-css

# Debug pipeline (show what would run)
dspec generate --target react --dry-run --verbose
```

---

## Example: Minimal Custom Pipeline

```yaml
# pipelines/simple-css.yaml

name: simple-css
description: Just generate CSS tokens

inputs:
  - tokens

output:
  dir: ./dist/css

steps:
  - name: variables
    type: static
    template: |
      :root {
      {{#each tokens.colors.background}}
        --bg-{{@key}}: {{value}};
      {{/each}}
      }
    output: variables.css
```

```bash
dspec generate --target simple-css
# Output: dist/css/variables.css
```

---

## Example: Full React Pipeline

```yaml
# This is approximately what builtin:react looks like

name: react
description: React components with TypeScript, Tailwind CSS, and CVA

inputs:
  - tokens
  - components
  - patterns
  - prose

output:
  dir: "{{ config.output }}"
  clean: true

package:
  name: "{{ config.package.name }}"
  version: "{{ config.package.version }}"
  main: ./dist/index.js
  module: ./dist/index.mjs
  types: ./dist/index.d.ts
  exports:
    ".":
      import: ./dist/index.mjs
      require: ./dist/index.js
      types: ./dist/index.d.ts
    "./styles.css":
      import: ./dist/tokens.css
  peerDependencies:
    react: "^18.0.0"
    react-dom: "^18.0.0"
  dependencies:
    class-variance-authority: "^0.7.0"
    clsx: "^2.0.0"
    tailwind-merge: "^2.0.0"

steps:
  # Tokens
  - name: tokens-css
    type: static
    input: tokens
    template: react/tokens.css.hbs
    output: src/tokens.css
    
  - name: tokens-ts
    type: static
    input: tokens
    template: react/tokens.ts.hbs
    output: src/lib/tokens.ts
    
  # Utils
  - name: cn-util
    type: static
    template: react/cn.ts.hbs
    output: src/lib/cn.ts

  # Components (runs for each)
  - name: component
    type: ai
    input: component
    prompt: react/component.md
    output: "src/{{ pascalCase component.name }}/{{ pascalCase component.name }}.tsx"
    validate: [typescript, eslint]
    retry: { attempts: 3 }
    
  - name: component-index
    type: static
    input: component
    template: react/component-index.ts.hbs
    output: "src/{{ pascalCase component.name }}/index.ts"
    
  - name: component-test
    type: ai
    input: component
    prompt: react/component-test.md
    output: "src/{{ pascalCase component.name }}/{{ pascalCase component.name }}.test.tsx"
    context:
      componentCode: "{{ steps.component.output }}"
    optional: true   # Don't fail if tests generation fails
    
  - name: component-story
    type: ai
    input: component
    prompt: react/component-story.md
    output: "src/{{ pascalCase component.name }}/{{ pascalCase component.name }}.stories.tsx"
    optional: true

  # Package files
  - name: main-index
    type: static
    input: components
    template: react/index.ts.hbs
    output: src/index.ts
    
  - name: package-json
    type: static
    template: react/package.json.hbs
    output: package.json
    
  - name: tsconfig
    type: static
    template: react/tsconfig.json.hbs
    output: tsconfig.json
    
  - name: readme
    type: ai
    input: [components, tokens, prose]
    prompt: react/readme.md
    output: README.md

hooks:
  afterGenerate:
    - command: npm install
      cwd: "{{ output.dir }}"
    - command: npm run build
      cwd: "{{ output.dir }}"
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Pipeline** | Sequence of steps that transform specs → output |
| **Step** | Single generation unit (static, ai, or script) |
| **Template** | Handlebars file for static generation |
| **Prompt** | Markdown file for AI generation |
| **Built-in** | Pipelines that ship with dspec |
| **Extends** | Inherit and override built-in pipelines |
| **Custom** | Define pipelines from scratch |
