# dspec CLI & Developer Experience Architecture

> Detailed specification for the dspec command-line interface, configuration system, and developer experience tooling.

---

## Table of Contents

1. [CLI Framework & Philosophy](#1-cli-framework--philosophy)
2. [Command Structure](#2-command-structure)
3. [Configuration Schema](#3-configuration-schema)
4. [Watch Mode Implementation](#4-watch-mode-implementation)
5. [IDE Integration](#5-ide-integration)
6. [Output Formatting](#6-output-formatting)
7. [Error Handling](#7-error-handling)
8. [Implementation Plan](#8-implementation-plan)

---

## 1. CLI Framework & Philosophy

### 1.1 Framework Choice: Commander.js

Based on research, **Commander.js** is the optimal choice for dspec:

| Criteria | Commander.js | Why It Wins |
|----------|--------------|-------------|
| Maturity | 50M+ weekly downloads | Battle-tested |
| TypeScript | First-class support | Type-safe commands |
| Subcommands | Native support | `dspec generate`, `dspec add` |
| Help generation | Automatic | Less maintenance |
| Ecosystem | Widely adopted | Familiar to users |

### 1.2 Distribution Model: shadcn-style

Following shadcn/ui's pioneering approach:

- **No monolithic npm package** — Components copied into user's project
- **Registry-based** — Components defined in a fetchable registry
- **Local ownership** — Users own and can modify generated code
- **Eject-friendly** — Easy to stop using dspec for any component

### 1.3 Design Principles

1. **Zero-config start** — `dspec init` gets you running
2. **Explicit over magic** — No hidden behaviors
3. **Fail fast, fail clear** — Errors explain what went wrong and how to fix
4. **Progressive disclosure** — Simple commands, advanced flags
5. **Offline-first** — Core functionality works without network

---

## 2. Command Structure

### 2.1 Command Overview

```
dspec <command> [options]

Commands:
  init          Initialize dspec in your project
  generate      Generate code from specs
  add           Add components from registry
  validate      Validate spec files
  diff          Show changes before generation
  eject         Remove dspec management from a component
  history       View generation history
  rollback      Restore previous generation
  doctor        Diagnose configuration issues

Options:
  -V, --version    Output version number
  -h, --help       Display help for command
  --verbose        Enable verbose logging
  --quiet          Suppress non-error output
  --no-color       Disable colored output
```

### 2.2 `dspec init`

Initialize dspec configuration in a project.

```
dspec init [options]

Options:
  -y, --yes              Skip prompts, use defaults
  --style <style>        Design style (neo-brutalism|minimal|saas|custom)
  --framework <fw>       Target framework (react|vue|svelte)
  --typescript           Use TypeScript (default: true)
  --no-typescript        Use JavaScript
  --tailwind             Use Tailwind CSS (default: true)
  --css-modules          Use CSS Modules instead
  --path <path>          Components directory (default: src/components)
  --force                Overwrite existing config

Examples:
  dspec init
  dspec init -y --style neo-brutalism
  dspec init --framework vue --css-modules
```

**Help Text:**
```
$ dspec init --help

Initialize dspec in your project

This command will:
  • Create dspec.config.yaml with your preferences
  • Set up the specs/ directory structure
  • Install required dependencies
  • Configure TypeScript paths (if applicable)

Options:
  -y, --yes              Skip prompts, use defaults
  --style <style>        Design style preset
                         • neo-brutalism - Bold borders, harsh shadows
                         • minimal - Clean, subtle styling
                         • saas - Modern SaaS aesthetic
                         • custom - Start from scratch
  --framework <fw>       Target framework (react|vue|svelte)
  --typescript           Use TypeScript (default: true)
  --no-typescript        Use JavaScript
  --tailwind             Use Tailwind CSS (default: true)
  --css-modules          Use CSS Modules instead
  --path <path>          Where to generate components (default: src/components)
  --force                Overwrite existing dspec.config.yaml

Examples:
  $ dspec init
  $ dspec init -y --style neo-brutalism
  $ dspec init --framework vue --no-typescript
```

### 2.3 `dspec generate`

Generate components and tokens from spec files.

```
dspec generate [specs...] [options]

Arguments:
  specs                  Spec files or patterns (default: all)

Options:
  -c, --component <n>    Generate specific component by name
  -t, --tokens-only      Generate only tokens, skip components
  --components-only      Generate only components, skip tokens
  --no-ai                Skip AI-assisted generation steps
  --cached               Use cached AI responses only
  --dry-run              Show what would be generated
  --diff                 Show diff before writing
  --force                Overwrite without confirmation
  -o, --output <dir>     Override output directory
  --watch                Watch for changes and regenerate

Examples:
  dspec generate
  dspec generate specs/button.yaml
  dspec generate --component Button --diff
  dspec generate --tokens-only
  dspec generate --watch
```

**Help Text:**
```
$ dspec generate --help

Generate code from design specifications

Reads spec files from specs/ directory (or specified paths) and generates:
  • Token files (CSS, TypeScript, Figma JSON)
  • Component files (TSX, tests, stories)
  • Type definitions
  • Barrel exports

By default, generates all specs. Use arguments or flags to filter.

Arguments:
  specs                  Specific spec files or glob patterns
                         Default: specs/**/*.yaml

Options:
  -c, --component <n>    Generate only the named component
  -t, --tokens-only      Generate only design tokens
  --components-only      Skip token generation
  --no-ai                Use static templates only (skip AI steps)
  --cached               Only use cached AI responses (fail if not cached)
  --dry-run              Preview without writing files
  --diff                 Show git-style diff before writing
  --force                Skip confirmation prompts
  -o, --output <dir>     Override configured output directory
  --watch                Watch mode - regenerate on spec changes

Generation Modes:
  Static (default)       Deterministic template-based generation
  AI-assisted            Optional AI completion for complex patterns
  Cached                 Replay previous AI responses for reproducibility

Examples:
  $ dspec generate                           # All specs
  $ dspec generate specs/button.yaml         # Single spec
  $ dspec generate --component Button --diff # Preview Button changes
  $ dspec generate --tokens-only             # Just tokens
  $ dspec generate --no-ai --force           # Static only, no prompts
  $ dspec generate --watch                   # Watch mode
```

### 2.4 `dspec add`

Add pre-built components from the registry.

```
dspec add [components...] [options]

Arguments:
  components             Component names to add

Options:
  --all                  Add all available components
  --registry <url>       Custom registry URL
  --overwrite            Overwrite existing components
  --spec-only            Add spec file only, don't generate
  --list                 List available components

Examples:
  dspec add button
  dspec add button input select
  dspec add --list
  dspec add --all --spec-only
```

**Help Text:**
```
$ dspec add --help

Add components from the dspec registry

Downloads component specifications and generates code. Components are copied
into your project - you own them and can customize freely.

Arguments:
  components             Names of components to add (space-separated)
                         Use --list to see available components

Options:
  --all                  Add all available components
  --registry <url>       Use custom registry (default: official registry)
  --overwrite            Replace existing specs and generated code
  --spec-only            Download specs without generating
  --list                 Display available components and exit

Component Dependencies:
  Some components depend on others. Dependencies are added automatically.
  Example: Select requires Popover, which requires Portal.

Examples:
  $ dspec add button                    # Add button component
  $ dspec add button input select       # Add multiple
  $ dspec add --list                    # See what's available
  $ dspec add dialog --spec-only        # Just the spec file
  $ dspec add --all                     # Everything
```

### 2.5 `dspec validate`

Validate spec files without generating.

```
dspec validate [specs...] [options]

Arguments:
  specs                  Spec files to validate (default: all)

Options:
  --strict               Treat warnings as errors
  --schema               Output JSON schema for specs
  --fix                  Auto-fix simple issues

Examples:
  dspec validate
  dspec validate specs/button.yaml --strict
  dspec validate --schema > dspec-schema.json
```

### 2.6 `dspec diff`

Preview changes before generation.

```
dspec diff [specs...] [options]

Arguments:
  specs                  Spec files to diff (default: all)

Options:
  -c, --component <n>    Diff specific component
  --stat                 Show only file statistics
  --name-only            Show only changed file names

Examples:
  dspec diff
  dspec diff --component Button
  dspec diff --stat
```

### 2.7 `dspec eject`

Remove dspec management from a component.

```
dspec eject <component> [options]

Arguments:
  component              Component name to eject

Options:
  --keep-metadata        Preserve generation metadata comments
  --keep-spec            Keep spec file (don't delete)
  --dry-run              Show what would change

Examples:
  dspec eject Button
  dspec eject Button --keep-spec
```

**Help Text:**
```
$ dspec eject --help

Remove dspec management from a component

Converts a generated component to a standalone, manually-maintained file.
After ejecting, dspec will no longer regenerate this component.

This command will:
  • Remove generation metadata comments
  • Delete the spec file (unless --keep-spec)
  • Update dspec.config.yaml to exclude component
  • Convert .gen.tsx to standard .tsx

Arguments:
  component              Name of the component to eject

Options:
  --keep-metadata        Preserve "Generated by dspec" comments
  --keep-spec            Keep the spec file for reference
  --dry-run              Preview changes without executing

After Ejecting:
  You have full ownership. Edit freely. dspec won't touch it again.
  To re-add later: delete the component and run `dspec add <component>`

Examples:
  $ dspec eject Button                  # Full eject
  $ dspec eject Button --keep-spec      # Keep spec for reference
  $ dspec eject Button --dry-run        # Preview only
```

### 2.8 `dspec history`

View generation history for components.

```
dspec history [component] [options]

Arguments:
  component              Component name (optional)

Options:
  -n, --limit <n>        Number of entries to show (default: 10)
  --all                  Show all history
  --json                 Output as JSON

Examples:
  dspec history
  dspec history Button
  dspec history Button -n 20
```

### 2.9 `dspec rollback`

Restore a previous generation.

```
dspec rollback <component> [options]

Arguments:
  component              Component to rollback

Options:
  --to <hash>            Rollback to specific generation hash
  --steps <n>            Rollback n generations (default: 1)
  --all                  Rollback all components
  --dry-run              Preview rollback

Examples:
  dspec rollback Button
  dspec rollback Button --to abc123
  dspec rollback --all --to 2025-03-15
```

### 2.10 `dspec doctor`

Diagnose configuration and environment issues.

```
dspec doctor [options]

Options:
  --fix                  Attempt to fix issues automatically

Examples:
  dspec doctor
  dspec doctor --fix
```

**Sample Output:**
```
$ dspec doctor

dspec Doctor

Checking environment...
  ✓ Node.js v22.0.0 (>=18 required)
  ✓ TypeScript 5.4.0 installed
  ✓ Tailwind CSS 3.4.0 configured

Checking configuration...
  ✓ dspec.config.yaml found
  ✓ Output directory exists (src/components)
  ✓ Specs directory exists (specs/)
  
Checking specs...
  ✓ 12 spec files found
  ✓ All specs valid
  ⚠ 2 specs have deprecation warnings
    └ specs/badge.yaml: 'styling.focus' deprecated, use 'accessibility.focusRing'
    └ specs/input.yaml: 'styling.focus' deprecated, use 'accessibility.focusRing'

Checking generated code...
  ✓ 10/12 components up to date
  ⚠ 2 components have pending changes
    └ Button: spec modified since last generation
    └ Input: spec modified since last generation

Summary: 2 warnings, 0 errors
Run 'dspec generate' to regenerate outdated components.
Run 'dspec doctor --fix' to auto-fix deprecation warnings.
```

---

## 3. Configuration Schema

### 3.1 Full Configuration File: `dspec.config.yaml`

```yaml
# dspec.config.yaml
# Configuration for dspec design system generator
# JSON Schema: https://dspec.dev/schemas/config.json

# Schema version for migrations
schemaVersion: 2

# Project information
project:
  name: "my-design-system"
  description: "Company design system built with dspec"
  version: "1.0.0"

# Framework and language settings
framework:
  target: react                    # react | vue | svelte
  typescript: true                 # Generate TypeScript
  jsx: preserve                    # preserve | react-jsx | react-jsxdev

# Styling configuration
styling:
  engine: tailwind                 # tailwind | css-modules | styled-components | vanilla
  tailwind:
    config: tailwind.config.js     # Path to Tailwind config
    css: src/styles/globals.css    # Path to global CSS
    prefix: ""                     # Optional class prefix
  theme:
    cssVariables: true             # Use CSS custom properties
    darkMode: class                # class | media | false
    
# Design style preset
style:
  preset: neo-brutalism            # neo-brutalism | minimal | saas | custom
  # Custom style overrides (when preset: custom)
  custom:
    borderRadius: "0"
    borderWidth: "3px"
    shadowStyle: "hard"
    fontWeight: "bold"

# Directory paths
paths:
  specs: specs/                    # Spec files location
  output: src/components/          # Generated components
  tokens: src/tokens/              # Generated tokens
  stories: src/stories/            # Storybook stories (optional)
  tests: __tests__/                # Test files (optional, co-located if omitted)

# Import aliases (for generated code)
aliases:
  "@/components": src/components
  "@/lib": src/lib
  "@/tokens": src/tokens
  utils: "@/lib/utils"             # cn() helper location

# Token generation settings
tokens:
  outputs:
    - format: css                  # CSS custom properties
      file: tokens.css
    - format: typescript           # TypeScript constants
      file: tokens.ts
    - format: figma                # Figma Tokens JSON
      file: figma-tokens.json
    - format: scss                 # SCSS variables
      file: _tokens.scss
      enabled: false               # Disabled by default
  
  # Token prefixes
  prefix:
    css: "--ds"                    # CSS: --ds-color-primary
    ts: "DS"                       # TS: DS_COLOR_PRIMARY

# Component generation settings
components:
  # File naming convention
  naming:
    component: PascalCase          # Button.tsx
    file: PascalCase               # Button/Button.tsx
    directory: PascalCase          # Button/
    test: PascalCase.test          # Button.test.tsx
    story: PascalCase.stories      # Button.stories.tsx
    
  # Generated file structure
  structure: directory             # directory | flat
  # directory: Button/Button.tsx, Button/index.ts
  # flat: Button.tsx (no directory)
  
  # What to generate per component
  generate:
    component: true
    types: true                    # Separate types file
    tests: true
    stories: true
    index: true                    # Barrel export
    
  # Generation approach
  mode: static                     # static | ai-assisted
  
  # Co-location preferences  
  colocate:
    tests: true                    # Tests next to components
    stories: true                  # Stories next to components
    types: false                   # Types in component file

# AI generation settings (when mode: ai-assisted)
ai:
  enabled: false                   # Master toggle
  provider: anthropic              # anthropic | openai | local
  model: claude-sonnet-4-20250514           # Model identifier
  temperature: 0                   # Determinism (0 = most deterministic)
  
  # Caching for reproducibility
  cache:
    enabled: true
    directory: .dspec/cache
    
  # What AI assists with
  assist:
    variants: true                 # CVA variant implementations
    tests: true                    # Test implementations  
    stories: true                  # Story examples
    documentation: false           # JSDoc comments
    
  # Cost controls
  limits:
    maxTokensPerGeneration: 4000
    maxCostPerMonth: 100           # USD

# Quality gates
quality:
  # Pre-generation validation
  validate:
    specs: true                    # Validate specs before generation
    strict: false                  # Treat warnings as errors
    
  # Post-generation checks
  checks:
    typescript: true               # Type check generated code
    eslint: true                   # Lint generated code
    prettier: true                 # Format generated code
    
  # Test requirements
  tests:
    coverage: 80                   # Minimum coverage %
    required: true                 # Fail if tests don't pass

# Watch mode settings
watch:
  enabled: true
  debounce: 300                    # ms to wait before regenerating
  clearScreen: true                # Clear terminal on rebuild
  ignored:                         # Patterns to ignore
    - "**/node_modules/**"
    - "**/.git/**"
    - "**/*.gen.tsx"               # Don't watch generated files

# Registry settings
registry:
  url: https://registry.dspec.dev # Official registry
  # Custom/private registries
  sources:
    - name: internal
      url: https://dspec.company.com/registry
      auth: token                  # none | token | basic

# Logging and output
output:
  colors: true                     # Colored terminal output
  emoji: true                      # Use emoji in output
  verbosity: normal                # quiet | normal | verbose | debug
  
  # Progress reporting
  progress:
    style: spinner                 # spinner | bar | dots | none
    
  # Diff display
  diff:
    context: 3                     # Lines of context
    colorScheme: github            # github | terminal

# Hooks (scripts to run at various points)
hooks:
  preGenerate: ""                  # Before any generation
  postGenerate: ""                 # After all generation
  preComponent: ""                 # Before each component
  postComponent: ""                # After each component
  # Example: postGenerate: "npm run lint:fix"

# Experimental features
experimental:
  parallelGeneration: false        # Generate components in parallel
  incrementalBuild: false          # Only regenerate changed specs
  figmaBidirectional: false        # Two-way Figma sync
```

### 3.2 Minimal Configuration

For simple projects, a minimal config works:

```yaml
# dspec.config.yaml (minimal)
schemaVersion: 2
framework:
  target: react
style:
  preset: minimal
paths:
  output: src/components
```

### 3.3 Configuration Precedence

1. CLI flags (highest priority)
2. Environment variables (`DSPEC_*`)
3. `dspec.config.yaml`
4. Default values (lowest priority)

```bash
# Environment variable examples
DSPEC_AI_ENABLED=true
DSPEC_AI_MODEL=claude-sonnet-4-20250514
DSPEC_OUTPUT_VERBOSITY=debug
```

---

## 4. Watch Mode Implementation

### 4.1 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Watch Mode System                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────┐ │
│  │   chokidar   │────▶│   Debouncer  │────▶│  Generator  │ │
│  │  (watcher)   │     │   (300ms)    │     │   Queue     │ │
│  └──────────────┘     └──────────────┘     └─────────────┘ │
│         │                                         │         │
│         │                                         ▼         │
│         │                                 ┌─────────────┐   │
│         │                                 │  Validator  │   │
│         │                                 └─────────────┘   │
│         │                                         │         │
│         │                                         ▼         │
│         │                                 ┌─────────────┐   │
│         │                                 │  Generator  │   │
│         │                                 └─────────────┘   │
│         │                                         │         │
│         ▼                                         ▼         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    Terminal UI                        │  │
│  │  ┌─────────┐  ┌──────────┐  ┌───────────────────┐   │  │
│  │  │ Status  │  │ Progress │  │  Error Display    │   │  │
│  │  └─────────┘  └──────────┘  └───────────────────┘   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 File Watching Strategy

```typescript
// lib/watch/watcher.ts
import chokidar from 'chokidar';
import { debounce } from './debounce';

interface WatchOptions {
  paths: string[];
  ignored: string[];
  debounceMs: number;
  onChange: (files: string[]) => Promise<void>;
}

export function createWatcher(options: WatchOptions) {
  const pendingFiles = new Set<string>();
  
  const processChanges = debounce(async () => {
    const files = Array.from(pendingFiles);
    pendingFiles.clear();
    await options.onChange(files);
  }, options.debounceMs);
  
  const watcher = chokidar.watch(options.paths, {
    ignored: options.ignored,
    persistent: true,
    ignoreInitial: true,
    awaitWriteFinish: {
      stabilityThreshold: 100,
      pollInterval: 50,
    },
  });
  
  watcher
    .on('add', (path) => { pendingFiles.add(path); processChanges(); })
    .on('change', (path) => { pendingFiles.add(path); processChanges(); })
    .on('unlink', (path) => { pendingFiles.add(path); processChanges(); });
    
  return {
    close: () => watcher.close(),
  };
}
```

### 4.3 Watch Mode UI

```typescript
// lib/watch/ui.ts
import ora from 'ora';
import chalk from 'chalk';

export class WatchUI {
  private spinner = ora();
  private lastBuild: Date | null = null;
  
  start() {
    this.clear();
    this.header();
    this.spinner.start('Watching for changes...');
  }
  
  private clear() {
    if (process.stdout.isTTY) {
      console.clear();
    }
  }
  
  private header() {
    console.log(chalk.bold.cyan('dspec') + chalk.dim(' watch mode'));
    console.log(chalk.dim('─'.repeat(40)));
    console.log();
  }
  
  building(files: string[]) {
    this.spinner.text = `Regenerating ${files.length} spec(s)...`;
  }
  
  success(duration: number, stats: BuildStats) {
    this.spinner.succeed(
      chalk.green(`Built in ${duration}ms`) +
      chalk.dim(` (${stats.components} components, ${stats.tokens} tokens)`)
    );
    this.lastBuild = new Date();
    this.spinner.start('Watching for changes...');
  }
  
  error(err: Error) {
    this.spinner.fail(chalk.red('Build failed'));
    console.log();
    console.log(chalk.red(err.message));
    console.log();
    this.spinner.start('Watching for changes...');
  }
  
  status() {
    if (this.lastBuild) {
      console.log(chalk.dim(`Last build: ${this.lastBuild.toLocaleTimeString()}`));
    }
  }
}
```

### 4.4 Watch Mode Output Example

```
$ dspec generate --watch

dspec watch mode
────────────────────────────────────

✔ Built in 234ms (3 components, 24 tokens)
⠋ Watching for changes...

[10:32:15] specs/button.yaml changed
✔ Built in 156ms (1 component)
⠋ Watching for changes...

[10:33:42] specs/tokens/colors.yaml changed  
✔ Built in 312ms (0 components, 24 tokens)
  └ Regenerated 5 dependent components
⠋ Watching for changes...

[10:35:01] specs/input.yaml changed
✖ Build failed

  Error: Invalid spec at specs/input.yaml:23
  
    22 │   variants:
  > 23 │     size: [sm, md]    # ← Expected object, got array
    24 │   defaultVariants:
  
  Fix the error and save to retry.

⠋ Watching for changes...
```

### 4.5 Keyboard Shortcuts

```
Watch Mode Commands:
  r, Enter    Force rebuild all
  c           Clear screen
  o           Open output directory
  q, Ctrl+C   Quit watch mode
  h           Show help
```

---

## 5. IDE Integration

### 5.1 JSON Schema for Autocomplete

Generate a JSON Schema for spec files to enable IDE autocomplete:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://dspec.dev/schemas/component-spec.json",
  "title": "dspec Component Specification",
  "description": "Schema for dspec component YAML files",
  "type": "object",
  "required": ["name", "props"],
  "properties": {
    "name": {
      "type": "string",
      "description": "Component name in PascalCase",
      "pattern": "^[A-Z][a-zA-Z0-9]*$"
    },
    "description": {
      "type": "string",
      "description": "Component description for documentation"
    },
    "category": {
      "type": "string",
      "enum": ["primitive", "composite", "layout", "feedback", "navigation"],
      "description": "Component category for organization"
    },
    "props": {
      "type": "array",
      "description": "Component props definition",
      "items": {
        "$ref": "#/definitions/prop"
      }
    },
    "variants": {
      "type": "object",
      "description": "CVA variant definitions",
      "additionalProperties": {
        "$ref": "#/definitions/variant"
      }
    },
    "defaultVariants": {
      "type": "object",
      "description": "Default variant values",
      "additionalProperties": {
        "type": "string"
      }
    },
    "compoundVariants": {
      "type": "array",
      "description": "Compound variant combinations",
      "items": {
        "$ref": "#/definitions/compoundVariant"
      }
    },
    "slots": {
      "type": "object",
      "description": "Named slots for compound components",
      "additionalProperties": {
        "$ref": "#/definitions/slot"
      }
    },
    "styling": {
      "$ref": "#/definitions/styling"
    },
    "accessibility": {
      "$ref": "#/definitions/accessibility"
    },
    "tokens": {
      "type": "object",
      "description": "Token references used by this component",
      "additionalProperties": {
        "type": "string",
        "pattern": "^\\$"
      }
    }
  },
  "definitions": {
    "prop": {
      "type": "object",
      "required": ["name", "type"],
      "properties": {
        "name": {
          "type": "string",
          "description": "Prop name"
        },
        "type": {
          "type": "string",
          "enum": ["string", "number", "boolean", "enum", "ReactNode", "function"],
          "description": "Prop type"
        },
        "values": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Enum values (when type is enum)"
        },
        "default": {
          "description": "Default value"
        },
        "required": {
          "type": "boolean",
          "default": false
        },
        "description": {
          "type": "string"
        }
      }
    },
    "variant": {
      "type": "object",
      "additionalProperties": {
        "oneOf": [
          { "type": "string" },
          { "type": "array", "items": { "type": "string" } }
        ]
      }
    },
    "compoundVariant": {
      "type": "object",
      "properties": {
        "conditions": {
          "type": "object",
          "additionalProperties": { "type": "string" }
        },
        "class": {
          "oneOf": [
            { "type": "string" },
            { "type": "array", "items": { "type": "string" } }
          ]
        }
      }
    },
    "slot": {
      "type": "object",
      "properties": {
        "element": { "type": "string" },
        "props": {
          "type": "array",
          "items": { "$ref": "#/definitions/prop" }
        },
        "styling": { "$ref": "#/definitions/styling" }
      }
    },
    "styling": {
      "type": "object",
      "properties": {
        "base": {
          "oneOf": [
            { "type": "string" },
            { "type": "array", "items": { "type": "string" } }
          ]
        },
        "variants": {
          "type": "object"
        }
      }
    },
    "accessibility": {
      "type": "object",
      "properties": {
        "role": { "type": "string" },
        "focusRing": {
          "type": "object",
          "properties": {
            "style": { "type": "string", "enum": ["ring", "outline", "shadow"] },
            "color": { "type": "string" },
            "width": { "type": "string" }
          }
        },
        "ariaLabel": { "type": "string" },
        "keyboardNav": { "type": "boolean" }
      }
    }
  }
}
```

### 5.2 VS Code Integration

**`.vscode/settings.json`:**
```json
{
  "yaml.schemas": {
    "https://dspec.dev/schemas/component-spec.json": "specs/components/**/*.yaml",
    "https://dspec.dev/schemas/token-spec.json": "specs/tokens/**/*.yaml",
    "https://dspec.dev/schemas/config.json": "dspec.config.yaml"
  },
  "yaml.customTags": [
    "!ref scalar",
    "!token scalar"
  ]
}
```

**VS Code Extension Features (Future):**
- Real-time spec validation
- Go to definition for token references
- Preview generated code
- Inline color swatches for color tokens
- Component preview pane

### 5.3 JetBrains Integration

**`.idea/jsonSchemas.xml`:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project version="4">
  <component name="JsonSchemaMappingsProjectConfiguration">
    <state>
      <map>
        <entry key="dspec-component">
          <value>
            <SchemaInfo>
              <option name="name" value="dspec Component Spec" />
              <option name="relativePathToSchema" value="https://dspec.dev/schemas/component-spec.json" />
              <option name="patterns">
                <list>
                  <Item>
                    <option name="path" value="specs/components/**/*.yaml" />
                  </Item>
                </list>
              </option>
            </SchemaInfo>
          </value>
        </entry>
      </map>
    </state>
  </component>
</project>
```

### 5.4 Schema Versioning

```yaml
# In spec files, reference schema version
# yaml-language-server: $schema=https://dspec.dev/schemas/v2/component-spec.json

name: Button
# ...
```

---

## 6. Output Formatting

### 6.1 Progress Display

```typescript
// lib/output/progress.ts
import ora from 'ora';
import chalk from 'chalk';

type ProgressStyle = 'spinner' | 'bar' | 'dots' | 'none';

export class ProgressReporter {
  private spinner = ora();
  private style: ProgressStyle;
  
  constructor(style: ProgressStyle = 'spinner') {
    this.style = style;
  }
  
  start(message: string) {
    if (this.style === 'none') return;
    this.spinner.start(message);
  }
  
  update(message: string) {
    this.spinner.text = message;
  }
  
  succeed(message: string) {
    this.spinner.succeed(chalk.green(message));
  }
  
  fail(message: string) {
    this.spinner.fail(chalk.red(message));
  }
  
  info(message: string) {
    this.spinner.info(chalk.blue(message));
  }
  
  warn(message: string) {
    this.spinner.warn(chalk.yellow(message));
  }
}
```

### 6.2 Generation Output Examples

**Successful Generation:**
```
$ dspec generate

  dspec v1.2.0

  ◐ Validating specs...
  ✓ Validated 12 specs

  ◐ Generating tokens...
  ✓ Generated tokens
    ├── src/tokens/tokens.css (2.1 KB)
    ├── src/tokens/tokens.ts (1.8 KB)
    └── src/tokens/figma-tokens.json (3.2 KB)

  ◐ Generating components...
  ✓ Generated 10 components
    ├── Button (new)
    ├── Input (updated)
    ├── Card (unchanged, skipped)
    └── ... 7 more

  ────────────────────────────────────────

  Summary:
    Tokens:     3 files (7.1 KB)
    Components: 10 generated, 2 unchanged
    Time:       1.23s

  ✨ Done!
```

**Generation with Warnings:**
```
$ dspec generate

  dspec v1.2.0

  ⚠ Validating specs...
  ⚠ Validated 12 specs with 2 warnings:
    │
    │ specs/badge.yaml:15
    │   warning: 'styling.focus' is deprecated, use 'accessibility.focusRing'
    │
    │ specs/input.yaml:42  
    │   warning: Missing description for prop 'onChange'
    │

  ◐ Generating tokens...
  ✓ Generated tokens (3 files)

  ◐ Generating components...
  ✓ Generated 10 components

  ────────────────────────────────────────

  Summary:
    Tokens:     3 files
    Components: 10 generated
    Warnings:   2
    Time:       1.45s

  ⚠ Completed with warnings. Run with --strict to treat warnings as errors.
```

**Failed Generation:**
```
$ dspec generate

  dspec v1.2.0

  ✖ Validation failed

  Error in specs/button.yaml:

    22 │   variants:
    23 │     variant:
  > 24 │       primary: [bg-blue-500]    ← Expected string, got array
    25 │       secondary: "bg-gray-200"
    26 │

  Expected: string
  Received: array

  Hint: Variant values should be strings, not arrays.
        Use space-separated classes: "bg-blue-500 text-white"

  ────────────────────────────────────────

  ✖ Generation failed. Fix errors and try again.
```

### 6.3 Diff Output

```
$ dspec diff --component Button

  Diff for Button

  ── src/components/Button/Button.tsx ───────────────────────

    @@ -12,7 +12,9 @@ const buttonVariants = cva(
         variant: {
           default: "bg-primary text-primary-foreground",
           secondary: "bg-secondary text-secondary-foreground",
    -      destructive: "bg-destructive text-destructive-foreground",
    +      destructive: "bg-red-500 text-white hover:bg-red-600",
    +      ghost: "hover:bg-accent hover:text-accent-foreground",
    +      link: "text-primary underline-offset-4 hover:underline",
         },
       },
     );

  ── src/components/Button/Button.test.tsx ──────────────────

    @@ -15,6 +15,18 @@ describe("Button", () => {
       it("renders destructive variant", () => {
    -    render(<Button variant="destructive">Delete</Button>);
    +    render(<Button variant="destructive">Delete</Button>);
    +    expect(screen.getByRole("button")).toHaveClass("bg-red-500");
    +  });
    +
    +  it("renders ghost variant", () => {
    +    render(<Button variant="ghost">Ghost</Button>);
    +    expect(screen.getByRole("button")).toHaveClass("hover:bg-accent");
       });

  ────────────────────────────────────────

  Files: 2 changed
  Insertions: +12  Deletions: -2

  Run 'dspec generate --component Button' to apply changes.
```

### 6.4 Statistics Output

```
$ dspec diff --stat

  Diff Summary

  src/components/Button/Button.tsx       | 12 ++++++--
  src/components/Button/Button.test.tsx  |  8 ++++++
  src/components/Input/Input.tsx         |  4 +--
  src/tokens/tokens.css                  |  6 ++--

  4 files changed, 22 insertions(+), 6 deletions(-)
```

### 6.5 JSON Output Mode

For scripting and CI integration:

```
$ dspec generate --json

{
  "success": true,
  "duration": 1234,
  "tokens": {
    "generated": 3,
    "files": [
      { "path": "src/tokens/tokens.css", "size": 2134 },
      { "path": "src/tokens/tokens.ts", "size": 1823 },
      { "path": "src/tokens/figma-tokens.json", "size": 3245 }
    ]
  },
  "components": {
    "generated": 10,
    "unchanged": 2,
    "items": [
      { "name": "Button", "status": "created", "files": 4 },
      { "name": "Input", "status": "updated", "files": 4 }
    ]
  },
  "warnings": [],
  "errors": []
}
```

---

## 7. Error Handling

### 7.1 Error Categories

```typescript
// lib/errors/types.ts

export enum ErrorCode {
  // Configuration errors (1xx)
  CONFIG_NOT_FOUND = 'E101',
  CONFIG_INVALID = 'E102',
  CONFIG_SCHEMA_MISMATCH = 'E103',
  
  // Spec errors (2xx)
  SPEC_NOT_FOUND = 'E201',
  SPEC_PARSE_ERROR = 'E202',
  SPEC_VALIDATION_ERROR = 'E203',
  SPEC_REFERENCE_ERROR = 'E204',
  
  // Generation errors (3xx)
  GEN_TEMPLATE_ERROR = 'E301',
  GEN_OUTPUT_ERROR = 'E302',
  GEN_AI_ERROR = 'E303',
  GEN_CONFLICT_ERROR = 'E304',
  
  // Quality gate errors (4xx)
  QA_TYPESCRIPT_ERROR = 'E401',
  QA_LINT_ERROR = 'E402',
  QA_TEST_ERROR = 'E403',
  
  // Registry errors (5xx)
  REGISTRY_UNREACHABLE = 'E501',
  REGISTRY_COMPONENT_NOT_FOUND = 'E502',
  REGISTRY_AUTH_ERROR = 'E503',
}

export interface DspecError {
  code: ErrorCode;
  message: string;
  file?: string;
  line?: number;
  column?: number;
  hint?: string;
  docs?: string;
}
```

### 7.2 Error Display Format

```typescript
// lib/errors/formatter.ts
import chalk from 'chalk';

export function formatError(error: DspecError): string {
  const lines: string[] = [];
  
  // Error header
  lines.push(chalk.red.bold(`Error ${error.code}: ${error.message}`));
  lines.push('');
  
  // Location (if available)
  if (error.file) {
    lines.push(chalk.dim(`  at ${error.file}${error.line ? `:${error.line}` : ''}`));
    lines.push('');
    
    // Code snippet with context
    if (error.line) {
      const snippet = getCodeSnippet(error.file, error.line, 3);
      lines.push(snippet);
      lines.push('');
    }
  }
  
  // Hint
  if (error.hint) {
    lines.push(chalk.yellow('Hint: ') + error.hint);
    lines.push('');
  }
  
  // Documentation link
  if (error.docs) {
    lines.push(chalk.dim(`Docs: ${error.docs}`));
  }
  
  return lines.join('\n');
}
```

### 7.3 Common Error Examples

**Missing Configuration:**
```
Error E101: Configuration file not found

  Could not find dspec.config.yaml in current directory or parents.

Hint: Run 'dspec init' to create a configuration file.

Docs: https://dspec.dev/docs/configuration
```

**Invalid Spec Syntax:**
```
Error E202: Failed to parse spec file

  at specs/button.yaml:15:3

    13 │ props:
    14 │   - name: variant
  > 15 │     type enum        ← Missing colon
    16 │     values:
    17 │       - primary

Hint: YAML requires colons after keys. Change 'type enum' to 'type: enum'

Docs: https://dspec.dev/docs/spec-format
```

**Token Reference Error:**
```
Error E204: Unknown token reference

  at specs/button.yaml:24:21

    22 │ styling:
    23 │   base:
  > 24 │     background: $colors.brand.primary
                         ^^^^^^^^^^^^^^^^^^^^^^
    25 │

  Token '$colors.brand.primary' does not exist.

  Did you mean one of these?
    • $colors.primary
    • $colors.brand-primary
    • $colors.interactive.primary

Hint: Check specs/tokens/colors.yaml for available color tokens.

Docs: https://dspec.dev/docs/tokens#references
```

**AI Generation Failure:**
```
Error E303: AI generation failed

  Component: Button
  Step: variants
  
  The AI provider returned an error:
    Rate limit exceeded. Please retry after 60 seconds.
    
  Generation will continue with static templates.

Hint: Use --no-ai to skip AI generation, or --cached to use cached responses.

Docs: https://dspec.dev/docs/ai-generation#troubleshooting
```

### 7.4 Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Configuration error |
| 3 | Spec validation error |
| 4 | Generation error |
| 5 | Quality gate failure |
| 130 | User interrupt (Ctrl+C) |

---

## 8. Implementation Plan

### 8.1 Phase 1: Core CLI (Week 1 of Phase 1)

**Priority Commands:**
1. `dspec init` — Basic config generation
2. `dspec generate` — Token generation only
3. `dspec validate` — Spec validation

**Dependencies:**
```json
{
  "commander": "^12.0.0",
  "chalk": "^5.3.0",
  "ora": "^8.0.0",
  "yaml": "^2.4.0",
  "zod": "^3.22.0",
  "handlebars": "^4.7.0",
  "chokidar": "^3.6.0"
}
```

### 8.2 Phase 2: Extended CLI (Week 2-3 of Phase 1)

**Additional Commands:**
1. `dspec diff` — Preview changes
2. `dspec doctor` — Environment diagnostics
3. Watch mode (`--watch`)

### 8.3 Phase 3: Registry CLI (Phase 2+)

**Commands:**
1. `dspec add` — Add from registry
2. `dspec eject` — Remove dspec management

### 8.4 Phase 4: Advanced CLI (Phase 4)

**Commands:**
1. `dspec history` — Generation history
2. `dspec rollback` — Restore previous generation

### 8.5 File Structure

```
packages/dspec-cli/
├── src/
│   ├── commands/
│   │   ├── init.ts
│   │   ├── generate.ts
│   │   ├── add.ts
│   │   ├── validate.ts
│   │   ├── diff.ts
│   │   ├── eject.ts
│   │   ├── history.ts
│   │   ├── rollback.ts
│   │   └── doctor.ts
│   ├── lib/
│   │   ├── config/
│   │   │   ├── loader.ts
│   │   │   ├── schema.ts
│   │   │   └── defaults.ts
│   │   ├── specs/
│   │   │   ├── parser.ts
│   │   │   ├── validator.ts
│   │   │   └── resolver.ts
│   │   ├── generator/
│   │   │   ├── tokens.ts
│   │   │   ├── components.ts
│   │   │   └── templates.ts
│   │   ├── watch/
│   │   │   ├── watcher.ts
│   │   │   └── ui.ts
│   │   ├── output/
│   │   │   ├── progress.ts
│   │   │   ├── diff.ts
│   │   │   └── format.ts
│   │   └── errors/
│   │       ├── types.ts
│   │       └── formatter.ts
│   ├── schemas/
│   │   ├── config.json
│   │   ├── component-spec.json
│   │   └── token-spec.json
│   └── index.ts
├── templates/
│   ├── component.hbs
│   ├── story.hbs
│   ├── test.hbs
│   └── index.hbs
├── package.json
└── tsconfig.json
```

---

## Appendix A: Complete Help Text Reference

```
$ dspec --help

dspec - Design System Specification Compiler

Usage: dspec <command> [options]

Commands:
  init              Initialize dspec in your project
  generate          Generate code from design specifications
  add               Add components from the dspec registry
  validate          Validate spec files
  diff              Preview changes before generation
  eject             Remove dspec management from a component
  history           View generation history
  rollback          Restore a previous generation
  doctor            Diagnose configuration issues

Options:
  -V, --version     Output version number
  -h, --help        Display help for command
  --verbose         Enable verbose logging
  --quiet           Suppress non-error output
  --no-color        Disable colored output
  --json            Output as JSON (for scripting)

Examples:
  $ dspec init                        # Set up dspec in your project
  $ dspec generate                    # Generate all specs
  $ dspec add button input            # Add components from registry
  $ dspec generate --watch            # Watch mode

Documentation: https://dspec.dev/docs
Report issues: https://github.com/dspec/dspec/issues
```

---

## Appendix B: Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DSPEC_CONFIG` | Path to config file | `./dspec.config.yaml` |
| `DSPEC_AI_ENABLED` | Enable AI generation | `false` |
| `DSPEC_AI_MODEL` | AI model to use | `claude-sonnet-4-20250514` |
| `DSPEC_AI_API_KEY` | AI provider API key | — |
| `DSPEC_REGISTRY_URL` | Registry URL | `https://registry.dspec.dev` |
| `DSPEC_REGISTRY_TOKEN` | Registry auth token | — |
| `DSPEC_OUTPUT_VERBOSITY` | Log level | `normal` |
| `DSPEC_NO_COLOR` | Disable colors | `false` |
| `DSPEC_CACHE_DIR` | Cache directory | `.dspec/cache` |
| `CI` | Detected CI environment | — |

---

## Appendix C: CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/dspec.yml
name: dspec

on:
  push:
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
          node-version: '22'
          
      - run: npm ci
      
      - name: Validate specs
        run: npx dspec validate --strict
        
      - name: Generate
        run: npx dspec generate --no-ai
        
      - name: Check for changes
        run: |
          if [ -n "$(git status --porcelain)" ]; then
            echo "::error::Generated files are out of date. Run 'dspec generate' locally."
            exit 1
          fi
```

### Pre-commit Hook

```bash
#!/bin/sh
# .husky/pre-commit

npx dspec validate --strict
```
