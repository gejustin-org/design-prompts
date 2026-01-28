# Token Pipeline Architecture

> **Status:** Architecture Specification  
> **Phase:** 1 (Token Foundation)  
> **Last Updated:** January 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Token Schema Design](#1-token-schema-design)
3. [Style Dictionary Integration](#2-style-dictionary-integration)
4. [Custom Formats](#3-custom-formats)
5. [Intermediate Representation (IR) Types](#4-intermediate-representation-ir-types)
6. [Integration with Component Generation](#5-integration-with-component-generation)
7. [Pipeline Configuration](#6-pipeline-configuration)
8. [Error Handling](#7-error-handling)

---

## Overview

### Architecture Principles

1. **DTCG-Native** — Adopt W3C Design Tokens Community Group format as canonical source
2. **Style Dictionary Core** — Leverage Style Dictionary v4 as the compilation engine
3. **Extension over Modification** — Custom formats and transforms, not forks
4. **Type-Safe IR** — Strong TypeScript types for all intermediate representations
5. **Deterministic Output** — Same input always produces identical output

### High-Level Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         TOKEN SOURCES                                │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │  YAML Specs  │    │ Figma Export │    │  tokens.json (DTCG)  │  │
│  │  (authored)  │    │ (Tokens      │    │  (canonical source)  │  │
│  │              │    │  Studio)     │    │                      │  │
│  └──────┬───────┘    └──────┬───────┘    └──────────┬───────────┘  │
│         │                   │                       │               │
│         └───────────────────┴───────────┬───────────┘               │
│                                         │                           │
│                                         ▼                           │
│                              ┌──────────────────┐                   │
│                              │  YAML → DTCG     │                   │
│                              │  Normalizer      │                   │
│                              └────────┬─────────┘                   │
│                                       │                             │
└───────────────────────────────────────│─────────────────────────────┘
                                        │
                                        ▼
┌───────────────────────────────────────────────────────────────────────┐
│                          TOKEN IR (Internal)                          │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  TokenIR {                                                      │  │
│  │    meta: { version, generated, source }                         │  │
│  │    tokens: { colors, typography, spacing, effects, ... }        │  │
│  │    themes: { light, dark, brand-a, ... }                        │  │
│  │  }                                                              │  │
│  └────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌───────────────────────────────────────────────────────────────────────┐
│                     STYLE DICTIONARY COMPILATION                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │
│  │  sd-transforms   │  │ Custom          │  │ Custom              │  │
│  │  (DTCG support)  │  │ Transforms      │  │ Formats             │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              ▼                         ▼                         ▼
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│      CSS Output      │  │  TypeScript Output   │  │  Figma JSON Output   │
│  ┌────────────────┐  │  │  ┌────────────────┐  │  │  ┌────────────────┐  │
│  │ tokens.css     │  │  │  │ tokens.ts      │  │  │  │ figma-tokens.  │  │
│  │ themes/*.css   │  │  │  │ tokens.d.ts    │  │  │  │ json           │  │
│  │ utilities.css  │  │  │  │ types.ts       │  │  │  │                │  │
│  └────────────────┘  │  │  └────────────────┘  │  │  └────────────────┘  │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘
```

---

## 1. Token Schema Design

### 1.1 YAML Source Format

We support YAML as the primary authoring format, which normalizes to DTCG JSON internally.

#### Basic Token Structure

```yaml
# tokens/colors.yaml
$schema: "https://dspec.dev/schemas/tokens/v1.json"

color:
  $description: "Color palette"
  
  primitive:
    $description: "Raw color values - do not use directly in components"
    blue:
      50: { $value: "#eff6ff", $type: color }
      100: { $value: "#dbeafe", $type: color }
      200: { $value: "#bfdbfe", $type: color }
      300: { $value: "#93c5fd", $type: color }
      400: { $value: "#60a5fa", $type: color }
      500: { $value: "#3b82f6", $type: color }
      600: { $value: "#2563eb", $type: color }
      700: { $value: "#1d4ed8", $type: color }
      800: { $value: "#1e40af", $type: color }
      900: { $value: "#1e3a8a", $type: color }
      950: { $value: "#172554", $type: color }
    
    gray:
      50: { $value: "#f9fafb", $type: color }
      # ... more shades

  semantic:
    $description: "Semantic colors - use these in components"
    interactive:
      primary:
        $value: "{color.primitive.blue.600}"
        $type: color
        $description: "Primary interactive elements"
      primary-hover:
        $value: "{color.primitive.blue.700}"
        $type: color
      primary-active:
        $value: "{color.primitive.blue.800}"
        $type: color
    
    feedback:
      success: { $value: "{color.primitive.green.600}", $type: color }
      warning: { $value: "{color.primitive.amber.500}", $type: color }
      error: { $value: "{color.primitive.red.600}", $type: color }
      info: { $value: "{color.primitive.blue.500}", $type: color }
    
    text:
      primary: { $value: "{color.primitive.gray.900}", $type: color }
      secondary: { $value: "{color.primitive.gray.600}", $type: color }
      tertiary: { $value: "{color.primitive.gray.500}", $type: color }
      inverse: { $value: "{color.primitive.gray.50}", $type: color }
      disabled: { $value: "{color.primitive.gray.400}", $type: color }
    
    surface:
      primary: { $value: "#ffffff", $type: color }
      secondary: { $value: "{color.primitive.gray.50}", $type: color }
      tertiary: { $value: "{color.primitive.gray.100}", $type: color }
      inverse: { $value: "{color.primitive.gray.900}", $type: color }
    
    border:
      default: { $value: "{color.primitive.gray.200}", $type: color }
      strong: { $value: "{color.primitive.gray.300}", $type: color }
      interactive: { $value: "{color.semantic.interactive.primary}", $type: color }
```

#### Typography Tokens

```yaml
# tokens/typography.yaml
$schema: "https://dspec.dev/schemas/tokens/v1.json"

font:
  family:
    sans:
      $value: ["Inter", "system-ui", "sans-serif"]
      $type: fontFamily
    mono:
      $value: ["JetBrains Mono", "Consolas", "monospace"]
      $type: fontFamily
  
  weight:
    regular: { $value: 400, $type: fontWeight }
    medium: { $value: 500, $type: fontWeight }
    semibold: { $value: 600, $type: fontWeight }
    bold: { $value: 700, $type: fontWeight }
  
  size:
    xs: { $value: "0.75rem", $type: dimension }    # 12px
    sm: { $value: "0.875rem", $type: dimension }   # 14px
    base: { $value: "1rem", $type: dimension }     # 16px
    lg: { $value: "1.125rem", $type: dimension }   # 18px
    xl: { $value: "1.25rem", $type: dimension }    # 20px
    2xl: { $value: "1.5rem", $type: dimension }    # 24px
    3xl: { $value: "1.875rem", $type: dimension }  # 30px
    4xl: { $value: "2.25rem", $type: dimension }   # 36px
  
  lineHeight:
    tight: { $value: 1.25, $type: number }
    normal: { $value: 1.5, $type: number }
    relaxed: { $value: 1.75, $type: number }

typography:
  $description: "Composite typography styles"
  
  heading:
    h1:
      $type: typography
      $value:
        fontFamily: "{font.family.sans}"
        fontSize: "{font.size.4xl}"
        fontWeight: "{font.weight.bold}"
        lineHeight: "{font.lineHeight.tight}"
        letterSpacing: "-0.02em"
    h2:
      $type: typography
      $value:
        fontFamily: "{font.family.sans}"
        fontSize: "{font.size.3xl}"
        fontWeight: "{font.weight.semibold}"
        lineHeight: "{font.lineHeight.tight}"
        letterSpacing: "-0.01em"
    # ... h3-h6
  
  body:
    default:
      $type: typography
      $value:
        fontFamily: "{font.family.sans}"
        fontSize: "{font.size.base}"
        fontWeight: "{font.weight.regular}"
        lineHeight: "{font.lineHeight.normal}"
    small:
      $type: typography
      $value:
        fontFamily: "{font.family.sans}"
        fontSize: "{font.size.sm}"
        fontWeight: "{font.weight.regular}"
        lineHeight: "{font.lineHeight.normal}"
  
  label:
    default:
      $type: typography
      $value:
        fontFamily: "{font.family.sans}"
        fontSize: "{font.size.sm}"
        fontWeight: "{font.weight.medium}"
        lineHeight: "{font.lineHeight.tight}"
```

#### Spacing and Sizing

```yaml
# tokens/spacing.yaml
$schema: "https://dspec.dev/schemas/tokens/v1.json"

spacing:
  $description: "Spacing scale (4px base unit)"
  0: { $value: "0", $type: dimension }
  px: { $value: "1px", $type: dimension }
  0.5: { $value: "0.125rem", $type: dimension }  # 2px
  1: { $value: "0.25rem", $type: dimension }     # 4px
  1.5: { $value: "0.375rem", $type: dimension }  # 6px
  2: { $value: "0.5rem", $type: dimension }      # 8px
  2.5: { $value: "0.625rem", $type: dimension }  # 10px
  3: { $value: "0.75rem", $type: dimension }     # 12px
  4: { $value: "1rem", $type: dimension }        # 16px
  5: { $value: "1.25rem", $type: dimension }     # 20px
  6: { $value: "1.5rem", $type: dimension }      # 24px
  8: { $value: "2rem", $type: dimension }        # 32px
  10: { $value: "2.5rem", $type: dimension }     # 40px
  12: { $value: "3rem", $type: dimension }       # 48px
  16: { $value: "4rem", $type: dimension }       # 64px
  20: { $value: "5rem", $type: dimension }       # 80px
  24: { $value: "6rem", $type: dimension }       # 96px

size:
  $description: "Component sizing"
  icon:
    xs: { $value: "{spacing.3}", $type: dimension }   # 12px
    sm: { $value: "{spacing.4}", $type: dimension }   # 16px
    md: { $value: "{spacing.5}", $type: dimension }   # 20px
    lg: { $value: "{spacing.6}", $type: dimension }   # 24px
    xl: { $value: "{spacing.8}", $type: dimension }   # 32px
  
  touch-target:
    minimum: { $value: "44px", $type: dimension }
    comfortable: { $value: "48px", $type: dimension }

radius:
  none: { $value: "0", $type: dimension }
  sm: { $value: "0.125rem", $type: dimension }   # 2px
  default: { $value: "0.25rem", $type: dimension } # 4px
  md: { $value: "0.375rem", $type: dimension }   # 6px
  lg: { $value: "0.5rem", $type: dimension }     # 8px
  xl: { $value: "0.75rem", $type: dimension }    # 12px
  2xl: { $value: "1rem", $type: dimension }      # 16px
  full: { $value: "9999px", $type: dimension }
```

#### Effects (Shadows, Transitions)

```yaml
# tokens/effects.yaml
$schema: "https://dspec.dev/schemas/tokens/v1.json"

shadow:
  $description: "Box shadows"
  none:
    $type: shadow
    $value:
      - offsetX: "0"
        offsetY: "0"
        blur: "0"
        spread: "0"
        color: "transparent"
  sm:
    $type: shadow
    $value:
      - offsetX: "0"
        offsetY: "1px"
        blur: "2px"
        spread: "0"
        color: "rgba(0, 0, 0, 0.05)"
  default:
    $type: shadow
    $value:
      - offsetX: "0"
        offsetY: "1px"
        blur: "3px"
        spread: "0"
        color: "rgba(0, 0, 0, 0.1)"
      - offsetX: "0"
        offsetY: "1px"
        blur: "2px"
        spread: "-1px"
        color: "rgba(0, 0, 0, 0.1)"
  md:
    $type: shadow
    $value:
      - offsetX: "0"
        offsetY: "4px"
        blur: "6px"
        spread: "-1px"
        color: "rgba(0, 0, 0, 0.1)"
      - offsetX: "0"
        offsetY: "2px"
        blur: "4px"
        spread: "-2px"
        color: "rgba(0, 0, 0, 0.1)"
  lg:
    $type: shadow
    $value:
      - offsetX: "0"
        offsetY: "10px"
        blur: "15px"
        spread: "-3px"
        color: "rgba(0, 0, 0, 0.1)"
      - offsetX: "0"
        offsetY: "4px"
        blur: "6px"
        spread: "-4px"
        color: "rgba(0, 0, 0, 0.1)"
  focus-ring:
    $type: shadow
    $value:
      - offsetX: "0"
        offsetY: "0"
        blur: "0"
        spread: "2px"
        color: "{color.semantic.interactive.primary}"

duration:
  instant: { $value: "0ms", $type: duration }
  fast: { $value: "100ms", $type: duration }
  normal: { $value: "200ms", $type: duration }
  slow: { $value: "300ms", $type: duration }
  slower: { $value: "500ms", $type: duration }

easing:
  linear: { $value: [0, 0, 1, 1], $type: cubicBezier }
  ease-in: { $value: [0.4, 0, 1, 1], $type: cubicBezier }
  ease-out: { $value: [0, 0, 0.2, 1], $type: cubicBezier }
  ease-in-out: { $value: [0.4, 0, 0.2, 1], $type: cubicBezier }
  spring: { $value: [0.175, 0.885, 0.32, 1.275], $type: cubicBezier }

transition:
  default:
    $type: transition
    $value:
      duration: "{duration.normal}"
      timingFunction: "{easing.ease-out}"
  fast:
    $type: transition
    $value:
      duration: "{duration.fast}"
      timingFunction: "{easing.ease-out}"
```

### 1.2 Token References

Token references use the DTCG `{path.to.token}` syntax:

```yaml
# References resolve at compile time
color:
  semantic:
    interactive:
      primary:
        $value: "{color.primitive.blue.600}"  # Resolves to #2563eb
```

#### Reference Resolution Rules

1. **Dot notation** — `{color.primitive.blue.600}` traverses the token tree
2. **Same-file and cross-file** — References work across YAML files
3. **Order-independent** — References resolve after all tokens are loaded
4. **Cycle detection** — Circular references produce a compile error
5. **Type inference** — Referenced token's type is inherited unless explicitly overridden

### 1.3 Theme Support

#### Theme Definition

```yaml
# tokens/themes/dark.yaml
$schema: "https://dspec.dev/schemas/tokens/v1.json"

$theme:
  name: dark
  extends: light  # Optional base theme

color:
  semantic:
    text:
      primary: { $value: "{color.primitive.gray.50}", $type: color }
      secondary: { $value: "{color.primitive.gray.300}", $type: color }
      tertiary: { $value: "{color.primitive.gray.400}", $type: color }
      inverse: { $value: "{color.primitive.gray.900}", $type: color }
    
    surface:
      primary: { $value: "{color.primitive.gray.900}", $type: color }
      secondary: { $value: "{color.primitive.gray.800}", $type: color }
      tertiary: { $value: "{color.primitive.gray.700}", $type: color }
      inverse: { $value: "{color.primitive.gray.50}", $type: color }
    
    border:
      default: { $value: "{color.primitive.gray.700}", $type: color }
      strong: { $value: "{color.primitive.gray.600}", $type: color }
```

#### Multi-Dimensional Theming

Support for brand + mode combinations:

```yaml
# tokens/themes/brand-a-dark.yaml
$theme:
  name: brand-a-dark
  dimensions:
    brand: brand-a
    mode: dark
  extends: [brand-a-base, dark]

color:
  primitive:
    blue:
      600: { $value: "#0066cc", $type: color }  # Brand A blue
```

### 1.4 JSON Schema Validation

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://dspec.dev/schemas/tokens/v1.json",
  "title": "dspec Token Schema",
  "type": "object",
  "properties": {
    "$schema": { "type": "string" },
    "$theme": {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "extends": { 
          "oneOf": [
            { "type": "string" },
            { "type": "array", "items": { "type": "string" } }
          ]
        },
        "dimensions": { "type": "object" }
      }
    }
  },
  "patternProperties": {
    "^(?!\\$).*$": {
      "$ref": "#/definitions/tokenOrGroup"
    }
  },
  "definitions": {
    "tokenOrGroup": {
      "oneOf": [
        { "$ref": "#/definitions/token" },
        { "$ref": "#/definitions/tokenGroup" }
      ]
    },
    "token": {
      "type": "object",
      "required": ["$value"],
      "properties": {
        "$value": {},
        "$type": { "$ref": "#/definitions/tokenType" },
        "$description": { "type": "string" },
        "$extensions": { "type": "object" }
      }
    },
    "tokenGroup": {
      "type": "object",
      "properties": {
        "$type": { "$ref": "#/definitions/tokenType" },
        "$description": { "type": "string" }
      },
      "patternProperties": {
        "^(?!\\$).*$": {
          "$ref": "#/definitions/tokenOrGroup"
        }
      }
    },
    "tokenType": {
      "enum": [
        "color",
        "dimension",
        "fontFamily",
        "fontWeight",
        "duration",
        "cubicBezier",
        "number",
        "typography",
        "shadow",
        "border",
        "gradient",
        "transition"
      ]
    }
  }
}
```

---

## 2. Style Dictionary Integration

### 2.1 Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          dspec Token Compiler                           │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        DspecTokenCompiler                        │   │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐    │   │
│  │  │ YamlParser   │──▶│ Normalizer   │──▶│ StyleDictionary  │    │   │
│  │  │              │   │ (→ DTCG)     │   │ Instance         │    │   │
│  │  └──────────────┘   └──────────────┘   └────────┬─────────┘    │   │
│  │                                                  │              │   │
│  │  ┌──────────────────────────────────────────────┼────────────┐ │   │
│  │  │              Registered Extensions           │            │ │   │
│  │  │  ┌────────────────┐  ┌────────────────┐  ┌──▼──────────┐ │ │   │
│  │  │  │ sd-transforms  │  │ Custom         │  │ Custom      │ │ │   │
│  │  │  │ (DTCG support) │  │ Transforms     │  │ Formats     │ │ │   │
│  │  │  └────────────────┘  └────────────────┘  └─────────────┘ │ │   │
│  │  └───────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Wrapper Class Implementation

```typescript
// src/compiler/token-compiler.ts

import StyleDictionary from 'style-dictionary';
import { register as registerSdTransforms } from '@tokens-studio/sd-transforms';
import { parse as parseYaml } from 'yaml';
import { Volume, createFsFromVolume } from 'memfs';
import type { TokenIR, CompileOptions, CompileResult } from '../types/token-ir';

// Register custom formats
import { registerFigmaFormat } from './formats/figma-json';
import { registerTypescriptFormat } from './formats/typescript-enhanced';
import { registerCssFormat } from './formats/css-enhanced';

export class DspecTokenCompiler {
  private sd: typeof StyleDictionary;
  private vol: InstanceType<typeof Volume>;
  
  constructor() {
    this.sd = new StyleDictionary();
    this.vol = new Volume();
    
    // Register Tokens Studio transforms for DTCG support
    registerSdTransforms(this.sd);
    
    // Register dspec custom formats
    registerFigmaFormat(this.sd);
    registerTypescriptFormat(this.sd);
    registerCssFormat(this.sd);
    
    // Register custom transforms
    this.registerCustomTransforms();
  }
  
  /**
   * Load tokens from YAML files
   */
  async loadYaml(yamlContent: string, filename: string): Promise<void> {
    // Parse YAML to JS object
    const parsed = parseYaml(yamlContent);
    
    // Normalize to DTCG format
    const dtcg = this.normalizeToStcg(parsed);
    
    // Write to in-memory filesystem
    this.vol.writeFileSync(
      `/tokens/${filename}.json`,
      JSON.stringify(dtcg, null, 2)
    );
  }
  
  /**
   * Load tokens from DTCG JSON
   */
  async loadJson(jsonContent: string, filename: string): Promise<void> {
    this.vol.writeFileSync(`/tokens/${filename}.json`, jsonContent);
  }
  
  /**
   * Compile tokens to all configured outputs
   */
  async compile(options: CompileOptions): Promise<CompileResult> {
    const config = this.buildConfig(options);
    
    const sd = new StyleDictionary(config, {
      volume: this.vol
    });
    
    // Build all platforms
    await sd.buildAllPlatforms();
    
    // Extract outputs from in-memory filesystem
    return this.extractOutputs(options);
  }
  
  /**
   * Get the parsed IR without compiling
   */
  async getIR(): Promise<TokenIR> {
    const sd = new StyleDictionary({
      source: ['/tokens/**/*.json'],
      platforms: {}
    }, { volume: this.vol });
    
    const tokens = await sd.exportPlatform('_internal');
    return this.styleDictionaryToIR(tokens);
  }
  
  private normalizeToStcg(yaml: unknown): Record<string, unknown> {
    // Convert dspec YAML conventions to DTCG JSON
    // Handles: $value, $type, $description, references
    return normalizeYamlToDtcg(yaml);
  }
  
  private buildConfig(options: CompileOptions): StyleDictionary.Config {
    const platforms: Record<string, StyleDictionary.Platform> = {};
    
    if (options.outputs.css) {
      platforms.css = {
        transformGroup: 'tokens-studio',
        transforms: ['name/kebab', 'dspec/color/css', 'dspec/shadow/css'],
        buildPath: '/build/css/',
        files: [
          {
            destination: 'tokens.css',
            format: 'dspec/css/variables',
            options: {
              outputReferences: true,
              selector: ':root'
            }
          }
        ]
      };
      
      // Add theme files if themes exist
      if (options.themes?.length) {
        for (const theme of options.themes) {
          platforms[`css-${theme}`] = {
            transformGroup: 'tokens-studio',
            transforms: ['name/kebab', 'dspec/color/css', 'dspec/shadow/css'],
            buildPath: '/build/css/themes/',
            files: [
              {
                destination: `${theme}.css`,
                format: 'dspec/css/variables',
                filter: (token) => token.$extensions?.theme === theme,
                options: {
                  outputReferences: true,
                  selector: `[data-theme="${theme}"]`
                }
              }
            ]
          };
        }
      }
    }
    
    if (options.outputs.typescript) {
      platforms.typescript = {
        transformGroup: 'tokens-studio',
        transforms: ['name/camel', 'dspec/ts/value'],
        buildPath: '/build/ts/',
        files: [
          {
            destination: 'tokens.ts',
            format: 'dspec/typescript/module'
          },
          {
            destination: 'types.ts',
            format: 'dspec/typescript/types'
          }
        ]
      };
    }
    
    if (options.outputs.figma) {
      platforms.figma = {
        transformGroup: 'tokens-studio',
        transforms: ['name/camel', 'dspec/figma/value'],
        buildPath: '/build/figma/',
        files: [
          {
            destination: 'tokens.json',
            format: 'dspec/figma/json'
          }
        ]
      };
    }
    
    return {
      source: ['/tokens/**/*.json'],
      preprocessors: ['tokens-studio'],
      platforms
    };
  }
  
  private registerCustomTransforms(): void {
    // CSS color transform (handles rgba, oklch, etc.)
    this.sd.registerTransform({
      name: 'dspec/color/css',
      type: 'value',
      transitive: true,
      filter: (token) => token.$type === 'color',
      transform: (token) => {
        const value = token.$value;
        if (typeof value === 'string') return value;
        // Handle color objects with alpha
        if (value.alpha !== undefined) {
          return `rgba(${value.r}, ${value.g}, ${value.b}, ${value.alpha})`;
        }
        return value;
      }
    });
    
    // CSS shadow transform
    this.sd.registerTransform({
      name: 'dspec/shadow/css',
      type: 'value',
      transitive: true,
      filter: (token) => token.$type === 'shadow',
      transform: (token) => {
        const shadows = Array.isArray(token.$value) ? token.$value : [token.$value];
        return shadows.map(s => 
          `${s.offsetX} ${s.offsetY} ${s.blur} ${s.spread} ${s.color}`
        ).join(', ');
      }
    });
    
    // TypeScript value transform
    this.sd.registerTransform({
      name: 'dspec/ts/value',
      type: 'value',
      transitive: true,
      transform: (token) => {
        // Keep structured values as objects for TS
        return token.$value;
      }
    });
    
    // Figma value transform
    this.sd.registerTransform({
      name: 'dspec/figma/value',
      type: 'value',
      transitive: true,
      filter: (token) => token.$type === 'color',
      transform: (token) => {
        // Convert to Figma color format (0-1 range)
        return hexToFigmaColor(token.$value);
      }
    });
  }
  
  private async extractOutputs(options: CompileOptions): Promise<CompileResult> {
    const result: CompileResult = {
      css: undefined,
      typescript: undefined,
      figma: undefined,
      meta: {
        generatedAt: new Date().toISOString(),
        version: '1.0.0',
        tokenCount: 0
      }
    };
    
    if (options.outputs.css) {
      result.css = {
        main: this.vol.readFileSync('/build/css/tokens.css', 'utf8') as string,
        themes: {}
      };
      
      for (const theme of options.themes || []) {
        const themePath = `/build/css/themes/${theme}.css`;
        if (this.vol.existsSync(themePath)) {
          result.css.themes[theme] = this.vol.readFileSync(themePath, 'utf8') as string;
        }
      }
    }
    
    if (options.outputs.typescript) {
      result.typescript = {
        tokens: this.vol.readFileSync('/build/ts/tokens.ts', 'utf8') as string,
        types: this.vol.readFileSync('/build/ts/types.ts', 'utf8') as string
      };
    }
    
    if (options.outputs.figma) {
      result.figma = this.vol.readFileSync('/build/figma/tokens.json', 'utf8') as string;
    }
    
    return result;
  }
  
  private styleDictionaryToIR(tokens: StyleDictionary.Dictionary): TokenIR {
    // Convert Style Dictionary's internal format to our TokenIR
    // See Section 4 for TokenIR type definitions
    return convertToIR(tokens);
  }
}
```

### 2.3 YAML to DTCG Normalizer

```typescript
// src/compiler/normalizer.ts

import type { DTCGTokens, DTCGToken, DTCGTokenGroup } from '../types/dtcg';

/**
 * Normalize dspec YAML format to DTCG JSON
 * 
 * Handles:
 * - Converting YAML structure to DTCG object format
 * - Preserving $value, $type, $description
 * - Converting references to DTCG format
 * - Handling $theme metadata
 */
export function normalizeYamlToDtcg(yaml: unknown): DTCGTokens {
  if (!yaml || typeof yaml !== 'object') {
    throw new Error('Invalid token input: expected object');
  }
  
  const input = yaml as Record<string, unknown>;
  const output: DTCGTokens = {};
  
  // Handle $schema (pass through)
  if (input.$schema) {
    output.$schema = input.$schema as string;
  }
  
  // Handle $theme metadata
  if (input.$theme) {
    output.$extensions = {
      ...output.$extensions,
      'com.dspec.theme': input.$theme
    };
  }
  
  // Process all non-$ keys as token groups
  for (const [key, value] of Object.entries(input)) {
    if (key.startsWith('$')) continue;
    output[key] = normalizeTokenOrGroup(value, key);
  }
  
  return output;
}

function normalizeTokenOrGroup(
  value: unknown,
  path: string
): DTCGToken | DTCGTokenGroup {
  if (!value || typeof value !== 'object') {
    throw new Error(`Invalid token at ${path}: expected object`);
  }
  
  const obj = value as Record<string, unknown>;
  
  // It's a token if it has $value
  if ('$value' in obj) {
    return normalizeToken(obj, path);
  }
  
  // Otherwise it's a group
  return normalizeGroup(obj, path);
}

function normalizeToken(
  obj: Record<string, unknown>,
  path: string
): DTCGToken {
  const token: DTCGToken = {
    $value: normalizeValue(obj.$value, obj.$type as string)
  };
  
  if (obj.$type) {
    token.$type = obj.$type as string;
  }
  
  if (obj.$description) {
    token.$description = obj.$description as string;
  }
  
  if (obj.$extensions) {
    token.$extensions = obj.$extensions as Record<string, unknown>;
  }
  
  return token;
}

function normalizeGroup(
  obj: Record<string, unknown>,
  path: string
): DTCGTokenGroup {
  const group: DTCGTokenGroup = {};
  
  // Preserve group-level $type and $description
  if (obj.$type) {
    group.$type = obj.$type as string;
  }
  
  if (obj.$description) {
    group.$description = obj.$description as string;
  }
  
  // Process child tokens/groups
  for (const [key, value] of Object.entries(obj)) {
    if (key.startsWith('$')) continue;
    group[key] = normalizeTokenOrGroup(value, `${path}.${key}`);
  }
  
  return group;
}

function normalizeValue(value: unknown, type?: string): unknown {
  // Handle references (already in correct format)
  if (typeof value === 'string' && value.startsWith('{') && value.endsWith('}')) {
    return value;
  }
  
  // Handle composite types
  if (type === 'typography' && typeof value === 'object') {
    return normalizeTypographyValue(value as Record<string, unknown>);
  }
  
  if (type === 'shadow' && Array.isArray(value)) {
    return value.map(normalizeShadowLayer);
  }
  
  if (type === 'shadow' && typeof value === 'object') {
    return [normalizeShadowLayer(value as Record<string, unknown>)];
  }
  
  return value;
}

function normalizeTypographyValue(
  value: Record<string, unknown>
): Record<string, unknown> {
  return {
    fontFamily: value.fontFamily,
    fontSize: value.fontSize,
    fontWeight: value.fontWeight,
    lineHeight: value.lineHeight,
    letterSpacing: value.letterSpacing,
    textTransform: value.textTransform
  };
}

function normalizeShadowLayer(
  layer: Record<string, unknown>
): Record<string, unknown> {
  return {
    offsetX: layer.offsetX ?? '0',
    offsetY: layer.offsetY ?? '0',
    blur: layer.blur ?? '0',
    spread: layer.spread ?? '0',
    color: layer.color ?? 'rgba(0,0,0,0)'
  };
}
```

---

## 3. Custom Formats

### 3.1 Figma JSON Format

```typescript
// src/compiler/formats/figma-json.ts

import type StyleDictionary from 'style-dictionary';

interface FigmaToken {
  value: string | number | FigmaColor | FigmaShadow[];
  type: string;
  description?: string;
}

interface FigmaColor {
  r: number;
  g: number;
  b: number;
  a: number;
}

interface FigmaShadow {
  color: FigmaColor;
  type: 'dropShadow' | 'innerShadow';
  x: number;
  y: number;
  blur: number;
  spread: number;
}

interface FigmaTokensFormat {
  [group: string]: {
    [token: string]: FigmaToken | FigmaTokensFormat;
  };
}

export function registerFigmaFormat(sd: typeof StyleDictionary): void {
  sd.registerFormat({
    name: 'dspec/figma/json',
    format: async ({ dictionary, options }) => {
      const output: FigmaTokensFormat = {};
      
      for (const token of dictionary.allTokens) {
        const path = token.path;
        const figmaToken = convertToFigmaToken(token);
        
        // Build nested structure
        setNestedValue(output, path, figmaToken);
      }
      
      return JSON.stringify(output, null, 2);
    }
  });
}

function convertToFigmaToken(token: StyleDictionary.TransformedToken): FigmaToken {
  const figmaToken: FigmaToken = {
    value: convertValue(token),
    type: mapTypeToFigma(token.$type)
  };
  
  if (token.$description) {
    figmaToken.description = token.$description;
  }
  
  return figmaToken;
}

function mapTypeToFigma(dtcgType: string): string {
  const typeMap: Record<string, string> = {
    'color': 'color',
    'dimension': 'dimension',
    'fontFamily': 'fontFamily',
    'fontWeight': 'fontWeight',
    'typography': 'typography',
    'shadow': 'boxShadow',
    'border': 'border',
    'duration': 'duration',
    'cubicBezier': 'cubicBezier',
    'number': 'number'
  };
  
  return typeMap[dtcgType] || dtcgType;
}

function convertValue(token: StyleDictionary.TransformedToken): FigmaToken['value'] {
  const type = token.$type;
  const value = token.$value;
  
  if (type === 'color') {
    return hexToFigmaColor(value as string);
  }
  
  if (type === 'shadow') {
    return convertShadowToFigma(value);
  }
  
  return value;
}

export function hexToFigmaColor(hex: string): FigmaColor {
  // Handle various color formats
  if (hex.startsWith('rgba')) {
    const match = hex.match(/rgba?\((\d+),\s*(\d+),\s*(\d+)(?:,\s*([\d.]+))?\)/);
    if (match) {
      return {
        r: parseInt(match[1]) / 255,
        g: parseInt(match[2]) / 255,
        b: parseInt(match[3]) / 255,
        a: match[4] ? parseFloat(match[4]) : 1
      };
    }
  }
  
  // Handle hex
  const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})?$/i.exec(hex);
  if (result) {
    return {
      r: parseInt(result[1], 16) / 255,
      g: parseInt(result[2], 16) / 255,
      b: parseInt(result[3], 16) / 255,
      a: result[4] ? parseInt(result[4], 16) / 255 : 1
    };
  }
  
  throw new Error(`Invalid color format: ${hex}`);
}

function convertShadowToFigma(shadows: unknown): FigmaShadow[] {
  const shadowArray = Array.isArray(shadows) ? shadows : [shadows];
  
  return shadowArray.map(shadow => ({
    color: typeof shadow.color === 'string' 
      ? hexToFigmaColor(shadow.color)
      : shadow.color,
    type: 'dropShadow' as const,
    x: parseFloat(shadow.offsetX) || 0,
    y: parseFloat(shadow.offsetY) || 0,
    blur: parseFloat(shadow.blur) || 0,
    spread: parseFloat(shadow.spread) || 0
  }));
}

function setNestedValue(
  obj: Record<string, unknown>,
  path: string[],
  value: unknown
): void {
  let current = obj;
  
  for (let i = 0; i < path.length - 1; i++) {
    const key = path[i];
    if (!(key in current)) {
      current[key] = {};
    }
    current = current[key] as Record<string, unknown>;
  }
  
  current[path[path.length - 1]] = value;
}
```

### 3.2 Enhanced TypeScript Format

```typescript
// src/compiler/formats/typescript-enhanced.ts

import type StyleDictionary from 'style-dictionary';

export function registerTypescriptFormat(sd: typeof StyleDictionary): void {
  // Main tokens module
  sd.registerFormat({
    name: 'dspec/typescript/module',
    format: async ({ dictionary, file }) => {
      const tokens = dictionary.allTokens;
      const lines: string[] = [
        '/**',
        ' * Design Tokens',
        ' * ',
        ' * Auto-generated by dspec. Do not edit manually.',
        ` * Generated: ${new Date().toISOString()}`,
        ' */',
        '',
        "import type { Tokens, ColorToken, DimensionToken, TypographyToken } from './types';",
        ''
      ];
      
      // Group tokens by top-level category
      const grouped = groupTokensByCategory(tokens);
      
      for (const [category, categoryTokens] of Object.entries(grouped)) {
        lines.push(`export const ${category} = {`);
        lines.push(...generateTokenObject(categoryTokens, 1));
        lines.push('} as const;');
        lines.push('');
      }
      
      // Export combined object
      lines.push('export const tokens: Tokens = {');
      for (const category of Object.keys(grouped)) {
        lines.push(`  ${category},`);
      }
      lines.push('};');
      lines.push('');
      lines.push('export default tokens;');
      
      return lines.join('\n');
    }
  });
  
  // Type definitions
  sd.registerFormat({
    name: 'dspec/typescript/types',
    format: async ({ dictionary }) => {
      const tokens = dictionary.allTokens;
      const lines: string[] = [
        '/**',
        ' * Design Token Type Definitions',
        ' * ',
        ' * Auto-generated by dspec. Do not edit manually.',
        ' */',
        '',
        '// Base token types',
        'export interface BaseToken<T> {',
        '  readonly value: T;',
        '  readonly type: string;',
        '  readonly description?: string;',
        '}',
        '',
        'export interface ColorToken extends BaseToken<string> {',
        '  readonly type: "color";',
        '}',
        '',
        'export interface DimensionToken extends BaseToken<string> {',
        '  readonly type: "dimension";',
        '}',
        '',
        'export interface TypographyToken extends BaseToken<{',
        '  readonly fontFamily: string | string[];',
        '  readonly fontSize: string;',
        '  readonly fontWeight: number;',
        '  readonly lineHeight: number | string;',
        '  readonly letterSpacing?: string;',
        '}> {',
        '  readonly type: "typography";',
        '}',
        '',
        'export interface ShadowToken extends BaseToken<ShadowValue[]> {',
        '  readonly type: "shadow";',
        '}',
        '',
        'export interface ShadowValue {',
        '  readonly offsetX: string;',
        '  readonly offsetY: string;',
        '  readonly blur: string;',
        '  readonly spread: string;',
        '  readonly color: string;',
        '}',
        '',
        'export interface DurationToken extends BaseToken<string> {',
        '  readonly type: "duration";',
        '}',
        '',
        'export interface EasingToken extends BaseToken<[number, number, number, number]> {',
        '  readonly type: "cubicBezier";',
        '}',
        ''
      ];
      
      // Generate specific type interfaces from actual tokens
      const grouped = groupTokensByCategory(tokens);
      
      for (const [category, categoryTokens] of Object.entries(grouped)) {
        lines.push(`export interface ${capitalize(category)}Tokens {`);
        lines.push(...generateTypeInterface(categoryTokens, 1));
        lines.push('}');
        lines.push('');
      }
      
      // Combined Tokens interface
      lines.push('export interface Tokens {');
      for (const category of Object.keys(grouped)) {
        lines.push(`  readonly ${category}: ${capitalize(category)}Tokens;`);
      }
      lines.push('}');
      
      return lines.join('\n');
    }
  });
}

function groupTokensByCategory(
  tokens: StyleDictionary.TransformedToken[]
): Record<string, StyleDictionary.TransformedToken[]> {
  const grouped: Record<string, StyleDictionary.TransformedToken[]> = {};
  
  for (const token of tokens) {
    const category = token.path[0];
    if (!grouped[category]) {
      grouped[category] = [];
    }
    grouped[category].push(token);
  }
  
  return grouped;
}

function generateTokenObject(
  tokens: StyleDictionary.TransformedToken[],
  depth: number
): string[] {
  const lines: string[] = [];
  const indent = '  '.repeat(depth);
  
  // Build tree structure
  const tree = buildTokenTree(tokens);
  
  for (const [key, value] of Object.entries(tree)) {
    if (isToken(value)) {
      const token = value as StyleDictionary.TransformedToken;
      lines.push(`${indent}${formatKey(key)}: {`);
      lines.push(`${indent}  value: ${formatValue(token.$value)},`);
      lines.push(`${indent}  type: "${token.$type}",`);
      if (token.$description) {
        lines.push(`${indent}  description: ${JSON.stringify(token.$description)},`);
      }
      lines.push(`${indent}},`);
    } else {
      lines.push(`${indent}${formatKey(key)}: {`);
      lines.push(...generateTokenObject(
        Object.values(value as Record<string, unknown>).filter(isToken) as StyleDictionary.TransformedToken[],
        depth + 1
      ));
      lines.push(`${indent}},`);
    }
  }
  
  return lines;
}

function generateTypeInterface(
  tokens: StyleDictionary.TransformedToken[],
  depth: number
): string[] {
  const lines: string[] = [];
  const indent = '  '.repeat(depth);
  
  const tree = buildTokenTree(tokens);
  
  for (const [key, value] of Object.entries(tree)) {
    if (isToken(value)) {
      const token = value as StyleDictionary.TransformedToken;
      const tokenType = getTypeScriptTokenType(token.$type);
      lines.push(`${indent}readonly ${formatKey(key)}: ${tokenType};`);
    } else {
      lines.push(`${indent}readonly ${formatKey(key)}: {`);
      // Recursively generate nested interface
      const nestedTokens = flattenTree(value as Record<string, unknown>);
      lines.push(...generateTypeInterface(nestedTokens, depth + 1));
      lines.push(`${indent}};`);
    }
  }
  
  return lines;
}

function getTypeScriptTokenType(dtcgType: string): string {
  const typeMap: Record<string, string> = {
    'color': 'ColorToken',
    'dimension': 'DimensionToken',
    'typography': 'TypographyToken',
    'shadow': 'ShadowToken',
    'duration': 'DurationToken',
    'cubicBezier': 'EasingToken',
    'fontFamily': 'BaseToken<string | string[]>',
    'fontWeight': 'BaseToken<number>',
    'number': 'BaseToken<number>'
  };
  
  return typeMap[dtcgType] || 'BaseToken<unknown>';
}

function buildTokenTree(
  tokens: StyleDictionary.TransformedToken[]
): Record<string, unknown> {
  const tree: Record<string, unknown> = {};
  
  for (const token of tokens) {
    // Skip first path segment (category, handled at higher level)
    const path = token.path.slice(1);
    setNestedValue(tree, path, token);
  }
  
  return tree;
}

function isToken(value: unknown): boolean {
  return value && typeof value === 'object' && '$value' in (value as object);
}

function flattenTree(tree: Record<string, unknown>): StyleDictionary.TransformedToken[] {
  const tokens: StyleDictionary.TransformedToken[] = [];
  
  for (const value of Object.values(tree)) {
    if (isToken(value)) {
      tokens.push(value as StyleDictionary.TransformedToken);
    } else if (typeof value === 'object' && value !== null) {
      tokens.push(...flattenTree(value as Record<string, unknown>));
    }
  }
  
  return tokens;
}

function formatKey(key: string): string {
  // Quote keys that aren't valid identifiers
  if (/^[a-zA-Z_$][a-zA-Z0-9_$]*$/.test(key)) {
    return key;
  }
  return `"${key}"`;
}

function formatValue(value: unknown): string {
  if (typeof value === 'string') {
    return JSON.stringify(value);
  }
  if (Array.isArray(value)) {
    return JSON.stringify(value);
  }
  if (typeof value === 'object' && value !== null) {
    return JSON.stringify(value);
  }
  return String(value);
}

function capitalize(str: string): string {
  return str.charAt(0).toUpperCase() + str.slice(1);
}

function setNestedValue(
  obj: Record<string, unknown>,
  path: string[],
  value: unknown
): void {
  let current = obj;
  
  for (let i = 0; i < path.length - 1; i++) {
    const key = path[i];
    if (!(key in current)) {
      current[key] = {};
    }
    current = current[key] as Record<string, unknown>;
  }
  
  current[path[path.length - 1]] = value;
}
```

### 3.3 Enhanced CSS Format

```typescript
// src/compiler/formats/css-enhanced.ts

import type StyleDictionary from 'style-dictionary';

export function registerCssFormat(sd: typeof StyleDictionary): void {
  sd.registerFormat({
    name: 'dspec/css/variables',
    format: async ({ dictionary, options, file }) => {
      const selector = options?.selector || ':root';
      const outputReferences = options?.outputReferences ?? true;
      
      const lines: string[] = [
        '/**',
        ' * Design Tokens - CSS Custom Properties',
        ' * ',
        ' * Auto-generated by dspec. Do not edit manually.',
        ` * Generated: ${new Date().toISOString()}`,
        ' */',
        '',
        `${selector} {`
      ];
      
      // Group by category for better organization
      const grouped = groupByCategory(dictionary.allTokens);
      
      for (const [category, tokens] of Object.entries(grouped)) {
        lines.push(`  /* ${category} */`);
        
        for (const token of tokens) {
          const name = `--${token.name}`;
          let value: string;
          
          if (outputReferences && hasReference(token)) {
            // Output as reference to another variable
            value = getReferencedVariableName(token);
          } else {
            value = formatCssValue(token);
          }
          
          // Add description as comment
          if (token.$description) {
            lines.push(`  /* ${token.$description} */`);
          }
          
          lines.push(`  ${name}: ${value};`);
        }
        
        lines.push('');
      }
      
      lines.push('}');
      
      // Add utility classes if enabled
      if (options?.utilities) {
        lines.push('');
        lines.push(...generateUtilityClasses(dictionary.allTokens));
      }
      
      return lines.join('\n');
    }
  });
}

function groupByCategory(
  tokens: StyleDictionary.TransformedToken[]
): Record<string, StyleDictionary.TransformedToken[]> {
  const grouped: Record<string, StyleDictionary.TransformedToken[]> = {};
  
  for (const token of tokens) {
    const category = token.path[0];
    if (!grouped[category]) {
      grouped[category] = [];
    }
    grouped[category].push(token);
  }
  
  return grouped;
}

function hasReference(token: StyleDictionary.TransformedToken): boolean {
  return token.original?.$value?.toString().includes('{');
}

function getReferencedVariableName(token: StyleDictionary.TransformedToken): string {
  // Convert reference path to CSS variable
  const refMatch = token.original.$value.match(/\{([^}]+)\}/);
  if (refMatch) {
    const refPath = refMatch[1].replace(/\./g, '-');
    return `var(--${refPath})`;
  }
  return formatCssValue(token);
}

function formatCssValue(token: StyleDictionary.TransformedToken): string {
  const type = token.$type;
  const value = token.$value;
  
  if (type === 'typography') {
    // Typography is a composite - we don't output as single variable
    // Instead, components reference individual properties
    return '/* composite - see individual properties */';
  }
  
  if (type === 'shadow') {
    return formatShadowValue(value);
  }
  
  if (type === 'cubicBezier') {
    const [x1, y1, x2, y2] = value as number[];
    return `cubic-bezier(${x1}, ${y1}, ${x2}, ${y2})`;
  }
  
  return String(value);
}

function formatShadowValue(shadows: unknown): string {
  const shadowArray = Array.isArray(shadows) ? shadows : [shadows];
  return shadowArray
    .map(s => `${s.offsetX} ${s.offsetY} ${s.blur} ${s.spread} ${s.color}`)
    .join(', ');
}

function generateUtilityClasses(
  tokens: StyleDictionary.TransformedToken[]
): string[] {
  const lines: string[] = ['/* Utility Classes */'];
  
  // Color utilities
  const colorTokens = tokens.filter(t => t.$type === 'color' && t.path[1] === 'semantic');
  for (const token of colorTokens) {
    const className = token.path.slice(2).join('-');
    lines.push(`.text-${className} { color: var(--${token.name}); }`);
    lines.push(`.bg-${className} { background-color: var(--${token.name}); }`);
    lines.push(`.border-${className} { border-color: var(--${token.name}); }`);
  }
  
  // Spacing utilities
  const spacingTokens = tokens.filter(t => t.path[0] === 'spacing');
  for (const token of spacingTokens) {
    const key = token.path[token.path.length - 1];
    lines.push(`.p-${key} { padding: var(--${token.name}); }`);
    lines.push(`.m-${key} { margin: var(--${token.name}); }`);
    lines.push(`.gap-${key} { gap: var(--${token.name}); }`);
  }
  
  return lines;
}
```

---

## 4. Intermediate Representation (IR) Types

### 4.1 Core IR Types

```typescript
// src/types/token-ir.ts

/**
 * Token IR - Internal Representation
 * 
 * This is the canonical type-safe representation of parsed tokens.
 * All compilation outputs derive from this structure.
 */

// ============================================================================
// Base Types
// ============================================================================

export interface TokenMeta {
  /** Unique identifier for the token */
  id: string;
  /** Dot-separated path (e.g., "color.semantic.interactive.primary") */
  path: string;
  /** Human-readable name */
  name: string;
  /** Token description */
  description?: string;
  /** Source file this token came from */
  source?: string;
  /** Extensions/metadata */
  extensions?: Record<string, unknown>;
}

export interface ResolvedReference {
  /** Original reference string (e.g., "{color.primitive.blue.600}") */
  original: string;
  /** Resolved token path */
  resolvedPath: string;
  /** Resolved value */
  resolvedValue: unknown;
}

// ============================================================================
// Token Value Types
// ============================================================================

export interface ColorValue {
  /** Hex color (e.g., "#2563eb") */
  hex: string;
  /** RGBA components (0-255 for rgb, 0-1 for alpha) */
  rgba: { r: number; g: number; b: number; a: number };
  /** Original value as specified */
  original: string;
}

export interface DimensionValue {
  /** Numeric value */
  value: number;
  /** Unit (px, rem, em, %, etc.) */
  unit: 'px' | 'rem' | 'em' | '%' | '';
  /** Original string value */
  original: string;
}

export interface FontFamilyValue {
  /** Font stack as array */
  stack: string[];
  /** Primary font (first in stack) */
  primary: string;
}

export interface TypographyValue {
  fontFamily: FontFamilyValue;
  fontSize: DimensionValue;
  fontWeight: number;
  lineHeight: number | DimensionValue;
  letterSpacing?: DimensionValue;
  textTransform?: 'none' | 'uppercase' | 'lowercase' | 'capitalize';
}

export interface ShadowLayer {
  offsetX: DimensionValue;
  offsetY: DimensionValue;
  blur: DimensionValue;
  spread: DimensionValue;
  color: ColorValue;
  inset?: boolean;
}

export interface ShadowValue {
  layers: ShadowLayer[];
}

export interface DurationValue {
  /** Duration in milliseconds */
  ms: number;
  /** Original string value */
  original: string;
}

export interface EasingValue {
  /** Cubic bezier control points */
  bezier: [number, number, number, number];
  /** Named preset if applicable */
  preset?: 'linear' | 'ease' | 'ease-in' | 'ease-out' | 'ease-in-out';
}

export interface TransitionValue {
  duration: DurationValue;
  timingFunction: EasingValue;
  delay?: DurationValue;
  property?: string;
}

export interface BorderValue {
  width: DimensionValue;
  style: 'solid' | 'dashed' | 'dotted' | 'none';
  color: ColorValue;
}

// ============================================================================
// Token Types
// ============================================================================

export interface BaseToken<T> {
  meta: TokenMeta;
  type: string;
  value: T;
  /** If this token references another */
  reference?: ResolvedReference;
}

export interface ColorToken extends BaseToken<ColorValue> {
  type: 'color';
}

export interface DimensionToken extends BaseToken<DimensionValue> {
  type: 'dimension';
}

export interface FontFamilyToken extends BaseToken<FontFamilyValue> {
  type: 'fontFamily';
}

export interface FontWeightToken extends BaseToken<number> {
  type: 'fontWeight';
}

export interface TypographyToken extends BaseToken<TypographyValue> {
  type: 'typography';
}

export interface ShadowToken extends BaseToken<ShadowValue> {
  type: 'shadow';
}

export interface DurationToken extends BaseToken<DurationValue> {
  type: 'duration';
}

export interface EasingToken extends BaseToken<EasingValue> {
  type: 'cubicBezier';
}

export interface TransitionToken extends BaseToken<TransitionValue> {
  type: 'transition';
}

export interface BorderToken extends BaseToken<BorderValue> {
  type: 'border';
}

export interface NumberToken extends BaseToken<number> {
  type: 'number';
}

export type Token =
  | ColorToken
  | DimensionToken
  | FontFamilyToken
  | FontWeightToken
  | TypographyToken
  | ShadowToken
  | DurationToken
  | EasingToken
  | TransitionToken
  | BorderToken
  | NumberToken;

// ============================================================================
// Token Collections
// ============================================================================

export interface ColorTokens {
  primitive: Record<string, Record<string, ColorToken>>;
  semantic: {
    interactive: Record<string, ColorToken>;
    feedback: Record<string, ColorToken>;
    text: Record<string, ColorToken>;
    surface: Record<string, ColorToken>;
    border: Record<string, ColorToken>;
  };
}

export interface TypographyTokens {
  font: {
    family: Record<string, FontFamilyToken>;
    weight: Record<string, FontWeightToken>;
    size: Record<string, DimensionToken>;
    lineHeight: Record<string, NumberToken>;
  };
  styles: {
    heading: Record<string, TypographyToken>;
    body: Record<string, TypographyToken>;
    label: Record<string, TypographyToken>;
  };
}

export interface SpacingTokens {
  scale: Record<string, DimensionToken>;
  size: {
    icon: Record<string, DimensionToken>;
    touchTarget: Record<string, DimensionToken>;
  };
  radius: Record<string, DimensionToken>;
}

export interface EffectTokens {
  shadow: Record<string, ShadowToken>;
  duration: Record<string, DurationToken>;
  easing: Record<string, EasingToken>;
  transition: Record<string, TransitionToken>;
}

// ============================================================================
// Theme Types
// ============================================================================

export interface ThemeMeta {
  /** Theme name (e.g., "dark", "brand-a") */
  name: string;
  /** Base theme this extends */
  extends?: string | string[];
  /** Theme dimensions for multi-dimensional theming */
  dimensions?: Record<string, string>;
  /** Theme description */
  description?: string;
}

export interface Theme {
  meta: ThemeMeta;
  /** Token overrides for this theme */
  overrides: Partial<TokenIR['tokens']>;
}

// ============================================================================
// Main IR Structure
// ============================================================================

export interface TokenIR {
  /** IR schema version */
  version: '1.0.0';
  
  /** Generation metadata */
  meta: {
    generatedAt: string;
    source: string[];
    checksum: string;
  };
  
  /** All resolved tokens organized by category */
  tokens: {
    color: ColorTokens;
    typography: TypographyTokens;
    spacing: SpacingTokens;
    effects: EffectTokens;
  };
  
  /** Theme definitions */
  themes: Record<string, Theme>;
  
  /** Flat lookup table for quick access by path */
  byPath: Map<string, Token>;
  
  /** Flat lookup table for quick access by ID */
  byId: Map<string, Token>;
}

// ============================================================================
// Compiler Types
// ============================================================================

export interface CompileOptions {
  /** Token source files/directories */
  source: string[];
  
  /** Output formats to generate */
  outputs: {
    css?: boolean | CssOutputOptions;
    typescript?: boolean | TypeScriptOutputOptions;
    figma?: boolean;
    swift?: boolean | SwiftOutputOptions;
  };
  
  /** Themes to compile */
  themes?: string[];
  
  /** Output directory */
  outDir: string;
  
  /** Watch mode */
  watch?: boolean;
}

export interface CssOutputOptions {
  /** CSS selector for variables (default: ":root") */
  selector?: string;
  /** Output references as var() (default: true) */
  outputReferences?: boolean;
  /** Generate utility classes */
  utilities?: boolean;
  /** File name (default: "tokens.css") */
  filename?: string;
}

export interface TypeScriptOutputOptions {
  /** Generate as ESM or CommonJS */
  module?: 'esm' | 'cjs';
  /** Generate declaration files */
  declarations?: boolean;
  /** File name (default: "tokens.ts") */
  filename?: string;
}

export interface SwiftOutputOptions {
  /** Swift package name */
  packageName?: string;
  /** Generate as enum or struct */
  style?: 'enum' | 'struct';
}

export interface CompileResult {
  css?: {
    main: string;
    themes: Record<string, string>;
  };
  typescript?: {
    tokens: string;
    types: string;
  };
  figma?: string;
  swift?: string;
  meta: {
    generatedAt: string;
    version: string;
    tokenCount: number;
  };
}

// ============================================================================
// Validation Types
// ============================================================================

export interface ValidationError {
  code: string;
  message: string;
  path: string;
  source?: string;
  line?: number;
  column?: number;
}

export interface ValidationWarning {
  code: string;
  message: string;
  path: string;
  source?: string;
}

export interface ParseResult {
  /** Parsed token IR (undefined if errors) */
  ir?: TokenIR;
  /** Validation errors (blocks compilation) */
  errors: ValidationError[];
  /** Validation warnings (informational) */
  warnings: ValidationWarning[];
  /** Parse successful */
  success: boolean;
}
```

### 4.2 IR Conversion Utilities

```typescript
// src/types/ir-utils.ts

import type { Token, TokenIR, ColorToken, DimensionToken, ColorValue, DimensionValue } from './token-ir';

/**
 * Parse a hex color string to ColorValue
 */
export function parseColor(value: string): ColorValue {
  const hex = value.startsWith('#') ? value : `#${value}`;
  
  // Parse hex to rgba
  const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})?$/i.exec(hex);
  
  if (!result) {
    throw new Error(`Invalid hex color: ${value}`);
  }
  
  return {
    hex,
    rgba: {
      r: parseInt(result[1], 16),
      g: parseInt(result[2], 16),
      b: parseInt(result[3], 16),
      a: result[4] ? parseInt(result[4], 16) / 255 : 1
    },
    original: value
  };
}

/**
 * Parse a dimension string (e.g., "16px", "1rem") to DimensionValue
 */
export function parseDimension(value: string): DimensionValue {
  const match = value.match(/^(-?[\d.]+)(px|rem|em|%)?$/);
  
  if (!match) {
    throw new Error(`Invalid dimension: ${value}`);
  }
  
  return {
    value: parseFloat(match[1]),
    unit: (match[2] || '') as DimensionValue['unit'],
    original: value
  };
}

/**
 * Get a token by path from the IR
 */
export function getTokenByPath(ir: TokenIR, path: string): Token | undefined {
  return ir.byPath.get(path);
}

/**
 * Get all tokens of a specific type
 */
export function getTokensByType<T extends Token>(
  ir: TokenIR,
  type: T['type']
): T[] {
  const tokens: T[] = [];
  
  for (const token of ir.byPath.values()) {
    if (token.type === type) {
      tokens.push(token as T);
    }
  }
  
  return tokens;
}

/**
 * Resolve a reference path to its target token
 */
export function resolveReference(ir: TokenIR, refPath: string): Token | undefined {
  // Remove curly braces if present
  const cleanPath = refPath.replace(/^\{|\}$/g, '');
  return ir.byPath.get(cleanPath);
}

/**
 * Convert ColorValue to CSS string
 */
export function colorToCss(color: ColorValue): string {
  if (color.rgba.a === 1) {
    return color.hex;
  }
  
  const { r, g, b, a } = color.rgba;
  return `rgba(${r}, ${g}, ${b}, ${a})`;
}

/**
 * Convert DimensionValue to CSS string
 */
export function dimensionToCss(dim: DimensionValue): string {
  if (dim.unit === '') {
    return String(dim.value);
  }
  return `${dim.value}${dim.unit}`;
}

/**
 * Build lookup maps for the IR
 */
export function buildLookupMaps(
  tokens: Token[]
): { byPath: Map<string, Token>; byId: Map<string, Token> } {
  const byPath = new Map<string, Token>();
  const byId = new Map<string, Token>();
  
  for (const token of tokens) {
    byPath.set(token.meta.path, token);
    byId.set(token.meta.id, token);
  }
  
  return { byPath, byId };
}
```

---

## 5. Integration with Component Generation

### 5.1 Token Consumption in Components

Components consume tokens through generated TypeScript and CSS:

```typescript
// Generated: src/components/Button/Button.tsx

import { cva, type VariantProps } from 'class-variance-authority';
import { tokens } from '@org/design-tokens';

/**
 * Button component
 * 
 * Generated by dspec
 * Spec: components/button.yaml@abc123
 */

const buttonVariants = cva(
  // Base styles using CSS custom properties
  [
    'inline-flex items-center justify-center',
    'rounded-[var(--radius-md)]',
    'font-[var(--font-weight-medium)]',
    'text-[var(--font-size-sm)]',
    'transition-[var(--transition-default)]',
    'focus:outline-none',
    'focus:ring-2',
    'focus:ring-[var(--color-semantic-interactive-primary)]',
    'focus:ring-offset-2',
    'disabled:opacity-50',
    'disabled:pointer-events-none',
  ],
  {
    variants: {
      variant: {
        primary: [
          'bg-[var(--color-semantic-interactive-primary)]',
          'text-[var(--color-semantic-text-inverse)]',
          'hover:bg-[var(--color-semantic-interactive-primary-hover)]',
          'active:bg-[var(--color-semantic-interactive-primary-active)]',
        ],
        secondary: [
          'bg-[var(--color-semantic-surface-secondary)]',
          'text-[var(--color-semantic-text-primary)]',
          'border',
          'border-[var(--color-semantic-border-default)]',
          'hover:bg-[var(--color-semantic-surface-tertiary)]',
        ],
        tertiary: [
          'bg-transparent',
          'text-[var(--color-semantic-interactive-primary)]',
          'hover:bg-[var(--color-semantic-surface-secondary)]',
        ],
        destructive: [
          'bg-[var(--color-semantic-feedback-error)]',
          'text-[var(--color-semantic-text-inverse)]',
          'hover:bg-[var(--color-semantic-feedback-error-hover)]',
        ],
      },
      size: {
        sm: [
          'h-8',
          'px-[var(--spacing-3)]',
          'text-[var(--font-size-xs)]',
        ],
        md: [
          'h-10',
          'px-[var(--spacing-4)]',
          'text-[var(--font-size-sm)]',
        ],
        lg: [
          'h-12',
          'px-[var(--spacing-6)]',
          'text-[var(--font-size-base)]',
        ],
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
  /** Loading state */
  loading?: boolean;
  /** Left icon */
  leftIcon?: React.ReactNode;
  /** Right icon */
  rightIcon?: React.ReactNode;
}

export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, loading, leftIcon, rightIcon, children, disabled, ...props }, ref) => {
    return (
      <button
        ref={ref}
        className={buttonVariants({ variant, size, className })}
        disabled={disabled || loading}
        {...props}
      >
        {loading && <Spinner className="mr-2" />}
        {!loading && leftIcon && <span className="mr-2">{leftIcon}</span>}
        {children}
        {rightIcon && <span className="ml-2">{rightIcon}</span>}
      </button>
    );
  }
);

Button.displayName = 'Button';
```

### 5.2 Component Spec to Token Mapping

```yaml
# specs/components/button.yaml

name: Button
description: Primary interaction element

# Token dependencies - explicitly declare which tokens this component uses
tokens:
  colors:
    - color.semantic.interactive.primary
    - color.semantic.interactive.primary-hover
    - color.semantic.interactive.primary-active
    - color.semantic.text.primary
    - color.semantic.text.inverse
    - color.semantic.surface.secondary
    - color.semantic.surface.tertiary
    - color.semantic.border.default
    - color.semantic.feedback.error
  spacing:
    - spacing.3
    - spacing.4
    - spacing.6
  typography:
    - font.size.xs
    - font.size.sm
    - font.size.base
    - font.weight.medium
  effects:
    - radius.md
    - transition.default
    - shadow.focus-ring

props:
  variant:
    type: enum
    values: [primary, secondary, tertiary, destructive]
    default: primary
    description: Visual style variant
  
  size:
    type: enum
    values: [sm, md, lg]
    default: md
    description: Button size

styling:
  base:
    display: inline-flex
    alignItems: center
    justifyContent: center
    borderRadius: $radius.md
    fontWeight: $font.weight.medium
    transition: $transition.default
    
  variants:
    primary:
      backgroundColor: $color.semantic.interactive.primary
      color: $color.semantic.text.inverse
      _hover:
        backgroundColor: $color.semantic.interactive.primary-hover
      _active:
        backgroundColor: $color.semantic.interactive.primary-active
    
    secondary:
      backgroundColor: $color.semantic.surface.secondary
      color: $color.semantic.text.primary
      borderWidth: 1px
      borderColor: $color.semantic.border.default
      _hover:
        backgroundColor: $color.semantic.surface.tertiary
    
    # ... more variants

  sizes:
    sm:
      height: 32px
      paddingX: $spacing.3
      fontSize: $font.size.xs
    md:
      height: 40px
      paddingX: $spacing.4
      fontSize: $font.size.sm
    lg:
      height: 48px
      paddingX: $spacing.6
      fontSize: $font.size.base

accessibility:
  role: button
  focusRing:
    color: $color.semantic.interactive.primary
    width: 2px
    offset: 2px
```

### 5.3 Token Dependency Graph

```typescript
// src/compiler/dependency-graph.ts

import type { TokenIR, Token } from '../types/token-ir';

export interface DependencyNode {
  token: Token;
  dependsOn: Set<string>;  // Token paths this depends on
  usedBy: Set<string>;     // Token paths that depend on this
}

export interface DependencyGraph {
  nodes: Map<string, DependencyNode>;
  
  /** Get all tokens that would be affected by changing a token */
  getAffected(tokenPath: string): Token[];
  
  /** Get all tokens a component depends on */
  getComponentDependencies(componentTokens: string[]): Token[];
  
  /** Detect circular references */
  detectCycles(): string[][] | null;
  
  /** Get topologically sorted tokens (for compilation order) */
  topologicalSort(): Token[];
}

export function buildDependencyGraph(ir: TokenIR): DependencyGraph {
  const nodes = new Map<string, DependencyNode>();
  
  // Build nodes
  for (const token of ir.byPath.values()) {
    nodes.set(token.meta.path, {
      token,
      dependsOn: new Set(),
      usedBy: new Set()
    });
  }
  
  // Build edges from references
  for (const token of ir.byPath.values()) {
    if (token.reference) {
      const node = nodes.get(token.meta.path)!;
      const refPath = token.reference.resolvedPath;
      
      node.dependsOn.add(refPath);
      
      const refNode = nodes.get(refPath);
      if (refNode) {
        refNode.usedBy.add(token.meta.path);
      }
    }
  }
  
  return {
    nodes,
    
    getAffected(tokenPath: string): Token[] {
      const affected: Token[] = [];
      const visited = new Set<string>();
      
      const visit = (path: string) => {
        if (visited.has(path)) return;
        visited.add(path);
        
        const node = nodes.get(path);
        if (node) {
          affected.push(node.token);
          for (const usedByPath of node.usedBy) {
            visit(usedByPath);
          }
        }
      };
      
      visit(tokenPath);
      return affected;
    },
    
    getComponentDependencies(componentTokens: string[]): Token[] {
      const deps: Token[] = [];
      const visited = new Set<string>();
      
      const visit = (path: string) => {
        if (visited.has(path)) return;
        visited.add(path);
        
        const node = nodes.get(path);
        if (node) {
          deps.push(node.token);
          for (const depPath of node.dependsOn) {
            visit(depPath);
          }
        }
      };
      
      for (const tokenPath of componentTokens) {
        visit(tokenPath);
      }
      
      return deps;
    },
    
    detectCycles(): string[][] | null {
      const cycles: string[][] = [];
      const visited = new Set<string>();
      const stack = new Set<string>();
      
      const dfs = (path: string, currentPath: string[]): boolean => {
        if (stack.has(path)) {
          // Found cycle
          const cycleStart = currentPath.indexOf(path);
          cycles.push(currentPath.slice(cycleStart));
          return true;
        }
        
        if (visited.has(path)) return false;
        
        visited.add(path);
        stack.add(path);
        
        const node = nodes.get(path);
        if (node) {
          for (const depPath of node.dependsOn) {
            dfs(depPath, [...currentPath, path]);
          }
        }
        
        stack.delete(path);
        return false;
      };
      
      for (const path of nodes.keys()) {
        dfs(path, []);
      }
      
      return cycles.length > 0 ? cycles : null;
    },
    
    topologicalSort(): Token[] {
      const sorted: Token[] = [];
      const visited = new Set<string>();
      
      const visit = (path: string) => {
        if (visited.has(path)) return;
        visited.add(path);
        
        const node = nodes.get(path);
        if (node) {
          for (const depPath of node.dependsOn) {
            visit(depPath);
          }
          sorted.push(node.token);
        }
      };
      
      for (const path of nodes.keys()) {
        visit(path);
      }
      
      return sorted;
    }
  };
}
```

---

## 6. Pipeline Configuration

### 6.1 dspec Configuration File

```yaml
# dspec.config.yaml

$schema: "https://dspec.dev/schemas/config/v1.json"

# Token sources
tokens:
  source:
    - tokens/**/*.yaml
    - tokens/**/*.json
  
  # Output configuration
  output:
    dir: build/tokens
    
    css:
      enabled: true
      filename: tokens.css
      selector: ":root"
      outputReferences: true
      utilities: true
    
    typescript:
      enabled: true
      filename: tokens.ts
      declarations: true
      module: esm
    
    figma:
      enabled: true
      filename: figma-tokens.json
  
  # Theme configuration
  themes:
    - name: light
      default: true
    - name: dark
      selector: '[data-theme="dark"]'
    - name: brand-a
      selector: '[data-brand="a"]'

# Component sources
components:
  source:
    - specs/components/**/*.yaml
  
  output:
    dir: src/components
    
    react:
      enabled: true
      framework: react
      styling: cva  # or tailwind, css-modules
    
    stories:
      enabled: true
      framework: storybook
    
    tests:
      enabled: true
      framework: vitest

# Publishing
publish:
  tokens:
    package: "@org/design-tokens"
    registry: https://npm.pkg.github.com
  
  components:
    package: "@org/design-system"
    registry: https://npm.pkg.github.com

# Watch mode
watch:
  debounce: 300
  clearScreen: true
```

### 6.2 CLI Commands

```bash
# Compile tokens only
dspec tokens build

# Compile tokens with watch mode
dspec tokens build --watch

# Generate CSS only
dspec tokens build --css-only

# Compile specific theme
dspec tokens build --theme dark

# Validate tokens without building
dspec tokens validate

# Generate visual preview
dspec tokens preview

# Show token dependency graph
dspec tokens deps --token color.semantic.interactive.primary

# Export IR for debugging
dspec tokens ir --output tokens-ir.json
```

---

## 7. Error Handling

### 7.1 Error Types

```typescript
// src/errors/token-errors.ts

export class TokenError extends Error {
  constructor(
    message: string,
    public code: string,
    public path?: string,
    public source?: string,
    public line?: number,
    public column?: number
  ) {
    super(message);
    this.name = 'TokenError';
  }
  
  toJSON() {
    return {
      code: this.code,
      message: this.message,
      path: this.path,
      source: this.source,
      line: this.line,
      column: this.column
    };
  }
}

export class TokenValidationError extends TokenError {
  constructor(message: string, path: string, source?: string) {
    super(message, 'VALIDATION_ERROR', path, source);
    this.name = 'TokenValidationError';
  }
}

export class TokenReferenceError extends TokenError {
  constructor(
    reference: string,
    path: string,
    source?: string
  ) {
    super(
      `Unresolved reference: ${reference}`,
      'UNRESOLVED_REFERENCE',
      path,
      source
    );
    this.name = 'TokenReferenceError';
  }
}

export class TokenCircularReferenceError extends TokenError {
  constructor(cycle: string[]) {
    super(
      `Circular reference detected: ${cycle.join(' -> ')}`,
      'CIRCULAR_REFERENCE'
    );
    this.name = 'TokenCircularReferenceError';
  }
}

export class TokenTypeError extends TokenError {
  constructor(
    expectedType: string,
    actualType: string,
    path: string,
    source?: string
  ) {
    super(
      `Type mismatch: expected ${expectedType}, got ${actualType}`,
      'TYPE_MISMATCH',
      path,
      source
    );
    this.name = 'TokenTypeError';
  }
}
```

### 7.2 Validation Pipeline

```typescript
// src/compiler/validator.ts

import type { TokenIR, ValidationError, ValidationWarning, ParseResult } from '../types/token-ir';
import { buildDependencyGraph } from './dependency-graph';

export interface ValidationRule {
  id: string;
  description: string;
  severity: 'error' | 'warning';
  validate(ir: TokenIR): (ValidationError | ValidationWarning)[];
}

const builtInRules: ValidationRule[] = [
  {
    id: 'no-circular-refs',
    description: 'Detect circular token references',
    severity: 'error',
    validate(ir) {
      const graph = buildDependencyGraph(ir);
      const cycles = graph.detectCycles();
      
      if (cycles) {
        return cycles.map(cycle => ({
          code: 'CIRCULAR_REFERENCE',
          message: `Circular reference: ${cycle.join(' -> ')}`,
          path: cycle[0]
        }));
      }
      
      return [];
    }
  },
  {
    id: 'no-unresolved-refs',
    description: 'Ensure all references resolve',
    severity: 'error',
    validate(ir) {
      const errors: ValidationError[] = [];
      
      for (const token of ir.byPath.values()) {
        if (token.reference && !token.reference.resolvedValue) {
          errors.push({
            code: 'UNRESOLVED_REFERENCE',
            message: `Unresolved reference: ${token.reference.original}`,
            path: token.meta.path,
            source: token.meta.source
          });
        }
      }
      
      return errors;
    }
  },
  {
    id: 'consistent-type-refs',
    description: 'References should match expected types',
    severity: 'error',
    validate(ir) {
      const errors: ValidationError[] = [];
      
      for (const token of ir.byPath.values()) {
        if (token.reference) {
          const refToken = ir.byPath.get(token.reference.resolvedPath);
          if (refToken && refToken.type !== token.type) {
            errors.push({
              code: 'TYPE_MISMATCH',
              message: `Reference type mismatch: ${token.meta.path} (${token.type}) references ${refToken.meta.path} (${refToken.type})`,
              path: token.meta.path
            });
          }
        }
      }
      
      return errors;
    }
  },
  {
    id: 'no-unused-primitives',
    description: 'Primitive tokens should be referenced by semantic tokens',
    severity: 'warning',
    validate(ir) {
      const warnings: ValidationWarning[] = [];
      const graph = buildDependencyGraph(ir);
      
      for (const [path, node] of graph.nodes) {
        if (path.includes('.primitive.') && node.usedBy.size === 0) {
          warnings.push({
            code: 'UNUSED_PRIMITIVE',
            message: `Primitive token is never referenced: ${path}`,
            path
          });
        }
      }
      
      return warnings;
    }
  },
  {
    id: 'color-contrast',
    description: 'Check color contrast ratios for accessibility',
    severity: 'warning',
    validate(ir) {
      // TODO: Implement WCAG contrast checking
      return [];
    }
  }
];

export function validateTokens(
  ir: TokenIR,
  rules: ValidationRule[] = builtInRules
): { errors: ValidationError[]; warnings: ValidationWarning[] } {
  const errors: ValidationError[] = [];
  const warnings: ValidationWarning[] = [];
  
  for (const rule of rules) {
    const results = rule.validate(ir);
    
    for (const result of results) {
      if (rule.severity === 'error') {
        errors.push(result as ValidationError);
      } else {
        warnings.push(result as ValidationWarning);
      }
    }
  }
  
  return { errors, warnings };
}
```

---

## Appendix: Example Outputs

### A. Generated CSS

```css
/**
 * Design Tokens - CSS Custom Properties
 * 
 * Auto-generated by dspec. Do not edit manually.
 * Generated: 2026-01-15T10:30:00Z
 */

:root {
  /* color.primitive */
  --color-primitive-blue-50: #eff6ff;
  --color-primitive-blue-100: #dbeafe;
  --color-primitive-blue-500: #3b82f6;
  --color-primitive-blue-600: #2563eb;
  --color-primitive-blue-700: #1d4ed8;
  --color-primitive-blue-800: #1e40af;
  
  /* color.semantic */
  --color-semantic-interactive-primary: var(--color-primitive-blue-600);
  --color-semantic-interactive-primary-hover: var(--color-primitive-blue-700);
  --color-semantic-interactive-primary-active: var(--color-primitive-blue-800);
  --color-semantic-text-primary: var(--color-primitive-gray-900);
  --color-semantic-text-secondary: var(--color-primitive-gray-600);
  --color-semantic-text-inverse: var(--color-primitive-gray-50);
  --color-semantic-surface-primary: #ffffff;
  --color-semantic-surface-secondary: var(--color-primitive-gray-50);
  --color-semantic-border-default: var(--color-primitive-gray-200);
  
  /* spacing */
  --spacing-1: 0.25rem;
  --spacing-2: 0.5rem;
  --spacing-3: 0.75rem;
  --spacing-4: 1rem;
  --spacing-6: 1.5rem;
  --spacing-8: 2rem;
  
  /* typography */
  --font-family-sans: "Inter", system-ui, sans-serif;
  --font-size-xs: 0.75rem;
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-weight-regular: 400;
  --font-weight-medium: 500;
  --font-weight-semibold: 600;
  
  /* effects */
  --radius-md: 0.375rem;
  --shadow-default: 0 1px 3px 0 rgba(0, 0, 0, 0.1), 0 1px 2px -1px rgba(0, 0, 0, 0.1);
  --duration-normal: 200ms;
  --easing-ease-out: cubic-bezier(0, 0, 0.2, 1);
  --transition-default: 200ms cubic-bezier(0, 0, 0.2, 1);
}
```

### B. Generated TypeScript

```typescript
/**
 * Design Tokens
 * 
 * Auto-generated by dspec. Do not edit manually.
 * Generated: 2026-01-15T10:30:00Z
 */

import type { Tokens, ColorToken, DimensionToken } from './types';

export const color = {
  primitive: {
    blue: {
      50: { value: "#eff6ff", type: "color" },
      100: { value: "#dbeafe", type: "color" },
      500: { value: "#3b82f6", type: "color" },
      600: { value: "#2563eb", type: "color" },
      700: { value: "#1d4ed8", type: "color" },
      800: { value: "#1e40af", type: "color" },
    },
  },
  semantic: {
    interactive: {
      primary: { value: "#2563eb", type: "color", description: "Primary interactive elements" },
      "primary-hover": { value: "#1d4ed8", type: "color" },
      "primary-active": { value: "#1e40af", type: "color" },
    },
    text: {
      primary: { value: "#111827", type: "color" },
      secondary: { value: "#4b5563", type: "color" },
      inverse: { value: "#f9fafb", type: "color" },
    },
  },
} as const;

export const spacing = {
  1: { value: "0.25rem", type: "dimension" },
  2: { value: "0.5rem", type: "dimension" },
  3: { value: "0.75rem", type: "dimension" },
  4: { value: "1rem", type: "dimension" },
  6: { value: "1.5rem", type: "dimension" },
  8: { value: "2rem", type: "dimension" },
} as const;

export const tokens: Tokens = {
  color,
  spacing,
  // ... other categories
};

export default tokens;
```

### C. Generated Figma JSON

```json
{
  "color": {
    "primitive": {
      "blue": {
        "50": {
          "value": { "r": 0.937, "g": 0.965, "b": 1, "a": 1 },
          "type": "color"
        },
        "600": {
          "value": { "r": 0.145, "g": 0.388, "b": 0.922, "a": 1 },
          "type": "color"
        }
      }
    },
    "semantic": {
      "interactive": {
        "primary": {
          "value": { "r": 0.145, "g": 0.388, "b": 0.922, "a": 1 },
          "type": "color",
          "description": "Primary interactive elements"
        }
      }
    }
  },
  "spacing": {
    "4": {
      "value": "16",
      "type": "dimension"
    }
  }
}
```

---

## Summary

This architecture provides:

1. **YAML-first authoring** with DTCG normalization
2. **Style Dictionary as the core compiler** with custom extensions
3. **Type-safe IR** for all intermediate processing
4. **Custom formats** for Figma JSON and enhanced TypeScript
5. **Dependency tracking** for efficient regeneration
6. **Robust validation** with clear error messages
7. **Seamless component integration** via CSS custom properties and TypeScript types

The design prioritizes extensibility, allowing new output formats to be added without modifying the core pipeline.
