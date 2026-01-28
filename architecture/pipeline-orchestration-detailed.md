# Pipeline Orchestration Architecture

> Detailed architecture specification for dspec's pipeline system.  
> **Key decision:** Handlebars for templates, custom orchestration for steps.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Pipeline YAML Schema](#2-pipeline-yaml-schema)
3. [Step Execution Engine](#3-step-execution-engine)
4. [Built-in Pipelines](#4-built-in-pipelines)
5. [Template System](#5-template-system)
6. [Dependency Graph](#6-dependency-graph)
7. [Hooks System](#7-hooks-system)
8. [Error Handling](#8-error-handling)
9. [CLI Integration](#9-cli-integration)

---

## 1. Overview

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PIPELINE ORCHESTRATOR                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │  YAML    │───▶│   Schema     │───▶│  Dependency  │───▶│   Step       │  │
│  │  Parser  │    │   Validator  │    │  Resolver    │    │   Scheduler  │  │
│  └──────────┘    └──────────────┘    └──────────────┘    └──────────────┘  │
│                                                                    │        │
│                                                                    ▼        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        STEP EXECUTION ENGINE                          │  │
│  │                                                                        │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐      │  │
│  │  │  Static    │  │    AI      │  │  Script    │  │  Validate  │      │  │
│  │  │  Executor  │  │  Executor  │  │  Executor  │  │  Executor  │      │  │
│  │  │            │  │            │  │            │  │            │      │  │
│  │  │ Handlebars │  │  Prompts   │  │ TypeScript │  │ ts-morph   │      │  │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                         │                                   │
│                                         ▼                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                          OUTPUT MANAGER                               │  │
│  │                                                                        │  │
│  │  • File writing with conflict detection                               │  │
│  │  • Generation metadata injection                                       │  │
│  │  • Override merging                                                    │  │
│  │  • Hook execution                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Core Principles

1. **Deterministic first** — Static steps are pure functions; same input = same output
2. **Fail fast** — Validation errors stop the pipeline immediately
3. **Resumable** — Failed AI steps can be retried without re-running successful steps
4. **Observable** — Every step logs its inputs, outputs, and timing
5. **Extensible** — Custom steps via script executor

---

## 2. Pipeline YAML Schema

### Complete Schema Specification

```yaml
# JSON Schema: https://dspec.dev/schemas/pipeline-v1.json

# ═══════════════════════════════════════════════════════════════════════════
# PIPELINE METADATA
# ═══════════════════════════════════════════════════════════════════════════

name: string                    # Required: Unique identifier
version: string                 # Optional: Semantic version (default: "1.0.0")
description: string             # Optional: Human-readable description
extends: string                 # Optional: Base pipeline (e.g., "builtin:react")

# ═══════════════════════════════════════════════════════════════════════════
# INPUTS
# ═══════════════════════════════════════════════════════════════════════════

inputs:
  - tokens                      # Token definitions (required for most)
  - components                  # Component specs
  - patterns                    # Pattern library
  - prose                       # Documentation content

# ═══════════════════════════════════════════════════════════════════════════
# OUTPUT CONFIGURATION
# ═══════════════════════════════════════════════════════════════════════════

output:
  dir: string                   # Required: Output directory (supports templates)
  clean: boolean                # Optional: Delete dir before generating (default: false)
  preservePatterns:             # Optional: Glob patterns to preserve during clean
    - "**/*.override.tsx"
    - "**/node_modules/**"

# ═══════════════════════════════════════════════════════════════════════════
# PACKAGE CONFIGURATION
# ═══════════════════════════════════════════════════════════════════════════

package:
  name: string                  # NPM package name
  version: string               # Package version (or "inherit" from config)
  description: string           # Package description
  main: string                  # CommonJS entry
  module: string                # ESM entry
  types: string                 # TypeScript declarations
  exports: object               # Package exports map
  dependencies: object          # Runtime dependencies
  devDependencies: object       # Development dependencies
  peerDependencies: object      # Peer dependencies
  sideEffects: boolean | array  # Side effects declaration

# ═══════════════════════════════════════════════════════════════════════════
# TEMPLATE CONFIGURATION
# ═══════════════════════════════════════════════════════════════════════════

templates:
  dir: string                   # Custom templates directory
  helpers: string               # Path to custom helpers module
  partials: string              # Path to partials directory

# ═══════════════════════════════════════════════════════════════════════════
# PROMPT CONFIGURATION
# ═══════════════════════════════════════════════════════════════════════════

prompts:
  dir: string                   # Custom prompts directory
  
# ═══════════════════════════════════════════════════════════════════════════
# AI CONFIGURATION
# ═══════════════════════════════════════════════════════════════════════════

ai:
  provider: string              # "anthropic" | "openai" | "ollama"
  model: string                 # Model identifier
  temperature: number           # 0-1, default: 0 for determinism
  maxTokens: number             # Maximum response tokens
  cache: boolean                # Enable response caching (default: true)
  cacheTTL: number              # Cache TTL in seconds (default: 86400)

# ═══════════════════════════════════════════════════════════════════════════
# STEPS
# ═══════════════════════════════════════════════════════════════════════════

steps:
  - name: string                # Required: Unique step identifier
    type: string                # Required: "static" | "ai" | "script" | "validate"
    description: string         # Optional: Human-readable description
    enabled: boolean            # Optional: Enable/disable step (default: true)
    
    # Input specification
    input: string | string[]    # What spec(s) this step consumes
                                # Single: "tokens", "component" (runs per item)
                                # Array: ["tokens", "components"] (all at once)
                                # "component" singular = runs for each component
                                # "components" plural = runs once with all
    
    # Output specification
    output: string              # Output file path (supports templates)
    
    # Step ordering
    after: string | string[]    # Run after these step(s)
    before: string | string[]   # Run before these step(s)
    dependsOn: string[]         # Explicit dependencies (outputs needed)
    
    # Execution control
    condition: string           # Template expression for conditional execution
    optional: boolean           # Don't fail pipeline if this step fails
    parallel: boolean           # Can run in parallel with other parallel steps
    
    # Type-specific configuration (see below)
    template: string            # For static: Handlebars template path or inline
    prompt: string              # For ai: Prompt file path
    script: string              # For script: TypeScript file path
    validator: string           # For validate: Validator name or path
    
    # Context injection
    context: object             # Additional variables for template/prompt
    variables: object           # Alias for context
    
    # AI-specific options
    options:
      model: string             # Override pipeline AI model
      temperature: number       # Override temperature
      maxTokens: number         # Override max tokens
      systemPrompt: string      # Custom system prompt
    
    # Validation
    validate: string[]          # Post-generation validators
    
    # Retry configuration
    retry:
      attempts: number          # Max retry attempts (default: 3)
      delayMs: number           # Delay between retries (default: 1000)
      backoff: string           # "linear" | "exponential" (default: "exponential")
      onFailure: string         # "error" | "skip" | "manual" (default: "error")

# ═══════════════════════════════════════════════════════════════════════════
# HOOKS
# ═══════════════════════════════════════════════════════════════════════════

hooks:
  beforeGenerate: HookAction[]
  afterGenerate: HookAction[]
  beforeStep: HookAction[]
  afterStep: HookAction[]
  onError: HookAction[]
  afterValidate: HookAction[]
  beforePublish: HookAction[]
  afterPublish: HookAction[]

# HookAction can be:
# - string: Shell command
# - object: { command: string, cwd: string, env: object, condition: string }
```

### TypeScript Type Definitions

```typescript
// src/types/pipeline.ts

export interface Pipeline {
  name: string;
  version?: string;
  description?: string;
  extends?: string;
  
  inputs: InputType[];
  output: OutputConfig;
  package?: PackageConfig;
  templates?: TemplateConfig;
  prompts?: PromptConfig;
  ai?: AIConfig;
  steps: Step[];
  hooks?: Hooks;
}

export type InputType = 'tokens' | 'components' | 'patterns' | 'prose';

export interface OutputConfig {
  dir: string;
  clean?: boolean;
  preservePatterns?: string[];
}

export interface PackageConfig {
  name: string;
  version: string;
  description?: string;
  main?: string;
  module?: string;
  types?: string;
  exports?: Record<string, ExportConfig>;
  dependencies?: Record<string, string>;
  devDependencies?: Record<string, string>;
  peerDependencies?: Record<string, string>;
  sideEffects?: boolean | string[];
}

export interface ExportConfig {
  import?: string;
  require?: string;
  types?: string;
}

export interface TemplateConfig {
  dir?: string;
  helpers?: string;
  partials?: string;
}

export interface PromptConfig {
  dir?: string;
}

export interface AIConfig {
  provider: 'anthropic' | 'openai' | 'ollama';
  model: string;
  temperature?: number;
  maxTokens?: number;
  cache?: boolean;
  cacheTTL?: number;
}

export type StepType = 'static' | 'ai' | 'script' | 'validate';

export interface Step {
  name: string;
  type: StepType;
  description?: string;
  enabled?: boolean;
  
  // Input
  input?: string | string[];
  
  // Output
  output?: string;
  
  // Ordering
  after?: string | string[];
  before?: string | string[];
  dependsOn?: string[];
  
  // Execution
  condition?: string;
  optional?: boolean;
  parallel?: boolean;
  
  // Type-specific
  template?: string;
  prompt?: string;
  script?: string;
  validator?: string;
  
  // Context
  context?: Record<string, unknown>;
  variables?: Record<string, unknown>;
  
  // AI options
  options?: AIStepOptions;
  
  // Validation
  validate?: string[];
  
  // Retry
  retry?: RetryConfig;
}

export interface AIStepOptions {
  model?: string;
  temperature?: number;
  maxTokens?: number;
  systemPrompt?: string;
}

export interface RetryConfig {
  attempts?: number;
  delayMs?: number;
  backoff?: 'linear' | 'exponential';
  onFailure?: 'error' | 'skip' | 'manual';
}

export interface Hooks {
  beforeGenerate?: HookAction[];
  afterGenerate?: HookAction[];
  beforeStep?: HookAction[];
  afterStep?: HookAction[];
  onError?: HookAction[];
  afterValidate?: HookAction[];
  beforePublish?: HookAction[];
  afterPublish?: HookAction[];
}

export type HookAction = string | HookActionConfig;

export interface HookActionConfig {
  command: string;
  cwd?: string;
  env?: Record<string, string>;
  condition?: string;
  timeout?: number;
}
```

---

## 3. Step Execution Engine

### Execution Lifecycle

```
┌────────────────────────────────────────────────────────────────────────────┐
│                          STEP EXECUTION LIFECYCLE                           │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐      │
│  │ PENDING │──▶│ QUEUED  │──▶│ RUNNING │──▶│VALIDATING│──▶│COMPLETE │      │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘      │
│       │             │             │              │             │           │
│       │             │             │              │             │           │
│       │             │             ▼              ▼             │           │
│       │             │        ┌─────────┐   ┌─────────┐        │           │
│       │             │        │ FAILED  │   │ INVALID │        │           │
│       │             │        └─────────┘   └─────────┘        │           │
│       │             │             │              │             │           │
│       │             │             ▼              │             │           │
│       │             │        ┌─────────┐        │             │           │
│       │             │        │ RETRYING│────────┘             │           │
│       │             │        └─────────┘                      │           │
│       │             │             │                           │           │
│       ▼             │             ▼                           │           │
│  ┌─────────┐        │        ┌─────────┐                      │           │
│  │ SKIPPED │◀───────┴────────│ ABORTED │                      │           │
│  └─────────┘                 └─────────┘                      │           │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### State Definitions

| State | Description |
|-------|-------------|
| `PENDING` | Step registered, waiting for dependencies |
| `QUEUED` | Dependencies satisfied, ready to execute |
| `RUNNING` | Currently executing |
| `VALIDATING` | Execution complete, running validators |
| `COMPLETE` | Successfully finished |
| `FAILED` | Execution failed (may retry) |
| `INVALID` | Validation failed |
| `RETRYING` | Attempting retry after failure |
| `SKIPPED` | Skipped due to condition or disabled |
| `ABORTED` | Manually stopped or pipeline failed |

### Step Context

```typescript
// src/engine/context.ts

export interface StepContext {
  // Identity
  step: Step;
  pipeline: Pipeline;
  
  // Inputs
  inputs: {
    tokens?: TokenSpec;
    components?: ComponentSpec[];
    component?: ComponentSpec;      // When iterating
    patterns?: PatternSpec[];
    prose?: ProseSpec[];
    config: DspecConfig;
  };
  
  // Previous step outputs
  steps: Record<string, StepOutput>;
  
  // Variables
  variables: Record<string, unknown>;
  
  // Utilities
  helpers: TemplateHelpers;
  
  // State
  state: {
    attempt: number;
    startTime: Date;
    previousErrors: Error[];
  };
  
  // Methods
  log: (level: LogLevel, message: string, meta?: object) => void;
  emit: (event: string, data: unknown) => void;
  getOutput: (stepName: string) => StepOutput | undefined;
  resolveTemplate: (template: string) => string;
}

export interface StepOutput {
  files: GeneratedFile[];
  metadata: {
    duration: number;
    cached: boolean;
    hash: string;
  };
}

export interface GeneratedFile {
  path: string;
  content: string;
  hash: string;
  metadata: GenerationMetadata;
}

export interface GenerationMetadata {
  generator: string;       // "dspec"
  version: string;         // dspec version
  specHash: string;        // Hash of input spec
  templateHash?: string;   // Hash of template (static)
  promptHash?: string;     // Hash of prompt (ai)
  modelVersion?: string;   // AI model version
  generationHash: string;  // Hash of output
  timestamp: string;       // ISO timestamp
}
```

### Executor Interface

```typescript
// src/engine/executors/base.ts

export interface StepExecutor {
  type: StepType;
  
  /**
   * Validate step configuration before execution
   */
  validate(step: Step, pipeline: Pipeline): ValidationResult;
  
  /**
   * Execute the step and return generated files
   */
  execute(ctx: StepContext): Promise<StepResult>;
  
  /**
   * Clean up resources (optional)
   */
  cleanup?(ctx: StepContext): Promise<void>;
}

export interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
  warnings: ValidationWarning[];
}

export interface StepResult {
  success: boolean;
  files: GeneratedFile[];
  errors?: Error[];
  warnings?: string[];
  metrics?: {
    duration: number;
    tokensUsed?: number;
    cached?: boolean;
  };
}
```

### Static Executor

```typescript
// src/engine/executors/static.ts

import Handlebars from 'handlebars';
import { StepExecutor, StepContext, StepResult } from './base';

export class StaticExecutor implements StepExecutor {
  type = 'static' as const;
  
  private handlebars: typeof Handlebars;
  private templateCache: Map<string, HandlebarsTemplateDelegate>;
  
  constructor(config: TemplateConfig) {
    this.handlebars = Handlebars.create();
    this.templateCache = new Map();
    this.registerHelpers(config.helpers);
    this.registerPartials(config.partials);
  }
  
  validate(step: Step): ValidationResult {
    const errors: ValidationError[] = [];
    
    if (!step.template) {
      errors.push({ code: 'MISSING_TEMPLATE', message: 'Static step requires template' });
    }
    
    if (!step.output) {
      errors.push({ code: 'MISSING_OUTPUT', message: 'Static step requires output path' });
    }
    
    return { valid: errors.length === 0, errors, warnings: [] };
  }
  
  async execute(ctx: StepContext): Promise<StepResult> {
    const { step, inputs, variables, helpers } = ctx;
    
    // Load template
    const template = await this.loadTemplate(step.template!);
    
    // Build template context
    const templateContext = {
      ...inputs,
      ...variables,
      ...ctx.context,
      config: inputs.config,
      steps: ctx.steps,
    };
    
    // Determine iteration
    const items = this.getIterationItems(step, inputs);
    const files: GeneratedFile[] = [];
    
    for (const item of items) {
      const itemContext = { ...templateContext, ...item };
      
      // Render template
      const content = template(itemContext);
      
      // Resolve output path
      const outputPath = this.resolvePath(step.output!, itemContext);
      
      // Generate file
      files.push({
        path: outputPath,
        content,
        hash: this.hash(content),
        metadata: this.buildMetadata(step, itemContext),
      });
    }
    
    return { success: true, files };
  }
  
  private getIterationItems(step: Step, inputs: StepContext['inputs']): object[] {
    if (!step.input) return [{}];
    
    // Singular "component" means iterate
    if (step.input === 'component') {
      return (inputs.components || []).map(c => ({ component: c }));
    }
    
    // Plural "components" means all at once
    if (step.input === 'components') {
      return [{ components: inputs.components }];
    }
    
    // Other inputs
    return [{}];
  }
  
  private async loadTemplate(templatePath: string): Promise<HandlebarsTemplateDelegate> {
    // Check for inline template
    if (templatePath.includes('\n') || templatePath.includes('{{')) {
      return this.handlebars.compile(templatePath);
    }
    
    // Check cache
    if (this.templateCache.has(templatePath)) {
      return this.templateCache.get(templatePath)!;
    }
    
    // Load from file
    const content = await fs.readFile(templatePath, 'utf-8');
    const compiled = this.handlebars.compile(content);
    this.templateCache.set(templatePath, compiled);
    return compiled;
  }
  
  private registerHelpers(helpersPath?: string): void {
    // Built-in helpers
    this.handlebars.registerHelper('pascalCase', pascalCase);
    this.handlebars.registerHelper('camelCase', camelCase);
    this.handlebars.registerHelper('kebabCase', kebabCase);
    this.handlebars.registerHelper('snakeCase', snakeCase);
    this.handlebars.registerHelper('constantCase', constantCase);
    this.handlebars.registerHelper('json', (obj) => JSON.stringify(obj, null, 2));
    this.handlebars.registerHelper('yaml', (obj) => yaml.dump(obj));
    this.handlebars.registerHelper('join', (arr, sep) => arr.join(sep));
    this.handlebars.registerHelper('tokenRef', (path) => `var(--${path.replace(/\./g, '-')})`);
    this.handlebars.registerHelper('eq', (a, b) => a === b);
    this.handlebars.registerHelper('ne', (a, b) => a !== b);
    this.handlebars.registerHelper('includes', (arr, val) => arr?.includes(val));
    
    // Custom helpers
    if (helpersPath) {
      const customHelpers = require(helpersPath);
      for (const [name, fn] of Object.entries(customHelpers)) {
        this.handlebars.registerHelper(name, fn as HelperDelegate);
      }
    }
  }
}
```

### AI Executor

```typescript
// src/engine/executors/ai.ts

import { StepExecutor, StepContext, StepResult } from './base';
import { AIProvider, createProvider } from '../providers';

export class AIExecutor implements StepExecutor {
  type = 'ai' as const;
  
  private provider: AIProvider;
  private cache: ResponseCache;
  private handlebars: typeof Handlebars;
  
  constructor(config: AIConfig, cache: ResponseCache) {
    this.provider = createProvider(config);
    this.cache = cache;
    this.handlebars = Handlebars.create();
  }
  
  async execute(ctx: StepContext): Promise<StepResult> {
    const { step, inputs, variables } = ctx;
    
    // Load and compile prompt
    const promptTemplate = await this.loadPrompt(step.prompt!);
    
    // Determine iteration
    const items = this.getIterationItems(step, inputs);
    const files: GeneratedFile[] = [];
    const errors: Error[] = [];
    
    for (const item of items) {
      const itemContext = {
        ...inputs,
        ...variables,
        ...step.context,
        ...item,
        steps: ctx.steps,
      };
      
      // Render prompt
      const prompt = promptTemplate(itemContext);
      
      // Check cache
      const cacheKey = this.buildCacheKey(prompt, step.options);
      const cached = await this.cache.get(cacheKey);
      
      let content: string;
      let fromCache = false;
      
      if (cached) {
        content = cached;
        fromCache = true;
        ctx.log('debug', `Using cached response for ${step.name}`);
      } else {
        // Call AI
        try {
          content = await this.provider.generate({
            prompt,
            model: step.options?.model,
            temperature: step.options?.temperature,
            maxTokens: step.options?.maxTokens,
            systemPrompt: step.options?.systemPrompt,
          });
          
          // Cache response
          await this.cache.set(cacheKey, content);
        } catch (error) {
          errors.push(error as Error);
          
          if (!step.optional) {
            return { success: false, files, errors };
          }
          continue;
        }
      }
      
      // Extract code from response
      const code = this.extractCode(content);
      
      // Resolve output path
      const outputPath = this.resolvePath(step.output!, itemContext);
      
      files.push({
        path: outputPath,
        content: code,
        hash: this.hash(code),
        metadata: this.buildMetadata(step, itemContext, fromCache),
      });
    }
    
    return {
      success: errors.length === 0 || step.optional,
      files,
      errors: errors.length > 0 ? errors : undefined,
      metrics: {
        duration: Date.now() - ctx.state.startTime.getTime(),
        cached: files.some(f => f.metadata.cached),
      },
    };
  }
  
  private extractCode(response: string): string {
    // Extract code from markdown code blocks
    const codeBlockMatch = response.match(/```(?:typescript|tsx|ts|jsx|js)?\n([\s\S]*?)```/);
    if (codeBlockMatch) {
      return codeBlockMatch[1].trim();
    }
    
    // Return raw response if no code block
    return response.trim();
  }
  
  private buildCacheKey(prompt: string, options?: AIStepOptions): string {
    const keyData = {
      prompt: this.hash(prompt),
      model: options?.model || this.provider.defaultModel,
      temperature: options?.temperature ?? 0,
    };
    return this.hash(JSON.stringify(keyData));
  }
}
```

### Script Executor

```typescript
// src/engine/executors/script.ts

import { StepExecutor, StepContext, StepResult, GeneratedFile } from './base';

export interface ScriptModule {
  default: (ctx: StepContext) => Promise<GeneratedFile[]>;
}

export class ScriptExecutor implements StepExecutor {
  type = 'script' as const;
  
  private moduleCache: Map<string, ScriptModule>;
  
  constructor() {
    this.moduleCache = new Map();
  }
  
  validate(step: Step): ValidationResult {
    const errors: ValidationError[] = [];
    
    if (!step.script) {
      errors.push({ code: 'MISSING_SCRIPT', message: 'Script step requires script path' });
    }
    
    return { valid: errors.length === 0, errors, warnings: [] };
  }
  
  async execute(ctx: StepContext): Promise<StepResult> {
    const { step } = ctx;
    
    try {
      // Load script module
      const module = await this.loadScript(step.script!);
      
      // Execute
      const files = await module.default(ctx);
      
      return {
        success: true,
        files: files.map(f => ({
          ...f,
          metadata: f.metadata || this.buildMetadata(step),
        })),
      };
    } catch (error) {
      return {
        success: false,
        files: [],
        errors: [error as Error],
      };
    }
  }
  
  private async loadScript(scriptPath: string): Promise<ScriptModule> {
    if (this.moduleCache.has(scriptPath)) {
      return this.moduleCache.get(scriptPath)!;
    }
    
    // Use tsx/ts-node for TypeScript
    const module = await import(scriptPath);
    this.moduleCache.set(scriptPath, module);
    return module;
  }
}
```

---

## 4. Built-in Pipelines

### builtin:react — Complete Specification

```yaml
# pipelines/builtin/react.yaml
# Full specification for the built-in React pipeline

name: react
version: "1.0.0"
description: |
  React components with TypeScript, Tailwind CSS, and class-variance-authority (CVA).
  Generates a complete publishable package with components, stories, and tests.

# ═══════════════════════════════════════════════════════════════════════════
# INPUTS
# ═══════════════════════════════════════════════════════════════════════════

inputs:
  - tokens        # Required: Design tokens
  - components    # Required: Component specifications
  - patterns      # Optional: Pattern library
  - prose         # Optional: Documentation

# ═══════════════════════════════════════════════════════════════════════════
# OUTPUT CONFIGURATION
# ═══════════════════════════════════════════════════════════════════════════

output:
  dir: "{{ config.targets.react.output | default: './dist/react' }}"
  clean: true
  preservePatterns:
    - "**/*.override.tsx"
    - "**/*.override.ts"
    - "**/node_modules/**"
    - "**/.git/**"

# ═══════════════════════════════════════════════════════════════════════════
# PACKAGE CONFIGURATION
# ═══════════════════════════════════════════════════════════════════════════

package:
  name: "{{ config.package.name | default: '@org/design-system' }}"
  version: "{{ config.package.version | default: '1.0.0' }}"
  description: "{{ config.package.description | default: 'Design system components' }}"
  main: ./dist/index.js
  module: ./dist/index.mjs
  types: ./dist/index.d.ts
  exports:
    ".":
      import: ./dist/index.mjs
      require: ./dist/index.js
      types: ./dist/index.d.ts
    "./styles.css":
      import: ./dist/styles.css
    "./tokens":
      import: ./dist/tokens.mjs
      require: ./dist/tokens.js
      types: ./dist/tokens.d.ts
  files:
    - dist
    - README.md
  sideEffects:
    - "**/*.css"
  peerDependencies:
    react: "^18.0.0 || ^19.0.0"
    react-dom: "^18.0.0 || ^19.0.0"
  dependencies:
    class-variance-authority: "^0.7.0"
    clsx: "^2.1.0"
    tailwind-merge: "^2.2.0"

# ═══════════════════════════════════════════════════════════════════════════
# TEMPLATE CONFIGURATION
# ═══════════════════════════════════════════════════════════════════════════

templates:
  dir: builtin:react/templates
  partials: builtin:react/templates/partials

# ═══════════════════════════════════════════════════════════════════════════
# PROMPT CONFIGURATION
# ═══════════════════════════════════════════════════════════════════════════

prompts:
  dir: builtin:react/prompts

# ═══════════════════════════════════════════════════════════════════════════
# GENERATION STEPS
# ═══════════════════════════════════════════════════════════════════════════

steps:
  # ─────────────────────────────────────────────────────────────────────────
  # PHASE 1: TOKENS
  # ─────────────────────────────────────────────────────────────────────────

  - name: tokens-css
    type: static
    description: Generate CSS custom properties from design tokens
    input: tokens
    template: builtin:react/templates/tokens.css.hbs
    output: src/styles/tokens.css

  - name: tokens-ts
    type: static
    description: Generate TypeScript token constants with types
    input: tokens
    template: builtin:react/templates/tokens.ts.hbs
    output: src/lib/tokens.ts

  - name: tokens-tailwind
    type: static
    description: Generate Tailwind CSS configuration
    input: tokens
    template: builtin:react/templates/tailwind.config.hbs
    output: tailwind.config.ts

  # ─────────────────────────────────────────────────────────────────────────
  # PHASE 2: UTILITIES
  # ─────────────────────────────────────────────────────────────────────────

  - name: cn-util
    type: static
    description: Generate className utility (clsx + tailwind-merge)
    template: builtin:react/templates/cn.ts.hbs
    output: src/lib/cn.ts

  - name: types-util
    type: static
    description: Generate shared type utilities
    template: builtin:react/templates/types.ts.hbs
    output: src/lib/types.ts

  # ─────────────────────────────────────────────────────────────────────────
  # PHASE 3: COMPONENTS
  # ─────────────────────────────────────────────────────────────────────────

  # 3.1 Component Types (Static - runs for each component)
  - name: component-types
    type: static
    description: Generate TypeScript props interface
    input: component
    template: builtin:react/templates/component-types.ts.hbs
    output: "src/components/{{ pascalCase component.name }}/types.ts"

  # 3.2 Component Implementation (AI-assisted)
  - name: component
    type: ai
    description: Generate React component with CVA variants
    input: component
    prompt: builtin:react/prompts/component.md
    output: "src/components/{{ pascalCase component.name }}/{{ pascalCase component.name }}.tsx"
    context:
      tokens: "{{ tokens }}"
      styling: tailwind-cva
    options:
      model: claude-sonnet-4-20250514
      temperature: 0
      maxTokens: 4000
    validate:
      - typescript
      - eslint
    retry:
      attempts: 3
      backoff: exponential
      onFailure: error
    dependsOn:
      - component-types

  # 3.3 Component Tests (AI-assisted, optional)
  - name: component-test
    type: ai
    description: Generate component unit tests
    input: component
    prompt: builtin:react/prompts/component-test.md
    output: "src/components/{{ pascalCase component.name }}/{{ pascalCase component.name }}.test.tsx"
    context:
      componentCode: "{{ steps.component.output }}"
      typesCode: "{{ steps.component-types.output }}"
    options:
      temperature: 0.1
    validate:
      - typescript
    optional: true
    dependsOn:
      - component

  # 3.4 Component Stories (AI-assisted, optional)
  - name: component-story
    type: ai
    description: Generate Storybook stories
    input: component
    prompt: builtin:react/prompts/component-story.md
    output: "src/components/{{ pascalCase component.name }}/{{ pascalCase component.name }}.stories.tsx"
    context:
      componentCode: "{{ steps.component.output }}"
    options:
      temperature: 0.1
    optional: true
    condition: "{{ config.features.stories | default: true }}"
    dependsOn:
      - component

  # 3.5 Component Barrel Export (Static)
  - name: component-index
    type: static
    description: Generate component barrel export
    input: component
    template: builtin:react/templates/component-index.ts.hbs
    output: "src/components/{{ pascalCase component.name }}/index.ts"
    dependsOn:
      - component
      - component-types

  # ─────────────────────────────────────────────────────────────────────────
  # PHASE 4: PACKAGE STRUCTURE
  # ─────────────────────────────────────────────────────────────────────────

  - name: components-index
    type: static
    description: Generate components barrel export
    input: components
    template: builtin:react/templates/components-index.ts.hbs
    output: src/components/index.ts
    dependsOn:
      - component-index  # Wait for all component indexes

  - name: main-index
    type: static
    description: Generate main package entry point
    input: components
    template: builtin:react/templates/index.ts.hbs
    output: src/index.ts
    dependsOn:
      - components-index
      - tokens-ts
      - cn-util

  - name: package-json
    type: static
    description: Generate package.json
    template: builtin:react/templates/package.json.hbs
    output: package.json
    context:
      package: "{{ pipeline.package }}"

  - name: tsconfig
    type: static
    description: Generate TypeScript configuration
    template: builtin:react/templates/tsconfig.json.hbs
    output: tsconfig.json

  - name: tsconfig-build
    type: static
    description: Generate TypeScript build configuration
    template: builtin:react/templates/tsconfig.build.json.hbs
    output: tsconfig.build.json

  - name: eslintrc
    type: static
    description: Generate ESLint configuration
    template: builtin:react/templates/eslintrc.hbs
    output: .eslintrc.cjs

  - name: prettierrc
    type: static
    description: Generate Prettier configuration
    template: builtin:react/templates/prettierrc.hbs
    output: .prettierrc

  - name: gitignore
    type: static
    description: Generate .gitignore
    template: builtin:react/templates/gitignore.hbs
    output: .gitignore

  # ─────────────────────────────────────────────────────────────────────────
  # PHASE 5: DOCUMENTATION
  # ─────────────────────────────────────────────────────────────────────────

  - name: readme
    type: ai
    description: Generate README with component documentation
    input:
      - components
      - tokens
    prompt: builtin:react/prompts/readme.md
    output: README.md
    options:
      temperature: 0.3
    optional: true

  - name: changelog
    type: static
    description: Generate CHANGELOG template
    template: builtin:react/templates/changelog.md.hbs
    output: CHANGELOG.md
    context:
      version: "{{ config.package.version }}"
      date: "{{ now 'YYYY-MM-DD' }}"

# ═══════════════════════════════════════════════════════════════════════════
# HOOKS
# ═══════════════════════════════════════════════════════════════════════════

hooks:
  afterGenerate:
    - command: npm install
      cwd: "{{ output.dir }}"
      
    - command: npm run lint:fix
      cwd: "{{ output.dir }}"
      condition: "{{ config.features.autoFix | default: true }}"
      
    - command: npm run build
      cwd: "{{ output.dir }}"

  afterValidate:
    - command: npm test
      cwd: "{{ output.dir }}"
      condition: "{{ config.features.runTests | default: true }}"
```

### Template Files for builtin:react

#### tokens.css.hbs

```handlebars
{{!-- builtin:react/templates/tokens.css.hbs --}}
/**
 * Design Tokens - CSS Custom Properties
 * 
 * Generated by dspec
 * DO NOT EDIT DIRECTLY
 */

:root {
  /* ════════════════════════════════════════════════════════════════════════
   * COLORS
   * ════════════════════════════════════════════════════════════════════════ */
{{#each tokens.colors}}
  {{#each this}}
  --color-{{@../key}}-{{@key}}: {{this.value}};
  {{/each}}
{{/each}}

  /* ════════════════════════════════════════════════════════════════════════
   * TYPOGRAPHY
   * ════════════════════════════════════════════════════════════════════════ */
{{#each tokens.typography.fontFamily}}
  --font-{{@key}}: {{this}};
{{/each}}

{{#each tokens.typography.fontSize}}
  --text-{{@key}}: {{this.size}};
  --text-{{@key}}-line-height: {{this.lineHeight}};
  {{#if this.letterSpacing}}
  --text-{{@key}}-letter-spacing: {{this.letterSpacing}};
  {{/if}}
{{/each}}

{{#each tokens.typography.fontWeight}}
  --font-weight-{{@key}}: {{this}};
{{/each}}

  /* ════════════════════════════════════════════════════════════════════════
   * SPACING
   * ════════════════════════════════════════════════════════════════════════ */
{{#each tokens.spacing}}
  --spacing-{{@key}}: {{this}}px;
{{/each}}

  /* ════════════════════════════════════════════════════════════════════════
   * RADII
   * ════════════════════════════════════════════════════════════════════════ */
{{#each tokens.radii}}
  --radius-{{@key}}: {{this}};
{{/each}}

  /* ════════════════════════════════════════════════════════════════════════
   * SHADOWS
   * ════════════════════════════════════════════════════════════════════════ */
{{#each tokens.shadows}}
  --shadow-{{@key}}: {{this}};
{{/each}}

  /* ════════════════════════════════════════════════════════════════════════
   * TRANSITIONS
   * ════════════════════════════════════════════════════════════════════════ */
{{#each tokens.transitions}}
  --transition-{{@key}}: {{this}};
{{/each}}
}

/* Dark mode overrides */
@media (prefers-color-scheme: dark) {
  :root {
{{#each tokens.colors}}
  {{#each this}}
    {{#if this.dark}}
    --color-{{@../key}}-{{@key}}: {{this.dark}};
    {{/if}}
  {{/each}}
{{/each}}
  }
}

[data-theme="dark"] {
{{#each tokens.colors}}
  {{#each this}}
    {{#if this.dark}}
  --color-{{@../key}}-{{@key}}: {{this.dark}};
    {{/if}}
  {{/each}}
{{/each}}
}
```

#### tokens.ts.hbs

```handlebars
{{!-- builtin:react/templates/tokens.ts.hbs --}}
/**
 * Design Tokens - TypeScript Constants
 * 
 * Generated by dspec
 * DO NOT EDIT DIRECTLY
 */

// ════════════════════════════════════════════════════════════════════════════
// COLORS
// ════════════════════════════════════════════════════════════════════════════

export const colors = {
{{#each tokens.colors}}
  {{@key}}: {
  {{#each this}}
    {{@key}}: 'var(--color-{{@../key}}-{{@key}})',
  {{/each}}
  },
{{/each}}
} as const;

export type ColorScale = keyof typeof colors;
export type ColorToken<S extends ColorScale = ColorScale> = keyof typeof colors[S];

// ════════════════════════════════════════════════════════════════════════════
// TYPOGRAPHY
// ════════════════════════════════════════════════════════════════════════════

export const fontFamily = {
{{#each tokens.typography.fontFamily}}
  {{@key}}: 'var(--font-{{@key}})',
{{/each}}
} as const;

export const fontSize = {
{{#each tokens.typography.fontSize}}
  {{@key}}: {
    size: 'var(--text-{{@key}})',
    lineHeight: 'var(--text-{{@key}}-line-height)',
    {{#if this.letterSpacing}}
    letterSpacing: 'var(--text-{{@key}}-letter-spacing)',
    {{/if}}
  },
{{/each}}
} as const;

export const fontWeight = {
{{#each tokens.typography.fontWeight}}
  {{@key}}: 'var(--font-weight-{{@key}})',
{{/each}}
} as const;

export type FontFamily = keyof typeof fontFamily;
export type FontSize = keyof typeof fontSize;
export type FontWeight = keyof typeof fontWeight;

// ════════════════════════════════════════════════════════════════════════════
// SPACING
// ════════════════════════════════════════════════════════════════════════════

export const spacing = {
{{#each tokens.spacing}}
  '{{@key}}': 'var(--spacing-{{@key}})',
{{/each}}
} as const;

export type Spacing = keyof typeof spacing;

// ════════════════════════════════════════════════════════════════════════════
// RADII
// ════════════════════════════════════════════════════════════════════════════

export const radii = {
{{#each tokens.radii}}
  {{@key}}: 'var(--radius-{{@key}})',
{{/each}}
} as const;

export type Radius = keyof typeof radii;

// ════════════════════════════════════════════════════════════════════════════
// SHADOWS
// ════════════════════════════════════════════════════════════════════════════

export const shadows = {
{{#each tokens.shadows}}
  {{@key}}: 'var(--shadow-{{@key}})',
{{/each}}
} as const;

export type Shadow = keyof typeof shadows;

// ════════════════════════════════════════════════════════════════════════════
// TRANSITIONS
// ════════════════════════════════════════════════════════════════════════════

export const transitions = {
{{#each tokens.transitions}}
  {{@key}}: 'var(--transition-{{@key}})',
{{/each}}
} as const;

export type Transition = keyof typeof transitions;

// ════════════════════════════════════════════════════════════════════════════
// COMBINED TOKENS OBJECT
// ════════════════════════════════════════════════════════════════════════════

export const tokens = {
  colors,
  fontFamily,
  fontSize,
  fontWeight,
  spacing,
  radii,
  shadows,
  transitions,
} as const;

export type Tokens = typeof tokens;
```

#### component-types.ts.hbs

```handlebars
{{!-- builtin:react/templates/component-types.ts.hbs --}}
/**
 * {{ pascalCase component.name }} - Type Definitions
 * 
 * Generated by dspec from {{ component._specPath }}
 * DO NOT EDIT DIRECTLY
 */

import type { VariantProps } from 'class-variance-authority';
import type { {{ camelCase component.name }}Variants } from './{{ pascalCase component.name }}';

// ════════════════════════════════════════════════════════════════════════════
// VARIANT TYPES
// ════════════════════════════════════════════════════════════════════════════

{{#each component.props}}
{{#if (eq this.type 'enum')}}
export type {{ pascalCase ../component.name }}{{ pascalCase this.name }} = {{#each this.values}}'{{this}}'{{#unless @last}} | {{/unless}}{{/each}};
{{/if}}
{{/each}}

// ════════════════════════════════════════════════════════════════════════════
// PROPS INTERFACE
// ════════════════════════════════════════════════════════════════════════════

export interface {{ pascalCase component.name }}Props
  extends React.{{#if component.element}}{{ component.element }}HTMLAttributes<HTML{{ component.element }}Element>{{else}}HTMLAttributes<HTMLElement>{{/if}},
    VariantProps<typeof {{ camelCase component.name }}Variants> {
{{#each component.props}}
{{#unless (eq this.type 'enum')}}
  /**
   * {{ this.description }}
   {{#if this.default}}
   * @default {{ json this.default }}
   {{/if}}
   */
  {{ this.name }}{{#unless this.required}}?{{/unless}}: {{ this.type }};
{{/unless}}
{{/each}}

{{#if component.slots}}
  // Slot props
{{#each component.slots}}
  /**
   * {{ this.description }}
   */
  {{ this.name }}{{#unless this.required}}?{{/unless}}: React.ReactNode;
{{/each}}
{{/if}}

  /**
   * Additional CSS classes
   */
  className?: string;

  /**
   * Component children
   */
  children{{#unless component.requiresChildren}}?{{/unless}}: React.ReactNode;
}
```

#### component-index.ts.hbs

```handlebars
{{!-- builtin:react/templates/component-index.ts.hbs --}}
/**
 * {{ pascalCase component.name }} - Barrel Export
 * 
 * Generated by dspec
 * DO NOT EDIT DIRECTLY
 */

export { {{ pascalCase component.name }} } from './{{ pascalCase component.name }}';
export { {{ camelCase component.name }}Variants } from './{{ pascalCase component.name }}';
export type { {{ pascalCase component.name }}Props } from './types';
{{#each component.props}}
{{#if (eq this.type 'enum')}}
export type { {{ pascalCase ../component.name }}{{ pascalCase this.name }} } from './types';
{{/if}}
{{/each}}
```

#### cn.ts.hbs

```handlebars
{{!-- builtin:react/templates/cn.ts.hbs --}}
/**
 * Utility for merging Tailwind CSS classes
 * 
 * Generated by dspec
 * DO NOT EDIT DIRECTLY
 */

import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

/**
 * Merges class names using clsx and tailwind-merge.
 * Handles conditional classes and resolves Tailwind conflicts.
 * 
 * @example
 * cn('px-2 py-1', 'px-4') // => 'py-1 px-4'
 * cn('text-red-500', condition && 'text-blue-500')
 */
export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs));
}
```

#### Prompt: component.md

```markdown
{{!-- builtin:react/prompts/component.md --}}
You are an expert React developer generating a component from a design specification.

## Component Specification

```yaml
{{ yaml component }}
```

## Design Tokens

```yaml
{{ yaml tokens }}
```

## Requirements

### Code Style
1. Use TypeScript with strict types
2. Use `forwardRef` for ref forwarding
3. Use `cva` from class-variance-authority for variant styling
4. Use the `cn` utility for className merging (import from '../lib/cn')
5. Use CSS custom properties for token references (e.g., `var(--color-primary-500)`)

### Variant Implementation
- Implement all variants from the spec using CVA
- Each variant should map to Tailwind classes
- Use compound variants for variant combinations
- Set appropriate default variants

### Accessibility
{{#if component.accessibility}}
- ARIA role: {{ component.accessibility.role }}
{{#each component.accessibility.attributes}}
- {{ @key }}: {{ this }}
{{/each}}
{{/if}}
- Ensure proper keyboard navigation
- Include focus-visible styles

### Component States
Implement these states:
{{#each component.states}}
- {{ this }}
{{/each}}

## Expected Output Structure

```typescript
import { forwardRef } from 'react';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '../lib/cn';
import type { {{ pascalCase component.name }}Props } from './types';

export const {{ camelCase component.name }}Variants = cva(
  // base classes
  "",
  {
    variants: {
      // variant definitions
    },
    compoundVariants: [
      // compound variant definitions
    ],
    defaultVariants: {
      // defaults
    },
  }
);

export const {{ pascalCase component.name }} = forwardRef<
  HTML{{ component.element | default: 'Div' }}Element,
  {{ pascalCase component.name }}Props
>(({ className, variant, size, ...props }, ref) => {
  return (
    <{{ component.element | default: 'div' }}
      ref={ref}
      className={cn({{ camelCase component.name }}Variants({ variant, size }), className)}
      {...props}
    />
  );
});

{{ pascalCase component.name }}.displayName = '{{ pascalCase component.name }}';
```

## Output

Generate ONLY the TypeScript/React code. No explanations or markdown.
The component must be production-ready and follow all requirements above.
```

---

## 5. Template System

### Handlebars Integration

```typescript
// src/templates/engine.ts

import Handlebars from 'handlebars';
import * as changeCase from 'change-case';
import * as yaml from 'yaml';

export class TemplateEngine {
  private handlebars: typeof Handlebars;
  private partials: Map<string, HandlebarsTemplateDelegate>;
  
  constructor() {
    this.handlebars = Handlebars.create();
    this.partials = new Map();
    this.registerBuiltinHelpers();
  }
  
  private registerBuiltinHelpers(): void {
    // ═══════════════════════════════════════════════════════════════════════
    // CASE TRANSFORMS
    // ═══════════════════════════════════════════════════════════════════════
    
    this.handlebars.registerHelper('pascalCase', (str: string) => 
      changeCase.pascalCase(str)
    );
    
    this.handlebars.registerHelper('camelCase', (str: string) => 
      changeCase.camelCase(str)
    );
    
    this.handlebars.registerHelper('kebabCase', (str: string) => 
      changeCase.kebabCase(str)
    );
    
    this.handlebars.registerHelper('snakeCase', (str: string) => 
      changeCase.snakeCase(str)
    );
    
    this.handlebars.registerHelper('constantCase', (str: string) => 
      changeCase.constantCase(str)
    );
    
    this.handlebars.registerHelper('titleCase', (str: string) => 
      changeCase.capitalCase(str)
    );
    
    // ═══════════════════════════════════════════════════════════════════════
    // SERIALIZATION
    // ═══════════════════════════════════════════════════════════════════════
    
    this.handlebars.registerHelper('json', (obj: unknown, indent?: number) => 
      JSON.stringify(obj, null, typeof indent === 'number' ? indent : 2)
    );
    
    this.handlebars.registerHelper('yaml', (obj: unknown) => 
      yaml.stringify(obj)
    );
    
    // ═══════════════════════════════════════════════════════════════════════
    // ARRAY/STRING OPERATIONS
    // ═══════════════════════════════════════════════════════════════════════
    
    this.handlebars.registerHelper('join', (arr: unknown[], sep: string) => 
      Array.isArray(arr) ? arr.join(sep) : ''
    );
    
    this.handlebars.registerHelper('split', (str: string, sep: string) => 
      typeof str === 'string' ? str.split(sep) : []
    );
    
    this.handlebars.registerHelper('first', (arr: unknown[]) => 
      Array.isArray(arr) ? arr[0] : undefined
    );
    
    this.handlebars.registerHelper('last', (arr: unknown[]) => 
      Array.isArray(arr) ? arr[arr.length - 1] : undefined
    );
    
    this.handlebars.registerHelper('length', (arr: unknown[] | string) => 
      arr?.length ?? 0
    );
    
    this.handlebars.registerHelper('slice', (arr: unknown[], start: number, end?: number) => 
      Array.isArray(arr) ? arr.slice(start, end) : []
    );
    
    // ═══════════════════════════════════════════════════════════════════════
    // COMPARISONS
    // ═══════════════════════════════════════════════════════════════════════
    
    this.handlebars.registerHelper('eq', (a: unknown, b: unknown) => a === b);
    this.handlebars.registerHelper('ne', (a: unknown, b: unknown) => a !== b);
    this.handlebars.registerHelper('lt', (a: number, b: number) => a < b);
    this.handlebars.registerHelper('lte', (a: number, b: number) => a <= b);
    this.handlebars.registerHelper('gt', (a: number, b: number) => a > b);
    this.handlebars.registerHelper('gte', (a: number, b: number) => a >= b);
    
    this.handlebars.registerHelper('and', (...args: unknown[]) => {
      // Remove Handlebars options object
      args.pop();
      return args.every(Boolean);
    });
    
    this.handlebars.registerHelper('or', (...args: unknown[]) => {
      args.pop();
      return args.some(Boolean);
    });
    
    this.handlebars.registerHelper('not', (value: unknown) => !value);
    
    this.handlebars.registerHelper('includes', (arr: unknown[], value: unknown) => 
      Array.isArray(arr) && arr.includes(value)
    );
    
    this.handlebars.registerHelper('hasKey', (obj: object, key: string) => 
      obj && typeof obj === 'object' && key in obj
    );
    
    // ═══════════════════════════════════════════════════════════════════════
    // TOKEN HELPERS
    // ═══════════════════════════════════════════════════════════════════════
    
    this.handlebars.registerHelper('tokenRef', (path: string) => 
      `var(--${path.replace(/\./g, '-')})`
    );
    
    this.handlebars.registerHelper('tokenPath', (path: string) => 
      path.replace(/\./g, '-')
    );
    
    // ═══════════════════════════════════════════════════════════════════════
    // DATE/TIME
    // ═══════════════════════════════════════════════════════════════════════
    
    this.handlebars.registerHelper('now', (format?: string) => {
      const now = new Date();
      if (!format || typeof format !== 'string') {
        return now.toISOString();
      }
      // Simple format support
      return format
        .replace('YYYY', now.getFullYear().toString())
        .replace('MM', (now.getMonth() + 1).toString().padStart(2, '0'))
        .replace('DD', now.getDate().toString().padStart(2, '0'))
        .replace('HH', now.getHours().toString().padStart(2, '0'))
        .replace('mm', now.getMinutes().toString().padStart(2, '0'))
        .replace('ss', now.getSeconds().toString().padStart(2, '0'));
    });
    
    // ═══════════════════════════════════════════════════════════════════════
    // CODE GENERATION
    // ═══════════════════════════════════════════════════════════════════════
    
    this.handlebars.registerHelper('indent', (str: string, spaces: number) => {
      const indent = ' '.repeat(spaces);
      return str.split('\n').map(line => indent + line).join('\n');
    });
    
    this.handlebars.registerHelper('comment', (str: string, style: string) => {
      switch (style) {
        case 'line':
          return `// ${str}`;
        case 'block':
          return `/* ${str} */`;
        case 'jsdoc':
          return `/** ${str} */`;
        default:
          return `// ${str}`;
      }
    });
    
    // ═══════════════════════════════════════════════════════════════════════
    // DEFAULT/COALESCE
    // ═══════════════════════════════════════════════════════════════════════
    
    this.handlebars.registerHelper('default', (value: unknown, defaultValue: unknown) => 
      value ?? defaultValue
    );
    
    this.handlebars.registerHelper('coalesce', (...args: unknown[]) => {
      args.pop(); // Remove options
      return args.find(arg => arg != null);
    });
  }
  
  /**
   * Register custom helpers from a module
   */
  async registerCustomHelpers(helpersPath: string): Promise<void> {
    const module = await import(helpersPath);
    const helpers = module.default || module;
    
    for (const [name, fn] of Object.entries(helpers)) {
      if (typeof fn === 'function') {
        this.handlebars.registerHelper(name, fn);
      }
    }
  }
  
  /**
   * Register partials from a directory
   */
  async registerPartials(partialsDir: string): Promise<void> {
    const files = await fs.readdir(partialsDir);
    
    for (const file of files) {
      if (file.endsWith('.hbs')) {
        const name = file.replace('.hbs', '');
        const content = await fs.readFile(path.join(partialsDir, file), 'utf-8');
        this.handlebars.registerPartial(name, content);
      }
    }
  }
  
  /**
   * Compile and render a template
   */
  render(template: string, context: object): string {
    const compiled = this.handlebars.compile(template);
    return compiled(context);
  }
  
  /**
   * Render a template file
   */
  async renderFile(templatePath: string, context: object): Promise<string> {
    const template = await fs.readFile(templatePath, 'utf-8');
    return this.render(template, context);
  }
}
```

### Custom Helpers Example

```typescript
// templates/helpers.ts

/**
 * Custom Handlebars helpers for dspec
 */

export default {
  /**
   * Generate CVA variant object
   */
  cvaVariants(variants: Record<string, Record<string, string>>): string {
    const lines: string[] = [];
    
    for (const [variantName, options] of Object.entries(variants)) {
      lines.push(`      ${variantName}: {`);
      for (const [optionName, classes] of Object.entries(options)) {
        lines.push(`        ${optionName}: '${classes}',`);
      }
      lines.push(`      },`);
    }
    
    return lines.join('\n');
  },
  
  /**
   * Generate TypeScript union type from array
   */
  unionType(values: string[]): string {
    return values.map(v => `'${v}'`).join(' | ');
  },
  
  /**
   * Check if component has a specific state
   */
  hasState(component: { states?: string[] }, state: string): boolean {
    return component.states?.includes(state) ?? false;
  },
  
  /**
   * Generate import statement
   */
  importStatement(
    items: string | string[], 
    from: string, 
    options?: { type?: boolean }
  ): string {
    const itemList = Array.isArray(items) ? items.join(', ') : items;
    const typeKeyword = options?.type ? 'type ' : '';
    return `import ${typeKeyword}{ ${itemList} } from '${from}';`;
  },
};
```

---

## 6. Dependency Graph

### Dependency Resolution

```typescript
// src/engine/dependency-graph.ts

import { Step, Pipeline } from '../types/pipeline';

interface DependencyNode {
  step: Step;
  dependencies: Set<string>;
  dependents: Set<string>;
  state: 'pending' | 'ready' | 'running' | 'complete' | 'failed';
}

export class DependencyGraph {
  private nodes: Map<string, DependencyNode>;
  private inputDependencies: Map<string, Set<string>>;
  
  constructor(pipeline: Pipeline) {
    this.nodes = new Map();
    this.inputDependencies = new Map();
    this.build(pipeline);
  }
  
  private build(pipeline: Pipeline): void {
    // First pass: create nodes
    for (const step of pipeline.steps) {
      this.nodes.set(step.name, {
        step,
        dependencies: new Set(),
        dependents: new Set(),
        state: 'pending',
      });
    }
    
    // Second pass: resolve dependencies
    for (const step of pipeline.steps) {
      const node = this.nodes.get(step.name)!;
      
      // Explicit dependencies
      if (step.dependsOn) {
        for (const dep of step.dependsOn) {
          node.dependencies.add(dep);
          this.nodes.get(dep)?.dependents.add(step.name);
        }
      }
      
      // "after" dependencies
      if (step.after) {
        const afters = Array.isArray(step.after) ? step.after : [step.after];
        for (const after of afters) {
          node.dependencies.add(after);
          this.nodes.get(after)?.dependents.add(step.name);
        }
      }
      
      // Implicit dependencies from input
      if (step.input) {
        const inputs = Array.isArray(step.input) ? step.input : [step.input];
        for (const input of inputs) {
          if (!this.inputDependencies.has(input)) {
            this.inputDependencies.set(input, new Set());
          }
          this.inputDependencies.get(input)!.add(step.name);
        }
      }
    }
    
    // Handle "before" declarations (reverse dependencies)
    for (const step of pipeline.steps) {
      if (step.before) {
        const befores = Array.isArray(step.before) ? step.before : [step.before];
        for (const before of befores) {
          this.nodes.get(before)?.dependencies.add(step.name);
          this.nodes.get(step.name)?.dependents.add(before);
        }
      }
    }
    
    // Detect cycles
    this.detectCycles();
  }
  
  private detectCycles(): void {
    const visited = new Set<string>();
    const recursionStack = new Set<string>();
    
    const dfs = (name: string, path: string[]): void => {
      visited.add(name);
      recursionStack.add(name);
      
      const node = this.nodes.get(name);
      if (!node) return;
      
      for (const dep of node.dependencies) {
        if (!visited.has(dep)) {
          dfs(dep, [...path, name]);
        } else if (recursionStack.has(dep)) {
          throw new Error(
            `Circular dependency detected: ${[...path, name, dep].join(' -> ')}`
          );
        }
      }
      
      recursionStack.delete(name);
    };
    
    for (const name of this.nodes.keys()) {
      if (!visited.has(name)) {
        dfs(name, []);
      }
    }
  }
  
  /**
   * Get steps ready to execute (all dependencies satisfied)
   */
  getReadySteps(): Step[] {
    const ready: Step[] = [];
    
    for (const [name, node] of this.nodes) {
      if (node.state !== 'pending') continue;
      
      const allDepsComplete = [...node.dependencies].every(
        dep => this.nodes.get(dep)?.state === 'complete'
      );
      
      if (allDepsComplete) {
        ready.push(node.step);
      }
    }
    
    return ready;
  }
  
  /**
   * Mark a step as complete
   */
  markComplete(stepName: string): void {
    const node = this.nodes.get(stepName);
    if (node) {
      node.state = 'complete';
    }
  }
  
  /**
   * Mark a step as failed
   */
  markFailed(stepName: string): void {
    const node = this.nodes.get(stepName);
    if (node) {
      node.state = 'failed';
    }
  }
  
  /**
   * Get all steps that depend on a given step
   */
  getDependents(stepName: string): string[] {
    return [...(this.nodes.get(stepName)?.dependents ?? [])];
  }
  
  /**
   * Get cascade path: all steps affected by regenerating a step
   */
  getCascade(stepName: string): string[] {
    const cascade: string[] = [];
    const visited = new Set<string>();
    
    const collect = (name: string): void => {
      if (visited.has(name)) return;
      visited.add(name);
      cascade.push(name);
      
      for (const dependent of this.getDependents(name)) {
        collect(dependent);
      }
    };
    
    collect(stepName);
    return cascade;
  }
  
  /**
   * Get optimal parallel execution groups
   */
  getExecutionPlan(): Step[][] {
    const plan: Step[][] = [];
    const completed = new Set<string>();
    
    while (completed.size < this.nodes.size) {
      const group: Step[] = [];
      
      for (const [name, node] of this.nodes) {
        if (completed.has(name)) continue;
        
        const allDepsComplete = [...node.dependencies].every(
          dep => completed.has(dep)
        );
        
        if (allDepsComplete && node.step.parallel !== false) {
          group.push(node.step);
        }
      }
      
      if (group.length === 0) {
        // No parallel steps available, take first sequential
        for (const [name, node] of this.nodes) {
          if (!completed.has(name)) {
            const allDepsComplete = [...node.dependencies].every(
              dep => completed.has(dep)
            );
            if (allDepsComplete) {
              group.push(node.step);
              break;
            }
          }
        }
      }
      
      if (group.length === 0) {
        throw new Error('Deadlock detected in execution plan');
      }
      
      plan.push(group);
      for (const step of group) {
        completed.add(step.name);
      }
    }
    
    return plan;
  }
  
  /**
   * Visualize the dependency graph (for debugging)
   */
  toMermaid(): string {
    const lines: string[] = ['graph TD'];
    
    for (const [name, node] of this.nodes) {
      for (const dep of node.dependencies) {
        lines.push(`  ${dep} --> ${name}`);
      }
    }
    
    return lines.join('\n');
  }
}
```

### Cascade Regeneration

```typescript
// src/engine/cascade.ts

import { DependencyGraph } from './dependency-graph';
import { StepExecutor } from './executors/base';

export interface CascadeOptions {
  /** Only regenerate if hash changed */
  smart?: boolean;
  /** Dry run - show what would regenerate */
  dryRun?: boolean;
  /** Force regenerate even if unchanged */
  force?: boolean;
}

export class CascadeRegenerator {
  private graph: DependencyGraph;
  private executors: Map<string, StepExecutor>;
  private hashStore: HashStore;
  
  constructor(
    graph: DependencyGraph,
    executors: Map<string, StepExecutor>,
    hashStore: HashStore
  ) {
    this.graph = graph;
    this.executors = executors;
    this.hashStore = hashStore;
  }
  
  /**
   * Regenerate a step and all its dependents
   */
  async regenerate(
    stepName: string,
    ctx: StepContext,
    options: CascadeOptions = {}
  ): Promise<CascadeResult> {
    const cascade = this.graph.getCascade(stepName);
    const results: StepResult[] = [];
    const skipped: string[] = [];
    
    console.log(`Cascade regeneration: ${cascade.join(' -> ')}`);
    
    if (options.dryRun) {
      return {
        cascade,
        results: [],
        skipped: [],
        dryRun: true,
      };
    }
    
    for (const name of cascade) {
      const node = this.graph.getNode(name);
      if (!node) continue;
      
      // Check if regeneration needed (smart mode)
      if (options.smart && !options.force) {
        const inputHash = this.computeInputHash(node.step, ctx);
        const storedHash = await this.hashStore.get(name);
        
        if (inputHash === storedHash) {
          skipped.push(name);
          continue;
        }
      }
      
      // Execute step
      const executor = this.executors.get(node.step.type);
      if (!executor) {
        throw new Error(`No executor for step type: ${node.step.type}`);
      }
      
      const result = await executor.execute({
        ...ctx,
        step: node.step,
      });
      
      results.push({ stepName: name, ...result });
      
      // Update hash
      if (result.success) {
        const outputHash = this.computeOutputHash(result.files);
        await this.hashStore.set(name, outputHash);
      }
    }
    
    return { cascade, results, skipped, dryRun: false };
  }
  
  /**
   * Determine what would regenerate if a spec changes
   */
  analyzeImpact(specType: string): ImpactAnalysis {
    const affectedSteps = this.graph.getStepsWithInput(specType);
    const allAffected = new Set<string>();
    
    for (const step of affectedSteps) {
      for (const name of this.graph.getCascade(step)) {
        allAffected.add(name);
      }
    }
    
    return {
      directlyAffected: affectedSteps,
      totalAffected: [...allAffected],
      cascadeDepth: this.computeCascadeDepth(affectedSteps),
    };
  }
}

export interface CascadeResult {
  cascade: string[];
  results: StepResult[];
  skipped: string[];
  dryRun: boolean;
}

export interface ImpactAnalysis {
  directlyAffected: string[];
  totalAffected: string[];
  cascadeDepth: number;
}
```

---

## 7. Hooks System

### Hook Execution Engine

```typescript
// src/engine/hooks.ts

import { spawn } from 'child_process';
import { Hooks, HookAction, HookActionConfig } from '../types/pipeline';

export type HookType = keyof Hooks;

export interface HookContext {
  pipeline: Pipeline;
  step?: Step;
  result?: StepResult;
  error?: Error;
  outputs: Record<string, StepOutput>;
  config: DspecConfig;
}

export class HookExecutor {
  private templateEngine: TemplateEngine;
  
  constructor(templateEngine: TemplateEngine) {
    this.templateEngine = templateEngine;
  }
  
  /**
   * Execute hooks of a specific type
   */
  async execute(
    hooks: Hooks | undefined,
    hookType: HookType,
    ctx: HookContext
  ): Promise<HookResult[]> {
    const actions = hooks?.[hookType];
    if (!actions || actions.length === 0) {
      return [];
    }
    
    const results: HookResult[] = [];
    
    for (const action of actions) {
      const result = await this.executeAction(action, ctx);
      results.push(result);
      
      // Stop on failure unless it's an error hook
      if (!result.success && hookType !== 'onError') {
        break;
      }
    }
    
    return results;
  }
  
  private async executeAction(
    action: HookAction,
    ctx: HookContext
  ): Promise<HookResult> {
    const config = this.normalizeAction(action);
    
    // Evaluate condition
    if (config.condition) {
      const shouldRun = this.evaluateCondition(config.condition, ctx);
      if (!shouldRun) {
        return {
          success: true,
          skipped: true,
          command: config.command,
        };
      }
    }
    
    // Resolve template variables in command
    const command = this.templateEngine.render(config.command, ctx);
    const cwd = config.cwd 
      ? this.templateEngine.render(config.cwd, ctx)
      : process.cwd();
    
    // Execute command
    return new Promise((resolve) => {
      const startTime = Date.now();
      
      const child = spawn(command, {
        shell: true,
        cwd,
        env: { ...process.env, ...config.env },
        stdio: ['inherit', 'pipe', 'pipe'],
      });
      
      let stdout = '';
      let stderr = '';
      
      child.stdout?.on('data', (data) => {
        stdout += data.toString();
        process.stdout.write(data);
      });
      
      child.stderr?.on('data', (data) => {
        stderr += data.toString();
        process.stderr.write(data);
      });
      
      const timeout = config.timeout 
        ? setTimeout(() => {
            child.kill('SIGTERM');
          }, config.timeout)
        : null;
      
      child.on('close', (code) => {
        if (timeout) clearTimeout(timeout);
        
        resolve({
          success: code === 0,
          skipped: false,
          command,
          cwd,
          exitCode: code ?? -1,
          stdout,
          stderr,
          duration: Date.now() - startTime,
        });
      });
      
      child.on('error', (error) => {
        if (timeout) clearTimeout(timeout);
        
        resolve({
          success: false,
          skipped: false,
          command,
          cwd,
          error: error.message,
          duration: Date.now() - startTime,
        });
      });
    });
  }
  
  private normalizeAction(action: HookAction): HookActionConfig {
    if (typeof action === 'string') {
      return { command: action };
    }
    return action;
  }
  
  private evaluateCondition(condition: string, ctx: HookContext): boolean {
    // Simple template-based condition evaluation
    const result = this.templateEngine.render(`{{#if ${condition}}}true{{/if}}`, ctx);
    return result === 'true';
  }
}

export interface HookResult {
  success: boolean;
  skipped: boolean;
  command: string;
  cwd?: string;
  exitCode?: number;
  stdout?: string;
  stderr?: string;
  error?: string;
  duration?: number;
}
```

### Hook Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              HOOK LIFECYCLE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────┐                                                        │
│   │ beforeGenerate  │  ← Runs once before any generation                    │
│   └────────┬────────┘                                                        │
│            │                                                                 │
│            ▼                                                                 │
│   ┌─────────────────────────────────────────────────────────┐               │
│   │                    FOR EACH STEP                         │               │
│   │                                                          │               │
│   │   ┌─────────────┐                                        │               │
│   │   │ beforeStep  │  ← Runs before each step execution    │               │
│   │   └──────┬──────┘                                        │               │
│   │          │                                               │               │
│   │          ▼                                               │               │
│   │   ┌─────────────┐                                        │               │
│   │   │   Execute   │                                        │               │
│   │   │    Step     │                                        │               │
│   │   └──────┬──────┘                                        │               │
│   │          │                                               │               │
│   │          ├──────────────┐                                │               │
│   │          │              │                                │               │
│   │          ▼              ▼                                │               │
│   │   ┌───────────┐  ┌───────────┐                          │               │
│   │   │ afterStep │  │  onError  │ ← Only on failure        │               │
│   │   │ (success) │  │           │                          │               │
│   │   └───────────┘  └───────────┘                          │               │
│   │                                                          │               │
│   └─────────────────────────────────────────────────────────┘               │
│            │                                                                 │
│            ▼                                                                 │
│   ┌─────────────────┐                                                        │
│   │ afterGenerate   │  ← Runs after all steps complete                      │
│   └────────┬────────┘                                                        │
│            │                                                                 │
│            ▼                                                                 │
│   ┌─────────────────┐                                                        │
│   │ afterValidate   │  ← Runs after validation (tests, etc.)                │
│   └────────┬────────┘                                                        │
│            │                                                                 │
│            ▼                                                                 │
│   ┌─────────────────┐                                                        │
│   │ beforePublish   │  ← Runs before npm publish                            │
│   └────────┬────────┘                                                        │
│            │                                                                 │
│            ▼                                                                 │
│   ┌─────────────────┐                                                        │
│   │ afterPublish    │  ← Runs after successful publish                      │
│   └─────────────────┘                                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Hook Examples

```yaml
hooks:
  # ═══════════════════════════════════════════════════════════════════════════
  # BEFORE GENERATION
  # ═══════════════════════════════════════════════════════════════════════════
  
  beforeGenerate:
    # Clean previous output
    - command: rm -rf {{ output.dir }}/src
      condition: output.clean
    
    # Validate environment
    - command: node -v && npm -v
    
  # ═══════════════════════════════════════════════════════════════════════════
  # AFTER GENERATION
  # ═══════════════════════════════════════════════════════════════════════════
  
  afterGenerate:
    # Install dependencies
    - command: npm install
      cwd: "{{ output.dir }}"
    
    # Format code
    - command: npm run format
      cwd: "{{ output.dir }}"
      condition: config.features.autoFormat
    
    # Build package
    - command: npm run build
      cwd: "{{ output.dir }}"
    
  # ═══════════════════════════════════════════════════════════════════════════
  # AFTER VALIDATION
  # ═══════════════════════════════════════════════════════════════════════════
  
  afterValidate:
    # Run tests
    - command: npm test
      cwd: "{{ output.dir }}"
      condition: config.features.runTests
      timeout: 300000  # 5 minutes
    
    # Run visual regression
    - command: npm run test:visual
      cwd: "{{ output.dir }}"
      condition: config.features.visualRegression
      
  # ═══════════════════════════════════════════════════════════════════════════
  # STEP-LEVEL HOOKS
  # ═══════════════════════════════════════════════════════════════════════════
  
  beforeStep:
    - command: echo "Starting step: {{ step.name }}"
    
  afterStep:
    - command: echo "Completed: {{ step.name }} ({{ result.metrics.duration }}ms)"
    
  # ═══════════════════════════════════════════════════════════════════════════
  # ERROR HANDLING
  # ═══════════════════════════════════════════════════════════════════════════
  
  onError:
    # Log error details
    - command: |
        echo "Error in step: {{ step.name }}"
        echo "Error: {{ error.message }}"
    
    # Send notification (optional)
    - command: |
        curl -X POST $SLACK_WEBHOOK \
          -H 'Content-Type: application/json' \
          -d '{"text": "dspec generation failed: {{ error.message }}"}'
      condition: env.SLACK_WEBHOOK
      
  # ═══════════════════════════════════════════════════════════════════════════
  # PUBLISHING
  # ═══════════════════════════════════════════════════════════════════════════
  
  beforePublish:
    # Verify build
    - command: npm run verify
      cwd: "{{ output.dir }}"
    
    # Check version
    - command: npm version {{ config.package.version }} --no-git-tag-version
      cwd: "{{ output.dir }}"
      
  afterPublish:
    # Create git tag
    - command: git tag v{{ config.package.version }}
      condition: config.features.gitTag
    
    # Push tag
    - command: git push --tags
      condition: config.features.gitTag
```

---

## 8. Error Handling

### Error Types

```typescript
// src/errors.ts

export class DspecError extends Error {
  code: string;
  
  constructor(code: string, message: string) {
    super(message);
    this.code = code;
    this.name = 'DspecError';
  }
}

export class PipelineError extends DspecError {
  pipeline: string;
  
  constructor(pipeline: string, message: string) {
    super('PIPELINE_ERROR', message);
    this.pipeline = pipeline;
  }
}

export class StepError extends DspecError {
  step: string;
  attempt: number;
  
  constructor(step: string, attempt: number, message: string, cause?: Error) {
    super('STEP_ERROR', message);
    this.step = step;
    this.attempt = attempt;
    this.cause = cause;
  }
}

export class ValidationError extends DspecError {
  step: string;
  validator: string;
  details: ValidationDetail[];
  
  constructor(step: string, validator: string, details: ValidationDetail[]) {
    super('VALIDATION_ERROR', `Validation failed in ${step}: ${validator}`);
    this.step = step;
    this.validator = validator;
    this.details = details;
  }
}

export class TemplateError extends DspecError {
  template: string;
  line?: number;
  
  constructor(template: string, message: string, line?: number) {
    super('TEMPLATE_ERROR', message);
    this.template = template;
    this.line = line;
  }
}

export class AIError extends DspecError {
  step: string;
  provider: string;
  
  constructor(step: string, provider: string, message: string) {
    super('AI_ERROR', message);
    this.step = step;
    this.provider = provider;
  }
}

export interface ValidationDetail {
  path: string;
  message: string;
  severity: 'error' | 'warning';
}
```

### Error Recovery

```typescript
// src/engine/recovery.ts

export interface RecoveryStrategy {
  /** Attempt to recover from error */
  recover(error: DspecError, ctx: StepContext): Promise<RecoveryResult>;
  
  /** Check if this strategy can handle the error */
  canHandle(error: DspecError): boolean;
}

export interface RecoveryResult {
  recovered: boolean;
  result?: StepResult;
  action?: 'retry' | 'skip' | 'abort' | 'manual';
}

export class RetryRecovery implements RecoveryStrategy {
  canHandle(error: DspecError): boolean {
    return error.code === 'AI_ERROR' || error.code === 'STEP_ERROR';
  }
  
  async recover(error: DspecError, ctx: StepContext): Promise<RecoveryResult> {
    const { step } = ctx;
    const retryConfig = step.retry || { attempts: 3 };
    
    if (ctx.state.attempt >= retryConfig.attempts) {
      return {
        recovered: false,
        action: retryConfig.onFailure as any || 'error',
      };
    }
    
    // Calculate delay
    const delay = this.calculateDelay(
      ctx.state.attempt,
      retryConfig.delayMs || 1000,
      retryConfig.backoff || 'exponential'
    );
    
    await this.sleep(delay);
    
    return {
      recovered: true,
      action: 'retry',
    };
  }
  
  private calculateDelay(attempt: number, baseDelay: number, backoff: string): number {
    if (backoff === 'exponential') {
      return baseDelay * Math.pow(2, attempt - 1);
    }
    return baseDelay * attempt;
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

export class ManualRecovery implements RecoveryStrategy {
  canHandle(error: DspecError): boolean {
    return true; // Last resort
  }
  
  async recover(error: DspecError, ctx: StepContext): Promise<RecoveryResult> {
    // Log error for manual intervention
    console.error(`
╔══════════════════════════════════════════════════════════════════════╗
║                       MANUAL INTERVENTION REQUIRED                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ Step: ${ctx.step.name.padEnd(58)}║
║ Error: ${error.message.slice(0, 56).padEnd(56)}║
╠══════════════════════════════════════════════════════════════════════╣
║ Options:                                                             ║
║   1. Fix the issue and run: dspec generate --resume                  ║
║   2. Skip this step: dspec generate --skip ${ctx.step.name.padEnd(25)}║
║   3. Abort generation                                                ║
╚══════════════════════════════════════════════════════════════════════╝
    `);
    
    return {
      recovered: false,
      action: 'manual',
    };
  }
}
```

---

## 9. CLI Integration

### CLI Commands

```typescript
// src/cli/commands/generate.ts

import { Command } from 'commander';
import { PipelineOrchestrator } from '../../engine/orchestrator';

export function createGenerateCommand(): Command {
  return new Command('generate')
    .description('Generate output from design specs')
    .option('-t, --target <target>', 'Target pipeline (e.g., react, ios)')
    .option('-s, --step <step>', 'Run only specific step')
    .option('--skip <steps...>', 'Skip specific steps')
    .option('--no-cache', 'Disable AI response caching')
    .option('--no-ai', 'Run static steps only (skip AI steps)')
    .option('--cached', 'Use cached AI responses only')
    .option('--dry-run', 'Show what would be generated without writing')
    .option('--verbose', 'Show detailed output')
    .option('--resume', 'Resume from last failed step')
    .option('-w, --watch', 'Watch for changes and regenerate')
    .option('--parallel', 'Run steps in parallel where possible')
    .action(async (options) => {
      const orchestrator = new PipelineOrchestrator();
      
      await orchestrator.run({
        target: options.target,
        step: options.step,
        skip: options.skip,
        cache: options.cache,
        ai: options.ai,
        cachedOnly: options.cached,
        dryRun: options.dryRun,
        verbose: options.verbose,
        resume: options.resume,
        watch: options.watch,
        parallel: options.parallel,
      });
    });
}
```

### Generation Output

```
$ dspec generate --target react --verbose

╔══════════════════════════════════════════════════════════════════════╗
║                    dspec v1.0.0 — Design to Code                     ║
╚══════════════════════════════════════════════════════════════════════╝

Pipeline: react
Output: ./dist/react

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Phase 1: Tokens
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ tokens-css            src/styles/tokens.css           12ms
✓ tokens-ts             src/lib/tokens.ts               8ms
✓ tokens-tailwind       tailwind.config.ts              5ms

Phase 2: Utilities
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ cn-util               src/lib/cn.ts                   3ms
✓ types-util            src/lib/types.ts                4ms

Phase 3: Components (5)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Button
  ✓ component-types     src/components/Button/types.ts       6ms
  ✓ component           src/components/Button/Button.tsx     1.2s (AI)
  ✓ component-test      src/components/Button/Button.test.tsx 0.8s (AI, cached)
  ✓ component-story     src/components/Button/Button.stories.tsx 0.6s (AI)
  ✓ component-index     src/components/Button/index.ts       3ms

Input
  ✓ component-types     src/components/Input/types.ts        5ms
  ✓ component           src/components/Input/Input.tsx       1.4s (AI)
  ...

Phase 4: Package Structure
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ components-index      src/components/index.ts          7ms
✓ main-index            src/index.ts                     4ms
✓ package-json          package.json                     3ms
✓ tsconfig              tsconfig.json                    2ms

Phase 5: Documentation
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ readme                README.md                        0.9s (AI)
✓ changelog             CHANGELOG.md                     3ms

Hooks: afterGenerate
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ npm install                                            12.4s
✓ npm run lint:fix                                       3.2s
✓ npm run build                                          8.7s

Hooks: afterValidate
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ npm test                                               6.3s

╔══════════════════════════════════════════════════════════════════════╗
║ ✓ Generation complete                                                ║
╠══════════════════════════════════════════════════════════════════════╣
║ Files: 47 generated                                                  ║
║ Time: 42.3s (AI: 5.8s, Static: 0.1s, Hooks: 30.6s)                  ║
║ Cache: 3 AI responses cached, 1 reused                               ║
║                                                                      ║
║ Output: ./dist/react                                                 ║
║ Package: @org/design-system@1.0.0                                    ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Summary

This architecture provides:

| Feature | Implementation |
|---------|---------------|
| **Pipeline Definition** | YAML schema with full type definitions |
| **Step Execution** | Lifecycle state machine with 10 states |
| **Static Generation** | Handlebars templates with custom helpers |
| **AI Generation** | Prompt templates with response caching |
| **Dependency Graph** | DAG with cycle detection and cascade analysis |
| **Hooks System** | 8 hook types with condition support |
| **Error Handling** | Typed errors with recovery strategies |
| **CLI Integration** | Full command suite with progress output |

The `builtin:react` pipeline is fully specified and ready for implementation.
