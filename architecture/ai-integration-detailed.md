# AI Integration Architecture for dspec

**Version:** 1.0.0  
**Date:** 2026-01-28  
**Status:** Draft  
**Authors:** Architecture Team

---

## Executive Summary

This document details the AI integration architecture for dspec's component generation system. Following the "static-first, AI-optional" philosophy from the plan, this architecture ensures:

- **Swappable providers** — Not locked to any single AI vendor
- **Deterministic generation** — Same input always produces same output via caching
- **Robust validation** — Multiple layers ensure generated code meets quality standards
- **Cost visibility** — Every generation's cost is tracked and projected

**Primary Stack:**
| Component | Choice | Rationale |
|-----------|--------|-----------|
| SDK | Vercel AI SDK | TypeScript-native, Zod integration, provider-agnostic |
| Validation | Zod schemas | Type-safe, excellent DX, built-in retry support |
| Caching | Content-addressable (SHA-256) | Deterministic, auditable, git-friendly |
| Secondary Provider | Anthropic SDK direct | For Anthropic-specific features when needed |

---

## Table of Contents

1. [Provider Abstraction](#1-provider-abstraction)
2. [Prompt System](#2-prompt-system)
3. [Caching Strategy](#3-caching-strategy)
4. [Validation Pipeline](#4-validation-pipeline)
5. [Error Handling and Retry Logic](#5-error-handling-and-retry-logic)
6. [Cost Management](#6-cost-management)
7. [Implementation Checklist](#7-implementation-checklist)

---

## 1. Provider Abstraction

### 1.1 Design Goals

- **Vendor Independence:** Switch between Claude, GPT, or future models without code changes
- **Consistent Interface:** Single API regardless of underlying provider
- **Feature Parity:** Normalize capabilities across providers
- **Graceful Degradation:** Fall back if a provider is unavailable

### 1.2 Core Interfaces

```typescript
// src/ai/types.ts

import { z } from 'zod';

/**
 * Supported AI providers
 */
export type AIProvider = 'anthropic' | 'openai' | 'google' | 'local';

/**
 * Model configuration
 */
export interface ModelConfig {
  provider: AIProvider;
  model: string;           // e.g., "claude-sonnet-4-5-20250929"
  version: string;         // Pinned version for reproducibility
  temperature: number;     // Always 0 for code generation
  maxTokens: number;
  timeout: number;         // Request timeout in ms
}

/**
 * Generation request
 */
export interface GenerationRequest<T extends z.ZodType> {
  prompt: ComposedPrompt;
  schema: T;
  config?: Partial<ModelConfig>;
  cacheStrategy?: 'always' | 'if-available' | 'never';
  metadata?: Record<string, string>;
}

/**
 * Generation result with full provenance
 */
export interface GenerationResult<T> {
  output: T;
  provenance: GenerationProvenance;
  usage: TokenUsage;
  cached: boolean;
}

/**
 * Full provenance for every generation
 */
export interface GenerationProvenance {
  specHash: string;        // Hash of input spec
  promptVersion: string;   // e.g., "react/cva-variants@2.1.0"
  promptHash: string;      // SHA-256 of rendered prompt
  model: string;           // Full model identifier
  generationHash: string;  // SHA-256 of output
  timestamp: string;       // ISO 8601
  cacheKey: string;        // Full cache key used
  dspecVersion: string;    // dspec CLI version
}

/**
 * Token usage for cost tracking
 */
export interface TokenUsage {
  inputTokens: number;
  outputTokens: number;
  cachedTokens?: number;   // Provider-level cache hits
  estimatedCost: number;   // USD
}
```

### 1.3 Provider Interface

```typescript
// src/ai/providers/base.ts

import { z } from 'zod';

/**
 * Abstract provider interface - all providers implement this
 */
export interface AIProviderClient {
  readonly name: AIProvider;
  readonly supportedModels: string[];
  
  /**
   * Generate structured output
   */
  generate<T extends z.ZodType>(
    request: ProviderRequest<T>
  ): Promise<ProviderResponse<z.infer<T>>>;
  
  /**
   * Count tokens before making request (for cost estimation)
   */
  countTokens(prompt: string): Promise<number>;
  
  /**
   * Check provider health/availability
   */
  healthCheck(): Promise<HealthStatus>;
}

interface ProviderRequest<T extends z.ZodType> {
  messages: Message[];
  schema: T;
  config: ModelConfig;
}

interface ProviderResponse<T> {
  output: T;
  usage: TokenUsage;
  raw?: unknown;  // Original provider response for debugging
}

interface HealthStatus {
  available: boolean;
  latencyMs?: number;
  error?: string;
}
```

### 1.4 Anthropic Provider Implementation

```typescript
// src/ai/providers/anthropic.ts

import Anthropic from '@anthropic-ai/sdk';
import { generateText, Output } from 'ai';
import { createAnthropic } from '@ai-sdk/anthropic';
import { z } from 'zod';

export class AnthropicProvider implements AIProviderClient {
  readonly name = 'anthropic' as const;
  readonly supportedModels = [
    'claude-sonnet-4-5-20250929',
    'claude-sonnet-4-20250514',
    'claude-3-5-haiku-20241022',
  ];
  
  private client: Anthropic;
  private aiSdkProvider: ReturnType<typeof createAnthropic>;
  
  constructor(apiKey: string) {
    this.client = new Anthropic({ apiKey });
    this.aiSdkProvider = createAnthropic({ apiKey });
  }
  
  async generate<T extends z.ZodType>(
    request: ProviderRequest<T>
  ): Promise<ProviderResponse<z.infer<T>>> {
    const { messages, schema, config } = request;
    
    // Use Vercel AI SDK for structured output
    const { output, usage } = await generateText({
      model: this.aiSdkProvider(config.model),
      messages: this.formatMessages(messages),
      output: Output.object({
        schema,
      }),
      temperature: config.temperature,
      maxTokens: config.maxTokens,
    });
    
    return {
      output,
      usage: {
        inputTokens: usage.promptTokens,
        outputTokens: usage.completionTokens,
        estimatedCost: this.calculateCost(usage, config.model),
      },
    };
  }
  
  async countTokens(prompt: string): Promise<number> {
    const result = await this.client.messages.count_tokens({
      model: 'claude-sonnet-4-5-20250929',
      messages: [{ role: 'user', content: prompt }],
    });
    return result.input_tokens;
  }
  
  async healthCheck(): Promise<HealthStatus> {
    const start = Date.now();
    try {
      await this.client.messages.create({
        model: 'claude-3-5-haiku-20241022',
        max_tokens: 1,
        messages: [{ role: 'user', content: 'ping' }],
      });
      return { available: true, latencyMs: Date.now() - start };
    } catch (error) {
      return { available: false, error: String(error) };
    }
  }
  
  private calculateCost(
    usage: { promptTokens: number; completionTokens: number },
    model: string
  ): number {
    // Pricing as of 2025 (update as needed)
    const pricing: Record<string, { input: number; output: number }> = {
      'claude-sonnet-4-5-20250929': { input: 0.003, output: 0.015 },
      'claude-sonnet-4-20250514': { input: 0.003, output: 0.015 },
      'claude-3-5-haiku-20241022': { input: 0.0008, output: 0.004 },
    };
    
    const p = pricing[model] ?? pricing['claude-sonnet-4-5-20250929'];
    return (
      (usage.promptTokens / 1000) * p.input +
      (usage.completionTokens / 1000) * p.output
    );
  }
  
  private formatMessages(messages: Message[]): Message[] {
    return messages;  // AI SDK uses same format
  }
}
```

### 1.5 Provider Registry

```typescript
// src/ai/providers/registry.ts

export class ProviderRegistry {
  private providers = new Map<AIProvider, AIProviderClient>();
  private defaultProvider: AIProvider = 'anthropic';
  
  register(provider: AIProviderClient): void {
    this.providers.set(provider.name, provider);
  }
  
  get(name: AIProvider): AIProviderClient {
    const provider = this.providers.get(name);
    if (!provider) {
      throw new Error(`Provider '${name}' not registered`);
    }
    return provider;
  }
  
  getDefault(): AIProviderClient {
    return this.get(this.defaultProvider);
  }
  
  setDefault(name: AIProvider): void {
    if (!this.providers.has(name)) {
      throw new Error(`Cannot set default: provider '${name}' not registered`);
    }
    this.defaultProvider = name;
  }
  
  async findHealthy(): Promise<AIProviderClient | null> {
    for (const [, provider] of this.providers) {
      const health = await provider.healthCheck();
      if (health.available) {
        return provider;
      }
    }
    return null;
  }
}

// Global singleton
export const providers = new ProviderRegistry();
```

### 1.6 Unified Generation Service

```typescript
// src/ai/generation-service.ts

export class GenerationService {
  constructor(
    private registry: ProviderRegistry,
    private cache: GenerationCache,
    private validator: ValidationPipeline,
  ) {}
  
  async generate<T extends z.ZodType>(
    request: GenerationRequest<T>
  ): Promise<GenerationResult<z.infer<T>>> {
    // 1. Generate cache key
    const cacheKey = this.generateCacheKey(request);
    
    // 2. Check cache first
    if (request.cacheStrategy !== 'never') {
      const cached = await this.cache.get<z.infer<T>>(cacheKey);
      if (cached) {
        return {
          ...cached,
          cached: true,
        };
      }
    }
    
    // 3. Get provider
    const config = this.resolveConfig(request.config);
    const provider = this.registry.get(config.provider);
    
    // 4. Render prompt
    const renderedPrompt = request.prompt.render();
    
    // 5. Generate
    const response = await provider.generate({
      messages: renderedPrompt.messages,
      schema: request.schema,
      config,
    });
    
    // 6. Validate
    await this.validator.validate(response.output, request.schema);
    
    // 7. Build provenance
    const provenance = this.buildProvenance(request, response, cacheKey);
    
    // 8. Cache result
    const result: GenerationResult<z.infer<T>> = {
      output: response.output,
      provenance,
      usage: response.usage,
      cached: false,
    };
    
    await this.cache.set(cacheKey, result);
    
    return result;
  }
  
  private generateCacheKey(request: GenerationRequest<any>): string {
    // Content-addressable: hash of all inputs
    const content = JSON.stringify({
      prompt: request.prompt.getHash(),
      schema: request.schema._def,
      config: this.resolveConfig(request.config),
    });
    return createHash('sha256').update(content).digest('hex');
  }
  
  private resolveConfig(partial?: Partial<ModelConfig>): ModelConfig {
    return {
      provider: partial?.provider ?? 'anthropic',
      model: partial?.model ?? 'claude-sonnet-4-5-20250929',
      version: partial?.version ?? '2025-01-28',
      temperature: partial?.temperature ?? 0,
      maxTokens: partial?.maxTokens ?? 4096,
      timeout: partial?.timeout ?? 30000,
    };
  }
  
  private buildProvenance(
    request: GenerationRequest<any>,
    response: ProviderResponse<any>,
    cacheKey: string
  ): GenerationProvenance {
    return {
      specHash: request.metadata?.specHash ?? 'unknown',
      promptVersion: request.prompt.version,
      promptHash: request.prompt.getHash(),
      model: request.config?.model ?? 'claude-sonnet-4-5-20250929',
      generationHash: createHash('sha256')
        .update(JSON.stringify(response.output))
        .digest('hex'),
      timestamp: new Date().toISOString(),
      cacheKey,
      dspecVersion: DSPEC_VERSION,
    };
  }
}
```

---

## 2. Prompt System

### 2.1 Design Goals

- **Versioned:** Every prompt has a semantic version
- **Composable:** Build complex prompts from reusable parts
- **Cacheable:** Static parts are separated from dynamic parts for cache efficiency
- **Testable:** Prompts can be unit tested in isolation

### 2.2 Prompt Structure

```
prompts/
├── system/
│   ├── code-generation.md        # Base system prompt
│   └── typescript-expert.md      # TypeScript-specific rules
├── components/
│   ├── react/
│   │   ├── cva-variants.md       # CVA variant generation
│   │   ├── component-base.md     # Base component structure
│   │   └── test-generation.md    # Test generation
│   └── shared/
│       ├── prop-types.md         # Prop type inference
│       └── accessibility.md      # A11y requirements
├── examples/
│   ├── button-spec.yaml
│   ├── button-output.tsx
│   └── ...
└── registry.ts                   # Prompt metadata and loading
```

### 2.3 Prompt File Format

```markdown
<!-- prompts/components/react/cva-variants.md -->
---
id: react/cva-variants
version: 2.1.0
description: Generate CVA variant implementations from component spec
requires:
  - system/code-generation
  - system/typescript-expert
variables:
  - name: componentName
    required: true
  - name: variantSpec
    required: true
  - name: tokenImports
    required: false
    default: "@org/design-tokens"
---

# CVA Variant Generation

You are generating CVA (class-variance-authority) variant implementations.

## Input Format

The component specification will be provided in YAML format:

```yaml
{{variantSpec}}
```

## Output Requirements

1. **Imports:** Import tokens from `{{tokenImports}}`
2. **Type Safety:** All variants must be strongly typed
3. **Completeness:** Every variant combination must be styled
4. **Token Usage:** Use design tokens, never hardcoded values

## Example

### Input:
```yaml
name: Button
variants:
  - name: variant
    values: [primary, secondary]
  - name: size
    values: [sm, md, lg]
```

### Output:
```typescript
import { cva, type VariantProps } from 'class-variance-authority';
import { colors, spacing } from '{{tokenImports}}';

export const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md font-medium transition-colors',
  {
    variants: {
      variant: {
        primary: `bg-[${colors.interactive.primary}] text-white`,
        secondary: `bg-[${colors.interactive.secondary}] text-gray-900`,
      },
      size: {
        sm: `h-8 px-3 text-sm`,
        md: `h-10 px-4 text-base`,
        lg: `h-12 px-6 text-lg`,
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

export type ButtonVariants = VariantProps<typeof buttonVariants>;
```
```

### 2.4 Prompt Composition

```typescript
// src/ai/prompts/composer.ts

import { createHash } from 'crypto';
import matter from 'gray-matter';

interface PromptMetadata {
  id: string;
  version: string;
  description: string;
  requires?: string[];
  variables?: VariableDefinition[];
}

interface VariableDefinition {
  name: string;
  required: boolean;
  default?: string;
}

export class ComposedPrompt {
  private parts: PromptPart[] = [];
  private variables: Map<string, string> = new Map();
  
  readonly version: string;
  
  constructor(
    private metadata: PromptMetadata,
    private content: string,
  ) {
    this.version = `${metadata.id}@${metadata.version}`;
  }
  
  /**
   * Load a prompt from the registry
   */
  static async load(id: string): Promise<ComposedPrompt> {
    const raw = await loadPromptFile(id);
    const { data, content } = matter(raw);
    return new ComposedPrompt(data as PromptMetadata, content);
  }
  
  /**
   * Compose with dependencies
   */
  async compose(): Promise<ComposedPrompt> {
    if (this.metadata.requires) {
      for (const dep of this.metadata.requires) {
        const depPrompt = await ComposedPrompt.load(dep);
        this.parts.unshift({ type: 'dependency', prompt: depPrompt });
      }
    }
    this.parts.push({ type: 'main', prompt: this });
    return this;
  }
  
  /**
   * Set variable values
   */
  setVariable(name: string, value: string): this {
    this.variables.set(name, value);
    return this;
  }
  
  /**
   * Set multiple variables
   */
  setVariables(vars: Record<string, string>): this {
    for (const [name, value] of Object.entries(vars)) {
      this.variables.set(name, value);
    }
    return this;
  }
  
  /**
   * Render to messages format
   */
  render(): RenderedPrompt {
    this.validateVariables();
    
    const systemParts: string[] = [];
    const userParts: string[] = [];
    
    for (const part of this.parts) {
      const rendered = this.renderContent(part.prompt.content);
      if (part.type === 'dependency') {
        systemParts.push(rendered);
      } else {
        userParts.push(rendered);
      }
    }
    
    return {
      messages: [
        { role: 'system', content: systemParts.join('\n\n---\n\n') },
        { role: 'user', content: userParts.join('\n\n') },
      ],
      staticPrefix: systemParts.join('\n\n---\n\n'),
      dynamicSuffix: userParts.join('\n\n'),
    };
  }
  
  /**
   * Get content-addressable hash
   */
  getHash(): string {
    const rendered = this.render();
    const content = JSON.stringify({
      messages: rendered.messages,
      version: this.version,
    });
    return createHash('sha256').update(content).digest('hex');
  }
  
  /**
   * Get hash of static parts only (for provider cache optimization)
   */
  getStaticHash(): string {
    const rendered = this.render();
    return createHash('sha256').update(rendered.staticPrefix).digest('hex');
  }
  
  private renderContent(content: string): string {
    let rendered = content;
    
    // Replace {{variable}} with values
    for (const [name, value] of this.variables) {
      rendered = rendered.replace(
        new RegExp(`\\{\\{${name}\\}\\}`, 'g'),
        value
      );
    }
    
    // Replace remaining {{variable}} with defaults
    for (const varDef of this.metadata.variables ?? []) {
      if (varDef.default !== undefined) {
        rendered = rendered.replace(
          new RegExp(`\\{\\{${varDef.name}\\}\\}`, 'g'),
          varDef.default
        );
      }
    }
    
    return rendered;
  }
  
  private validateVariables(): void {
    const required = (this.metadata.variables ?? [])
      .filter(v => v.required)
      .map(v => v.name);
    
    const missing = required.filter(name => !this.variables.has(name));
    
    if (missing.length > 0) {
      throw new Error(
        `Missing required variables: ${missing.join(', ')}`
      );
    }
  }
}

interface PromptPart {
  type: 'dependency' | 'main';
  prompt: ComposedPrompt;
}

interface RenderedPrompt {
  messages: Message[];
  staticPrefix: string;
  dynamicSuffix: string;
}
```

### 2.5 Prompt Registry

```typescript
// src/ai/prompts/registry.ts

interface PromptEntry {
  id: string;
  version: string;
  path: string;
  deprecated?: boolean;
  replacedBy?: string;
}

export const promptRegistry: PromptEntry[] = [
  // System prompts
  { id: 'system/code-generation', version: '1.0.0', path: 'system/code-generation.md' },
  { id: 'system/typescript-expert', version: '1.0.0', path: 'system/typescript-expert.md' },
  
  // Component prompts
  { id: 'react/cva-variants', version: '2.1.0', path: 'components/react/cva-variants.md' },
  { id: 'react/component-base', version: '1.2.0', path: 'components/react/component-base.md' },
  { id: 'react/test-generation', version: '1.1.0', path: 'components/react/test-generation.md' },
  
  // Deprecated
  { 
    id: 'react/variants-v1', 
    version: '1.0.0', 
    path: 'components/react/variants-v1.md',
    deprecated: true,
    replacedBy: 'react/cva-variants',
  },
];

export function getPromptPath(id: string): string {
  const entry = promptRegistry.find(e => e.id === id);
  if (!entry) {
    throw new Error(`Unknown prompt: ${id}`);
  }
  if (entry.deprecated) {
    console.warn(
      `Prompt '${id}' is deprecated. Use '${entry.replacedBy}' instead.`
    );
  }
  return entry.path;
}
```

### 2.6 Example: Full Prompt Composition

```typescript
// Usage example
async function generateButtonVariants(spec: ComponentSpec) {
  // Load and compose prompt
  const prompt = await ComposedPrompt.load('react/cva-variants');
  await prompt.compose();
  
  // Set variables
  prompt.setVariables({
    componentName: spec.name,
    variantSpec: yaml.stringify(spec),
    tokenImports: '@org/design-tokens',
  });
  
  // Generate
  const result = await generationService.generate({
    prompt,
    schema: CVAOutputSchema,
    metadata: {
      specHash: hashSpec(spec),
    },
  });
  
  return result;
}
```

---

## 3. Caching Strategy

### 3.1 Design Goals

- **Determinism:** Same inputs always produce same outputs
- **Auditability:** Full history of what was generated and when
- **Git-Friendly:** Cache can be committed to repo
- **Fast CI:** CI uses cached responses by default

### 3.2 Cache Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Cache Layers                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐                                           │
│  │  L1: Memory      │  ← Hot cache, current session             │
│  │  (LRU, 100 items)│                                           │
│  └────────┬─────────┘                                           │
│           │ miss                                                 │
│           ▼                                                      │
│  ┌──────────────────┐                                           │
│  │  L2: Filesystem  │  ← Content-addressable, git-tracked       │
│  │  (.dspec/cache/) │                                           │
│  └────────┬─────────┘                                           │
│           │ miss                                                 │
│           ▼                                                      │
│  ┌──────────────────┐                                           │
│  │  L3: Provider    │  ← Anthropic/OpenAI prompt caching        │
│  │  (Automatic)     │    (5-10 min, reduces token cost)         │
│  └────────┬─────────┘                                           │
│           │ miss                                                 │
│           ▼                                                      │
│  ┌──────────────────┐                                           │
│  │  L4: LLM Request │  ← Actual API call                        │
│  └──────────────────┘                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 Cache Key Generation

```typescript
// src/ai/cache/keys.ts

import { createHash } from 'crypto';

interface CacheKeyInput {
  // Content factors (what we're generating)
  specContent: string;      // Full spec YAML
  specVersion: string;      // Spec schema version
  
  // Prompt factors (how we're generating)
  promptId: string;         // e.g., "react/cva-variants"
  promptVersion: string;    // e.g., "2.1.0"
  promptHash: string;       // SHA-256 of rendered prompt
  
  // Model factors (who is generating)
  provider: string;         // e.g., "anthropic"
  model: string;            // e.g., "claude-sonnet-4-5-20250929"
  temperature: number;      // Always 0 for determinism
}

export function generateCacheKey(input: CacheKeyInput): string {
  // Normalize and sort for consistent hashing
  const normalized = {
    spec: {
      content: normalizeWhitespace(input.specContent),
      version: input.specVersion,
    },
    prompt: {
      id: input.promptId,
      version: input.promptVersion,
      hash: input.promptHash,
    },
    model: {
      provider: input.provider,
      model: input.model,
      temperature: input.temperature,
    },
  };
  
  const serialized = JSON.stringify(normalized, Object.keys(normalized).sort());
  const hash = createHash('sha256').update(serialized).digest('hex');
  
  // Return prefixed for readability: gen_<first16chars>
  return `gen_${hash.substring(0, 16)}`;
}

function normalizeWhitespace(s: string): string {
  return s.trim().replace(/\r\n/g, '\n');
}

/**
 * Generate a short display key for logs
 */
export function shortKey(fullKey: string): string {
  return fullKey.substring(0, 12);
}
```

### 3.4 Filesystem Cache Implementation

```typescript
// src/ai/cache/filesystem.ts

import { mkdir, readFile, writeFile, readdir, stat } from 'fs/promises';
import { join } from 'path';

interface CacheEntry<T> {
  key: string;
  value: T;
  provenance: GenerationProvenance;
  usage: TokenUsage;
  createdAt: string;
}

export class FilesystemCache implements GenerationCache {
  constructor(
    private basePath: string = '.dspec/cache'
  ) {}
  
  async get<T>(key: string): Promise<GenerationResult<T> | null> {
    const path = this.getPath(key);
    
    try {
      const raw = await readFile(path, 'utf-8');
      const entry: CacheEntry<T> = JSON.parse(raw);
      
      return {
        output: entry.value,
        provenance: entry.provenance,
        usage: entry.usage,
        cached: true,
      };
    } catch (error) {
      if ((error as NodeJS.ErrnoException).code === 'ENOENT') {
        return null;
      }
      throw error;
    }
  }
  
  async set<T>(
    key: string,
    result: GenerationResult<T>
  ): Promise<void> {
    const path = this.getPath(key);
    
    // Ensure directory exists
    await mkdir(this.basePath, { recursive: true });
    
    const entry: CacheEntry<T> = {
      key,
      value: result.output,
      provenance: result.provenance,
      usage: result.usage,
      createdAt: new Date().toISOString(),
    };
    
    await writeFile(path, JSON.stringify(entry, null, 2), 'utf-8');
  }
  
  async has(key: string): Promise<boolean> {
    try {
      await stat(this.getPath(key));
      return true;
    } catch {
      return false;
    }
  }
  
  async delete(key: string): Promise<boolean> {
    try {
      await rm(this.getPath(key));
      return true;
    } catch {
      return false;
    }
  }
  
  async list(): Promise<string[]> {
    try {
      const files = await readdir(this.basePath);
      return files
        .filter(f => f.endsWith('.json'))
        .map(f => f.replace('.json', ''));
    } catch {
      return [];
    }
  }
  
  async stats(): Promise<CacheStats> {
    const keys = await this.list();
    let totalSize = 0;
    let totalCost = 0;
    
    for (const key of keys) {
      const result = await this.get(key);
      if (result) {
        totalCost += result.usage.estimatedCost;
      }
      const s = await stat(this.getPath(key));
      totalSize += s.size;
    }
    
    return {
      entries: keys.length,
      totalSizeBytes: totalSize,
      totalCostSaved: totalCost,
    };
  }
  
  private getPath(key: string): string {
    // Use subdirectories for better filesystem performance
    const subdir = key.substring(4, 6);  // First 2 chars after "gen_"
    return join(this.basePath, subdir, `${key}.json`);
  }
}

interface CacheStats {
  entries: number;
  totalSizeBytes: number;
  totalCostSaved: number;
}
```

### 3.5 Cache Directory Structure

```
.dspec/
├── cache/
│   ├── ab/
│   │   ├── gen_ab12cd34ef56.json
│   │   └── gen_ab98fe76dc54.json
│   ├── cd/
│   │   └── gen_cd34ef56ab12.json
│   └── ...
├── cache-index.json          # Optional: index for fast lookups
└── cache-stats.json          # Aggregated statistics
```

### 3.6 Cache Entry Format

```json
{
  "key": "gen_ab12cd34ef5678",
  "value": {
    "code": "export const buttonVariants = cva(...)",
    "imports": ["class-variance-authority", "@org/design-tokens"],
    "types": "export type ButtonVariants = VariantProps<typeof buttonVariants>"
  },
  "provenance": {
    "specHash": "sha256:abc123...",
    "promptVersion": "react/cva-variants@2.1.0",
    "promptHash": "sha256:def456...",
    "model": "claude-sonnet-4-5-20250929",
    "generationHash": "sha256:789xyz...",
    "timestamp": "2025-03-15T10:30:00Z",
    "cacheKey": "gen_ab12cd34ef5678",
    "dspecVersion": "1.2.0"
  },
  "usage": {
    "inputTokens": 1847,
    "outputTokens": 523,
    "estimatedCost": 0.0134
  },
  "createdAt": "2025-03-15T10:30:00Z"
}
```

### 3.7 CI Integration

```yaml
# .github/workflows/generate.yml
name: Generate Components

on:
  push:
    paths:
      - 'specs/**'
      - 'prompts/**'

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Restore cache from git
      - name: Restore generation cache
        run: |
          # Cache is committed to repo, already available
          echo "Cache entries: $(ls .dspec/cache/*/*.json 2>/dev/null | wc -l)"
      
      # Generate with cache
      - name: Generate components
        run: |
          dspec generate --cached
        env:
          # Only needed for cache misses
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      
      # Commit any new cache entries
      - name: Commit cache updates
        run: |
          git add .dspec/cache/
          git diff --cached --quiet || git commit -m "chore: update generation cache"
          git push
```

### 3.8 Cache Commands

```bash
# Show cache statistics
dspec cache stats
# Output:
# Cache entries: 47
# Total size: 1.2 MB
# Estimated cost saved: $12.34

# List cached generations
dspec cache list
# Output:
# gen_ab12cd34  Button variants  2025-03-15  $0.01
# gen_cd34ef56  Input styles     2025-03-14  $0.02
# ...

# Clear cache for specific component
dspec cache clear --component Button

# Clear all cache
dspec cache clear --all

# Export cache for sharing
dspec cache export --output cache-bundle.tar.gz

# Import shared cache
dspec cache import cache-bundle.tar.gz
```

---

## 4. Validation Pipeline

### 4.1 Design Goals

- **Multi-Layer:** Schema → Syntax → Semantic → Runtime
- **Fast Feedback:** Quick failures before expensive checks
- **Actionable Errors:** Clear messages for fixing issues
- **AI-Friendly:** Errors can be fed back to AI for self-correction

### 4.2 Validation Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Validation Pipeline                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [AI Output] ───► [Layer 1: Schema] ───► [Layer 2: Syntax]     │
│                        │                       │                │
│                        ▼                       ▼                │
│                   Zod Parse              TypeScript AST         │
│                   Type Check             Parse & Validate       │
│                        │                       │                │
│                        ├───────────────────────┤                │
│                        ▼                       ▼                │
│  [Layer 3: Semantic] ◄─┴───────────────────────┘                │
│        │                                                         │
│        ▼                                                         │
│   ESLint Rules                                                   │
│   Custom Validators                                              │
│   Token Reference Check                                          │
│        │                                                         │
│        ▼                                                         │
│  [Layer 4: Runtime] (Optional)                                   │
│        │                                                         │
│        ▼                                                         │
│   Test Execution                                                 │
│   Visual Regression                                              │
│        │                                                         │
│        ▼                                                         │
│  [PASS] or [FAIL with actionable errors]                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Output Schemas (Zod)

```typescript
// src/ai/schemas/component-output.ts

import { z } from 'zod';

/**
 * Schema for generated component code
 */
export const ComponentOutputSchema = z.object({
  // Main component file
  component: z.object({
    code: z.string()
      .min(50, 'Component code too short')
      .describe('The main React component implementation'),
    
    filename: z.string()
      .regex(/^[A-Z][a-zA-Z0-9]*\.tsx$/, 'Invalid component filename'),
  }),
  
  // Variant definitions (CVA)
  variants: z.object({
    code: z.string()
      .describe('CVA variant definitions'),
    
    filename: z.string()
      .regex(/^[a-z][a-zA-Z0-9]*\.variants\.ts$/, 'Invalid variants filename'),
  }).optional(),
  
  // Type definitions
  types: z.object({
    code: z.string()
      .describe('TypeScript type definitions'),
    
    filename: z.string()
      .regex(/^[a-z][a-zA-Z0-9]*\.types\.ts$/, 'Invalid types filename'),
  }),
  
  // Test file
  tests: z.object({
    code: z.string()
      .describe('Jest/Vitest test implementations'),
    
    filename: z.string()
      .regex(/^[A-Z][a-zA-Z0-9]*\.test\.tsx$/, 'Invalid test filename'),
  }),
  
  // Storybook stories
  stories: z.object({
    code: z.string()
      .describe('Storybook story definitions'),
    
    filename: z.string()
      .regex(/^[A-Z][a-zA-Z0-9]*\.stories\.tsx$/, 'Invalid stories filename'),
  }),
  
  // Required imports
  imports: z.array(z.string())
    .describe('NPM packages required by this component'),
  
  // Barrel export
  index: z.string()
    .describe('Index file content for barrel export'),
});

export type ComponentOutput = z.infer<typeof ComponentOutputSchema>;

/**
 * Schema for CVA variant output specifically
 */
export const CVAOutputSchema = z.object({
  code: z.string()
    .min(100, 'CVA code too short')
    .refine(
      (code) => code.includes('cva('),
      'Must include cva() call'
    )
    .refine(
      (code) => code.includes('variants:'),
      'Must define variants object'
    )
    .describe('CVA variant implementation'),
  
  imports: z.array(z.string())
    .min(1, 'Must have at least one import')
    .refine(
      (imports) => imports.includes('class-variance-authority'),
      'Must import class-variance-authority'
    ),
  
  types: z.string()
    .describe('Type export for variant props'),
  
  defaultVariants: z.record(z.string(), z.string())
    .describe('Default variant values'),
});

export type CVAOutput = z.infer<typeof CVAOutputSchema>;
```

### 4.4 Validation Pipeline Implementation

```typescript
// src/ai/validation/pipeline.ts

import { z } from 'zod';
import ts from 'typescript';
import { ESLint } from 'eslint';

interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
  warnings: ValidationWarning[];
}

interface ValidationError {
  layer: 'schema' | 'syntax' | 'semantic' | 'runtime';
  code: string;
  message: string;
  location?: {
    file?: string;
    line?: number;
    column?: number;
  };
  suggestion?: string;
}

interface ValidationWarning {
  layer: string;
  message: string;
}

export class ValidationPipeline {
  private eslint: ESLint;
  private customValidators: CustomValidator[] = [];
  
  constructor(config: ValidationConfig) {
    this.eslint = new ESLint({
      baseConfig: config.eslintConfig,
      useEslintrc: false,
    });
  }
  
  registerValidator(validator: CustomValidator): void {
    this.customValidators.push(validator);
  }
  
  async validate<T extends z.ZodType>(
    output: unknown,
    schema: T
  ): Promise<ValidationResult> {
    const errors: ValidationError[] = [];
    const warnings: ValidationWarning[] = [];
    
    // Layer 1: Schema validation
    const schemaResult = await this.validateSchema(output, schema);
    if (!schemaResult.valid) {
      return schemaResult;  // Fail fast on schema errors
    }
    
    const parsed = output as z.infer<T>;
    
    // Layer 2: Syntax validation (TypeScript)
    const syntaxResult = await this.validateSyntax(parsed);
    errors.push(...syntaxResult.errors);
    warnings.push(...syntaxResult.warnings);
    
    if (errors.some(e => e.layer === 'syntax')) {
      return { valid: false, errors, warnings };  // Fail fast on syntax
    }
    
    // Layer 3: Semantic validation (ESLint + custom)
    const semanticResult = await this.validateSemantic(parsed);
    errors.push(...semanticResult.errors);
    warnings.push(...semanticResult.warnings);
    
    return {
      valid: errors.length === 0,
      errors,
      warnings,
    };
  }
  
  private async validateSchema<T extends z.ZodType>(
    output: unknown,
    schema: T
  ): Promise<ValidationResult> {
    const result = schema.safeParse(output);
    
    if (result.success) {
      return { valid: true, errors: [], warnings: [] };
    }
    
    const errors: ValidationError[] = result.error.errors.map(err => ({
      layer: 'schema' as const,
      code: 'SCHEMA_VALIDATION_FAILED',
      message: `${err.path.join('.')}: ${err.message}`,
      suggestion: this.getSchemaSuggestion(err),
    }));
    
    return { valid: false, errors, warnings: [] };
  }
  
  private async validateSyntax(
    output: ComponentOutput
  ): Promise<ValidationResult> {
    const errors: ValidationError[] = [];
    const warnings: ValidationWarning[] = [];
    
    // Validate each code file
    const files = [
      { name: output.component.filename, code: output.component.code },
      { name: output.types.filename, code: output.types.code },
      { name: output.tests.filename, code: output.tests.code },
      { name: output.stories.filename, code: output.stories.code },
    ];
    
    if (output.variants) {
      files.push({ name: output.variants.filename, code: output.variants.code });
    }
    
    for (const file of files) {
      const result = this.parseTypeScript(file.code, file.name);
      
      for (const diag of result.diagnostics) {
        errors.push({
          layer: 'syntax',
          code: `TS${diag.code}`,
          message: ts.flattenDiagnosticMessageText(diag.messageText, '\n'),
          location: diag.start ? {
            file: file.name,
            line: this.getLineNumber(file.code, diag.start),
          } : undefined,
        });
      }
    }
    
    return { valid: errors.length === 0, errors, warnings };
  }
  
  private parseTypeScript(
    code: string,
    filename: string
  ): { ast: ts.SourceFile; diagnostics: ts.Diagnostic[] } {
    const sourceFile = ts.createSourceFile(
      filename,
      code,
      ts.ScriptTarget.Latest,
      true,
      ts.ScriptKind.TSX
    );
    
    // Get syntactic diagnostics (parse errors)
    const diagnostics = [
      ...sourceFile.parseDiagnostics,
    ];
    
    return { ast: sourceFile, diagnostics };
  }
  
  private async validateSemantic(
    output: ComponentOutput
  ): Promise<ValidationResult> {
    const errors: ValidationError[] = [];
    const warnings: ValidationWarning[] = [];
    
    // ESLint validation
    const files = [
      { path: output.component.filename, code: output.component.code },
      { path: output.types.filename, code: output.types.code },
    ];
    
    for (const file of files) {
      const results = await this.eslint.lintText(file.code, {
        filePath: file.path,
      });
      
      for (const result of results) {
        for (const msg of result.messages) {
          if (msg.severity === 2) {
            errors.push({
              layer: 'semantic',
              code: msg.ruleId ?? 'ESLINT_ERROR',
              message: msg.message,
              location: {
                file: file.path,
                line: msg.line,
                column: msg.column,
              },
            });
          } else {
            warnings.push({
              layer: 'semantic',
              message: `${file.path}:${msg.line} - ${msg.message}`,
            });
          }
        }
      }
    }
    
    // Custom validators
    for (const validator of this.customValidators) {
      const result = await validator.validate(output);
      errors.push(...result.errors);
      warnings.push(...result.warnings);
    }
    
    return { valid: errors.length === 0, errors, warnings };
  }
  
  private getSchemaSuggestion(error: z.ZodIssue): string {
    switch (error.code) {
      case 'too_small':
        return 'Generate more content to meet minimum requirements';
      case 'invalid_type':
        return `Expected ${error.expected}, received ${error.received}`;
      case 'custom':
        return error.message;
      default:
        return 'Check the schema requirements';
    }
  }
  
  private getLineNumber(code: string, position: number): number {
    return code.substring(0, position).split('\n').length;
  }
}

interface CustomValidator {
  name: string;
  validate(output: ComponentOutput): Promise<{
    errors: ValidationError[];
    warnings: ValidationWarning[];
  }>;
}
```

### 4.5 Custom Validators

```typescript
// src/ai/validation/custom-validators.ts

/**
 * Validates that all token references resolve to actual tokens
 */
export class TokenReferenceValidator implements CustomValidator {
  name = 'token-reference';
  
  constructor(private tokenRegistry: TokenRegistry) {}
  
  async validate(output: ComponentOutput): Promise<ValidationResult> {
    const errors: ValidationError[] = [];
    const warnings: ValidationWarning[] = [];
    
    // Find all token references like $colors.primary or tokens.spacing.md
    const tokenPattern = /\$?(?:colors|spacing|typography|tokens)\.[a-zA-Z0-9.]+/g;
    
    const allCode = [
      output.component.code,
      output.variants?.code ?? '',
    ].join('\n');
    
    const matches = allCode.matchAll(tokenPattern);
    
    for (const match of matches) {
      const tokenPath = match[0].replace(/^\$/, '');
      if (!this.tokenRegistry.has(tokenPath)) {
        errors.push({
          layer: 'semantic',
          code: 'INVALID_TOKEN_REFERENCE',
          message: `Token '${tokenPath}' not found in token registry`,
          suggestion: `Available tokens: ${this.tokenRegistry.suggest(tokenPath).join(', ')}`,
        });
      }
    }
    
    return { errors, warnings };
  }
}

/**
 * Validates component follows accessibility patterns
 */
export class AccessibilityValidator implements CustomValidator {
  name = 'accessibility';
  
  async validate(output: ComponentOutput): Promise<ValidationResult> {
    const errors: ValidationError[] = [];
    const warnings: ValidationWarning[] = [];
    
    const code = output.component.code;
    
    // Check for interactive elements without accessible names
    if (code.includes('<button') && !code.includes('aria-label')) {
      warnings.push({
        layer: 'semantic',
        message: 'Button may need aria-label for icon-only variants',
      });
    }
    
    // Check for role usage
    if (code.includes('onClick') && !code.includes('role=')) {
      const hasSemanticElement = ['<button', '<a ', '<input'].some(el => 
        code.includes(el)
      );
      
      if (!hasSemanticElement) {
        errors.push({
          layer: 'semantic',
          code: 'A11Y_MISSING_ROLE',
          message: 'Clickable element should use semantic HTML or have role attribute',
        });
      }
    }
    
    return { errors, warnings };
  }
}

/**
 * Validates generated tests have assertions
 */
export class TestQualityValidator implements CustomValidator {
  name = 'test-quality';
  
  async validate(output: ComponentOutput): Promise<ValidationResult> {
    const errors: ValidationError[] = [];
    const warnings: ValidationWarning[] = [];
    
    const testCode = output.tests.code;
    
    // Count test blocks and assertions
    const testCount = (testCode.match(/\bit\s*\(/g) ?? []).length;
    const assertCount = (testCode.match(/expect\s*\(/g) ?? []).length;
    
    if (testCount === 0) {
      errors.push({
        layer: 'semantic',
        code: 'NO_TESTS',
        message: 'Test file has no test cases',
      });
    } else if (assertCount < testCount) {
      warnings.push({
        layer: 'semantic',
        message: `Some tests may be missing assertions (${assertCount} expects for ${testCount} tests)`,
      });
    }
    
    // Check for render tests
    if (!testCode.includes('render(')) {
      errors.push({
        layer: 'semantic',
        code: 'NO_RENDER_TESTS',
        message: 'Test file should include render tests',
      });
    }
    
    return { errors, warnings };
  }
}
```

### 4.6 Error Formatting for AI Self-Correction

```typescript
// src/ai/validation/error-formatter.ts

/**
 * Format errors for inclusion in retry prompt
 */
export function formatErrorsForRetry(
  errors: ValidationError[]
): string {
  const grouped = groupBy(errors, e => e.layer);
  
  let output = 'The previous generation failed validation:\n\n';
  
  for (const [layer, layerErrors] of Object.entries(grouped)) {
    output += `## ${layer.toUpperCase()} ERRORS\n\n`;
    
    for (const error of layerErrors) {
      output += `- **${error.code}**: ${error.message}\n`;
      if (error.location) {
        output += `  Location: ${error.location.file}:${error.location.line}\n`;
      }
      if (error.suggestion) {
        output += `  Suggestion: ${error.suggestion}\n`;
      }
      output += '\n';
    }
  }
  
  output += '\nPlease fix these issues and regenerate.';
  
  return output;
}
```

---

## 5. Error Handling and Retry Logic

### 5.1 Design Goals

- **Automatic Recovery:** Retry transient failures without user intervention
- **Self-Correction:** Feed errors back to AI for intelligent fixes
- **Exponential Backoff:** Don't hammer failing services
- **Circuit Breaker:** Stop trying after repeated failures

### 5.2 Error Categories

```typescript
// src/ai/errors/types.ts

export enum ErrorCategory {
  // Transient - retry with backoff
  RATE_LIMIT = 'rate_limit',
  TIMEOUT = 'timeout',
  SERVICE_UNAVAILABLE = 'service_unavailable',
  
  // Self-correctable - retry with error feedback
  VALIDATION_FAILED = 'validation_failed',
  SCHEMA_MISMATCH = 'schema_mismatch',
  SYNTAX_ERROR = 'syntax_error',
  
  // Fatal - don't retry
  AUTHENTICATION = 'authentication',
  QUOTA_EXCEEDED = 'quota_exceeded',
  INVALID_REQUEST = 'invalid_request',
  CONTENT_POLICY = 'content_policy',
  
  // Unknown - retry once
  UNKNOWN = 'unknown',
}

export function categorizeError(error: unknown): ErrorCategory {
  if (error instanceof APIError) {
    switch (error.status) {
      case 429: return ErrorCategory.RATE_LIMIT;
      case 408: return ErrorCategory.TIMEOUT;
      case 503: return ErrorCategory.SERVICE_UNAVAILABLE;
      case 401: return ErrorCategory.AUTHENTICATION;
      case 402: return ErrorCategory.QUOTA_EXCEEDED;
      case 400: return ErrorCategory.INVALID_REQUEST;
      default: return ErrorCategory.UNKNOWN;
    }
  }
  
  if (error instanceof ValidationError) {
    return ErrorCategory.VALIDATION_FAILED;
  }
  
  return ErrorCategory.UNKNOWN;
}
```

### 5.3 Retry Configuration

```typescript
// src/ai/errors/retry-config.ts

export interface RetryConfig {
  // Maximum retry attempts
  maxRetries: number;
  
  // Base delay in ms (doubles each retry)
  baseDelayMs: number;
  
  // Maximum delay cap
  maxDelayMs: number;
  
  // Jitter factor (0-1, adds randomness to prevent thundering herd)
  jitterFactor: number;
  
  // Categories that should be retried
  retryableCategories: Set<ErrorCategory>;
  
  // Categories that should include error feedback in prompt
  selfCorrectableCategories: Set<ErrorCategory>;
}

export const defaultRetryConfig: RetryConfig = {
  maxRetries: 3,
  baseDelayMs: 1000,
  maxDelayMs: 30000,
  jitterFactor: 0.2,
  
  retryableCategories: new Set([
    ErrorCategory.RATE_LIMIT,
    ErrorCategory.TIMEOUT,
    ErrorCategory.SERVICE_UNAVAILABLE,
    ErrorCategory.VALIDATION_FAILED,
    ErrorCategory.SCHEMA_MISMATCH,
    ErrorCategory.SYNTAX_ERROR,
    ErrorCategory.UNKNOWN,
  ]),
  
  selfCorrectableCategories: new Set([
    ErrorCategory.VALIDATION_FAILED,
    ErrorCategory.SCHEMA_MISMATCH,
    ErrorCategory.SYNTAX_ERROR,
  ]),
};
```

### 5.4 Retry Logic Implementation

```typescript
// src/ai/errors/retry.ts

export class RetryHandler {
  private consecutiveFailures = 0;
  private circuitOpen = false;
  private circuitOpenUntil?: Date;
  
  constructor(
    private config: RetryConfig = defaultRetryConfig,
    private circuitBreakerThreshold = 5,
    private circuitBreakerDurationMs = 60000,
  ) {}
  
  async execute<T>(
    operation: () => Promise<T>,
    context: RetryContext
  ): Promise<T> {
    // Check circuit breaker
    if (this.isCircuitOpen()) {
      throw new CircuitOpenError(
        `Circuit breaker open until ${this.circuitOpenUntil?.toISOString()}`
      );
    }
    
    let lastError: unknown;
    let attempt = 0;
    
    while (attempt <= this.config.maxRetries) {
      try {
        const result = await operation();
        this.recordSuccess();
        return result;
        
      } catch (error) {
        lastError = error;
        const category = categorizeError(error);
        
        // Should we retry?
        if (!this.shouldRetry(category, attempt)) {
          this.recordFailure();
          throw error;
        }
        
        // Calculate delay
        const delay = this.calculateDelay(attempt);
        
        // Log retry attempt
        context.onRetry?.({
          attempt: attempt + 1,
          maxRetries: this.config.maxRetries,
          error,
          category,
          delayMs: delay,
        });
        
        // Wait before retry
        await sleep(delay);
        
        // Modify prompt for self-correctable errors
        if (this.config.selfCorrectableCategories.has(category)) {
          context.modifyPrompt?.(error);
        }
        
        attempt++;
      }
    }
    
    this.recordFailure();
    throw lastError;
  }
  
  private shouldRetry(category: ErrorCategory, attempt: number): boolean {
    if (attempt >= this.config.maxRetries) {
      return false;
    }
    return this.config.retryableCategories.has(category);
  }
  
  private calculateDelay(attempt: number): number {
    // Exponential backoff with jitter
    const exponentialDelay = this.config.baseDelayMs * Math.pow(2, attempt);
    const cappedDelay = Math.min(exponentialDelay, this.config.maxDelayMs);
    const jitter = cappedDelay * this.config.jitterFactor * Math.random();
    return Math.floor(cappedDelay + jitter);
  }
  
  private recordSuccess(): void {
    this.consecutiveFailures = 0;
    this.circuitOpen = false;
  }
  
  private recordFailure(): void {
    this.consecutiveFailures++;
    
    if (this.consecutiveFailures >= this.circuitBreakerThreshold) {
      this.circuitOpen = true;
      this.circuitOpenUntil = new Date(
        Date.now() + this.circuitBreakerDurationMs
      );
    }
  }
  
  private isCircuitOpen(): boolean {
    if (!this.circuitOpen) return false;
    
    if (this.circuitOpenUntil && new Date() > this.circuitOpenUntil) {
      // Allow one attempt through (half-open state)
      this.circuitOpen = false;
      return false;
    }
    
    return true;
  }
}

interface RetryContext {
  onRetry?: (info: RetryInfo) => void;
  modifyPrompt?: (error: unknown) => void;
}

interface RetryInfo {
  attempt: number;
  maxRetries: number;
  error: unknown;
  category: ErrorCategory;
  delayMs: number;
}
```

### 5.5 Self-Correction Flow

```typescript
// src/ai/generation-service.ts (addition)

async generateWithRetry<T extends z.ZodType>(
  request: GenerationRequest<T>
): Promise<GenerationResult<z.infer<T>>> {
  const retryHandler = new RetryHandler();
  let currentPrompt = request.prompt;
  const validationErrors: ValidationError[] = [];
  
  return retryHandler.execute(
    async () => {
      // 1. Generate
      const response = await this.generate({
        ...request,
        prompt: currentPrompt,
      });
      
      // 2. Validate
      const validation = await this.validator.validate(
        response.output,
        request.schema
      );
      
      if (!validation.valid) {
        // Store errors for self-correction
        validationErrors.push(...validation.errors);
        throw new ValidationError(validation.errors);
      }
      
      return response;
    },
    {
      onRetry: (info) => {
        console.log(
          `Retry ${info.attempt}/${info.maxRetries} ` +
          `after ${info.delayMs}ms (${info.category})`
        );
      },
      
      modifyPrompt: (error) => {
        if (error instanceof ValidationError) {
          // Append error feedback to prompt for self-correction
          const errorFeedback = formatErrorsForRetry(validationErrors);
          currentPrompt = currentPrompt.appendUserMessage(errorFeedback);
        }
      },
    }
  );
}
```

### 5.6 Retry Telemetry

```typescript
// src/ai/telemetry/retry-metrics.ts

export interface RetryMetrics {
  totalAttempts: number;
  successfulFirstAttempt: number;
  successfulAfterRetry: number;
  failedAfterRetries: number;
  
  byCategory: Record<ErrorCategory, {
    count: number;
    avgRetries: number;
    successRate: number;
  }>;
  
  avgRetryDelay: number;
  circuitBreakerTrips: number;
}

export class RetryTelemetry {
  private metrics: RetryMetrics = {
    totalAttempts: 0,
    successfulFirstAttempt: 0,
    successfulAfterRetry: 0,
    failedAfterRetries: 0,
    byCategory: {},
    avgRetryDelay: 0,
    circuitBreakerTrips: 0,
  };
  
  record(event: RetryEvent): void {
    this.metrics.totalAttempts++;
    
    if (event.success) {
      if (event.attempts === 1) {
        this.metrics.successfulFirstAttempt++;
      } else {
        this.metrics.successfulAfterRetry++;
      }
    } else {
      this.metrics.failedAfterRetries++;
    }
    
    // Track by category
    const cat = this.metrics.byCategory[event.category] ??= {
      count: 0,
      avgRetries: 0,
      successRate: 0,
    };
    cat.count++;
    // ... update averages
  }
  
  getMetrics(): RetryMetrics {
    return { ...this.metrics };
  }
}
```

---

## 6. Cost Management

### 6.1 Design Goals

- **Visibility:** Know exactly what every generation costs
- **Budgets:** Set limits and alerts
- **Optimization:** Identify and reduce expensive operations
- **Projections:** Forecast monthly costs

### 6.2 Cost Tracking

```typescript
// src/ai/cost/tracker.ts

interface CostRecord {
  timestamp: string;
  operation: string;
  component?: string;
  
  // Token usage
  inputTokens: number;
  outputTokens: number;
  cachedTokens: number;
  
  // Cost breakdown
  inputCost: number;
  outputCost: number;
  totalCost: number;
  
  // Cache impact
  cacheHit: boolean;
  costSavedByCache: number;
  
  // Metadata
  model: string;
  promptVersion: string;
  retryAttempts: number;
}

export class CostTracker {
  private records: CostRecord[] = [];
  private persistPath: string;
  
  constructor(persistPath: string = '.dspec/costs.jsonl') {
    this.persistPath = persistPath;
  }
  
  record(record: CostRecord): void {
    this.records.push(record);
    this.persist(record);
  }
  
  private async persist(record: CostRecord): Promise<void> {
    await appendFile(
      this.persistPath,
      JSON.stringify(record) + '\n'
    );
  }
  
  /**
   * Get cost summary for time period
   */
  async getSummary(
    since: Date,
    until: Date = new Date()
  ): Promise<CostSummary> {
    const records = await this.loadRecords(since, until);
    
    return {
      period: { since, until },
      
      totalCost: sum(records.map(r => r.totalCost)),
      totalSaved: sum(records.map(r => r.costSavedByCache)),
      
      byComponent: groupAndSum(records, r => r.component ?? 'unknown', r => r.totalCost),
      byModel: groupAndSum(records, r => r.model, r => r.totalCost),
      byOperation: groupAndSum(records, r => r.operation, r => r.totalCost),
      
      tokenUsage: {
        input: sum(records.map(r => r.inputTokens)),
        output: sum(records.map(r => r.outputTokens)),
        cached: sum(records.map(r => r.cachedTokens)),
      },
      
      cacheMetrics: {
        hits: records.filter(r => r.cacheHit).length,
        misses: records.filter(r => !r.cacheHit).length,
        hitRate: records.filter(r => r.cacheHit).length / records.length,
        totalSaved: sum(records.map(r => r.costSavedByCache)),
      },
      
      avgCostPerComponent: sum(records.map(r => r.totalCost)) / 
        new Set(records.map(r => r.component)).size,
    };
  }
  
  /**
   * Project monthly cost based on current usage
   */
  async projectMonthlyCost(): Promise<CostProjection> {
    const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
    const summary = await this.getSummary(thirtyDaysAgo);
    
    const daysInPeriod = 30;
    const dailyAvg = summary.totalCost / daysInPeriod;
    
    return {
      dailyAverage: dailyAvg,
      projectedMonthly: dailyAvg * 30,
      projectedYearly: dailyAvg * 365,
      
      // Breakdown by component
      componentProjections: Object.entries(summary.byComponent).map(
        ([component, cost]) => ({
          component,
          monthlyProjection: (cost / daysInPeriod) * 30,
        })
      ),
      
      // Confidence based on data points
      confidence: records.length > 100 ? 'high' : 
                  records.length > 30 ? 'medium' : 'low',
    };
  }
}

interface CostSummary {
  period: { since: Date; until: Date };
  totalCost: number;
  totalSaved: number;
  byComponent: Record<string, number>;
  byModel: Record<string, number>;
  byOperation: Record<string, number>;
  tokenUsage: {
    input: number;
    output: number;
    cached: number;
  };
  cacheMetrics: {
    hits: number;
    misses: number;
    hitRate: number;
    totalSaved: number;
  };
  avgCostPerComponent: number;
}

interface CostProjection {
  dailyAverage: number;
  projectedMonthly: number;
  projectedYearly: number;
  componentProjections: Array<{
    component: string;
    monthlyProjection: number;
  }>;
  confidence: 'low' | 'medium' | 'high';
}
```

### 6.3 Budget Enforcement

```typescript
// src/ai/cost/budget.ts

interface BudgetConfig {
  daily?: number;
  weekly?: number;
  monthly?: number;
  perComponent?: number;
  alertThreshold: number;  // 0-1, e.g., 0.8 = alert at 80%
}

export class BudgetEnforcer {
  constructor(
    private config: BudgetConfig,
    private tracker: CostTracker,
    private alertHandler: AlertHandler,
  ) {}
  
  async checkBudget(
    operation: string,
    estimatedCost: number
  ): Promise<BudgetCheckResult> {
    const now = new Date();
    
    // Check daily budget
    if (this.config.daily) {
      const today = startOfDay(now);
      const summary = await this.tracker.getSummary(today);
      const remaining = this.config.daily - summary.totalCost;
      
      if (remaining < estimatedCost) {
        return {
          allowed: false,
          reason: `Daily budget exhausted ($${summary.totalCost.toFixed(2)} / $${this.config.daily})`,
        };
      }
      
      if (remaining < this.config.daily * (1 - this.config.alertThreshold)) {
        this.alertHandler.warn(
          `Daily budget ${(summary.totalCost / this.config.daily * 100).toFixed(0)}% used`
        );
      }
    }
    
    // Check monthly budget
    if (this.config.monthly) {
      const monthStart = startOfMonth(now);
      const summary = await this.tracker.getSummary(monthStart);
      const remaining = this.config.monthly - summary.totalCost;
      
      if (remaining < estimatedCost) {
        return {
          allowed: false,
          reason: `Monthly budget exhausted ($${summary.totalCost.toFixed(2)} / $${this.config.monthly})`,
        };
      }
    }
    
    return { allowed: true };
  }
}

interface BudgetCheckResult {
  allowed: boolean;
  reason?: string;
}
```

### 6.4 Cost Optimization Recommendations

```typescript
// src/ai/cost/optimizer.ts

export class CostOptimizer {
  constructor(private tracker: CostTracker) {}
  
  async analyze(): Promise<OptimizationReport> {
    const summary = await this.tracker.getSummary(
      new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)
    );
    
    const recommendations: Recommendation[] = [];
    
    // 1. Low cache hit rate
    if (summary.cacheMetrics.hitRate < 0.5) {
      recommendations.push({
        priority: 'high',
        category: 'caching',
        title: 'Improve cache hit rate',
        description: `Current cache hit rate is ${(summary.cacheMetrics.hitRate * 100).toFixed(0)}%. ` +
          `Consider committing .dspec/cache/ to git or sharing cache across CI runs.`,
        estimatedSavings: summary.totalCost * 0.3,
      });
    }
    
    // 2. High-cost components
    const sortedComponents = Object.entries(summary.byComponent)
      .sort(([, a], [, b]) => b - a);
    
    if (sortedComponents.length > 0) {
      const [topComponent, topCost] = sortedComponents[0];
      if (topCost > summary.totalCost * 0.3) {
        recommendations.push({
          priority: 'medium',
          category: 'component',
          title: `Optimize ${topComponent} generation`,
          description: `${topComponent} accounts for ${(topCost / summary.totalCost * 100).toFixed(0)}% of costs. ` +
            `Consider simplifying spec or using static-only generation.`,
          estimatedSavings: topCost * 0.5,
        });
      }
    }
    
    // 3. Model selection
    if (summary.byModel['claude-sonnet-4-5-20250929'] > summary.totalCost * 0.5) {
      recommendations.push({
        priority: 'low',
        category: 'model',
        title: 'Consider using Haiku for simple components',
        description: 'Sonnet is used for majority of generations. ' +
          'Simple components may work well with cheaper Haiku model.',
        estimatedSavings: summary.totalCost * 0.2,
      });
    }
    
    return {
      currentMonthlyCost: summary.totalCost,
      recommendations,
      potentialSavings: sum(recommendations.map(r => r.estimatedSavings ?? 0)),
    };
  }
}

interface Recommendation {
  priority: 'high' | 'medium' | 'low';
  category: 'caching' | 'component' | 'model' | 'prompt';
  title: string;
  description: string;
  estimatedSavings?: number;
}

interface OptimizationReport {
  currentMonthlyCost: number;
  recommendations: Recommendation[];
  potentialSavings: number;
}
```

### 6.5 Cost CLI Commands

```bash
# Show cost summary
dspec cost summary
# Output:
# Period: Last 30 days
# Total cost: $45.67
# Saved by cache: $123.45
# 
# By Component:
#   Button: $5.23 (11%)
#   Select: $12.34 (27%)
#   Input: $8.90 (19%)
#   ...
# 
# Cache hit rate: 73%

# Show projections
dspec cost forecast
# Output:
# Daily average: $1.52
# Monthly projection: $45.67
# Yearly projection: $547.80
# Confidence: high (147 data points)

# Show optimization suggestions
dspec cost optimize
# Output:
# Current monthly: $45.67
# Potential savings: $18.50 (40%)
# 
# Recommendations:
# [HIGH] Improve cache hit rate
#   Current: 73% → Target: 90%
#   Estimated savings: $12.00/month
# 
# [MEDIUM] Optimize Select generation
#   Accounts for 27% of costs
#   Consider static-only for this component

# Set budget alerts
dspec cost budget --monthly 100 --alert-at 80
# Output:
# Budget set: $100/month
# Alert threshold: 80% ($80)
```

---

## 7. Implementation Checklist

### Phase 0 (Validation)

- [ ] Set up Vercel AI SDK with Anthropic provider
- [ ] Create basic Zod schema for Button output
- [ ] Implement content-addressable caching (file-based)
- [ ] Test 20 Button generations, measure pass rate
- [ ] Document cost of 20 generations

### Phase 1 (Tokens)

- [ ] N/A - No AI in Phase 1

### Phase 2 (First Component)

- [ ] Provider abstraction complete
- [ ] Prompt versioning system
- [ ] Full validation pipeline (schema + syntax + semantic)
- [ ] Retry with self-correction
- [ ] Cost tracking per generation
- [ ] Cache committed to git
- [ ] CI uses cached responses

### Phase 3 (Component Library)

- [ ] Custom validators (tokens, a11y, tests)
- [ ] Budget enforcement
- [ ] Cost optimization recommendations
- [ ] Monitoring dashboard
- [ ] 95% pass rate maintained

### Phase 4 (Stabilization)

- [ ] Circuit breaker tested
- [ ] Retry telemetry dashboard
- [ ] Cost projections accurate
- [ ] Cache hit rate > 80%

---

## Appendix A: Example Prompts

### System Prompt: Code Generation Base

```markdown
---
id: system/code-generation
version: 1.0.0
---

You are an expert TypeScript/React code generator for a design system.

## Core Principles

1. **Type Safety:** All code must be strictly typed. No `any`, no type assertions.
2. **Design Tokens:** Always use design tokens from `@org/design-tokens`. Never hardcode colors, spacing, or typography.
3. **Accessibility:** Follow WCAG 2.1 AA. Include proper ARIA attributes, keyboard navigation.
4. **Testing:** Generated tests must be meaningful. Test behavior, not implementation.
5. **Completeness:** Generated code must compile and pass linting with zero errors.

## Output Format

Always respond with valid JSON matching the provided schema. Do not include markdown code fences or explanations outside the JSON.

## Common Patterns

### CVA Variants
Use `class-variance-authority` for variant styling:
```typescript
import { cva, type VariantProps } from 'class-variance-authority';

export const componentVariants = cva('base-classes', {
  variants: { ... },
  defaultVariants: { ... },
});
```

### forwardRef Pattern
Always use forwardRef for components that render DOM elements:
```typescript
const Component = React.forwardRef<HTMLElement, Props>((props, ref) => {
  return <element ref={ref} {...props} />;
});
Component.displayName = 'Component';
```
```

### Component Prompt: React CVA Variants

```markdown
---
id: react/cva-variants
version: 2.1.0
requires:
  - system/code-generation
variables:
  - name: componentName
    required: true
  - name: spec
    required: true
  - name: tokenPackage
    default: "@org/design-tokens"
---

# Generate CVA Variants for {{componentName}}

## Component Specification

```yaml
{{spec}}
```

## Requirements

1. Import tokens from `{{tokenPackage}}`
2. Define all variants from the specification
3. Include compound variants where styling depends on multiple variant values
4. Export typed variant props
5. Set sensible default variants

## Token Usage

Map specification tokens to imports:
- `$colors.*` → `import { colors } from '{{tokenPackage}}'`
- `$spacing.*` → `import { spacing } from '{{tokenPackage}}'`
- `$typography.*` → `import { typography } from '{{tokenPackage}}'`

Generate the CVA implementation now.
```

---

## Appendix B: Caching Pseudocode

```typescript
// Simplified caching flow

async function generate(request: GenerationRequest): Promise<GenerationResult> {
  // 1. Build deterministic cache key
  const cacheKey = sha256({
    specContent: request.spec,
    promptVersion: request.prompt.version,
    promptHash: sha256(request.prompt.render()),
    model: request.model,
    temperature: 0,  // Always 0 for determinism
  });
  
  // 2. Check L1 (memory)
  if (memoryCache.has(cacheKey)) {
    metrics.record('cache_hit', { layer: 'memory' });
    return memoryCache.get(cacheKey);
  }
  
  // 3. Check L2 (filesystem)
  const cachePath = `.dspec/cache/${cacheKey.slice(0,2)}/${cacheKey}.json`;
  if (await fileExists(cachePath)) {
    const cached = await readJson(cachePath);
    memoryCache.set(cacheKey, cached);  // Promote to L1
    metrics.record('cache_hit', { layer: 'filesystem' });
    return cached;
  }
  
  // 4. Cache miss - call LLM
  metrics.record('cache_miss');
  
  const response = await llmClient.generate({
    messages: request.prompt.render(),
    schema: request.schema,
    temperature: 0,
  });
  
  // 5. Validate
  const validation = await validate(response.output, request.schema);
  if (!validation.valid) {
    throw new ValidationError(validation.errors);
  }
  
  // 6. Build result with provenance
  const result: GenerationResult = {
    output: response.output,
    provenance: {
      cacheKey,
      promptVersion: request.prompt.version,
      model: request.model,
      timestamp: new Date().toISOString(),
      generationHash: sha256(response.output),
    },
    usage: response.usage,
    cached: false,
  };
  
  // 7. Store in all cache layers
  memoryCache.set(cacheKey, result);
  await writeJson(cachePath, result);
  
  // 8. Track cost
  costTracker.record({
    ...result.usage,
    operation: 'generate',
    component: request.metadata?.component,
    cacheHit: false,
  });
  
  return result;
}
```

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2026-01-28 | Architecture Team | Initial draft |
