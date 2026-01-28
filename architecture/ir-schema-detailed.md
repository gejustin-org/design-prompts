# dspec IR & Schema Architecture

> Detailed specification for the Intermediate Representation (IR) and schema validation system.

**Version:** 1.0.0  
**Status:** Draft  
**Last Updated:** 2025-01-20

---

## Table of Contents

1. [Overview](#1-overview)
2. [Component Schema Design](#2-component-schema-design)
3. [Token Schema Design](#3-token-schema-design)
4. [Complete IR TypeScript Types](#4-complete-ir-typescript-types)
5. [JSON Schema Definitions](#5-json-schema-definitions)
6. [Reference Resolution](#6-reference-resolution)
7. [Schema Versioning & Migration](#7-schema-versioning--migration)
8. [Validation Pipeline](#8-validation-pipeline)

---

## 1. Overview

### 1.1 Architecture Principles

1. **Single Source of Truth**: Zod schemas define both TypeScript types and JSON Schema for IDE support
2. **Layered Validation**: Parse → Schema Validate → Reference Resolve → Semantic Validate
3. **Recoverable Errors**: Continue parsing/validating to report all errors, not just the first
4. **Version-Aware**: All specs carry schema version, migrations are explicit

### 1.2 Technology Stack

| Purpose | Tool | Rationale |
|---------|------|-----------|
| YAML Parsing | `yaml` (eemeli) | Comment preservation, full spec compliance |
| Schema Definition | Zod | TypeScript inference, JSON Schema export |
| Runtime Validation | Ajv | Performance, comprehensive error paths |
| IDE Autocomplete | YAML Language Server | JSON Schema → autocomplete |

### 1.3 Data Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           SPEC FILES                                     │
│  tokens.yaml    components/*.yaml    patterns/*.yaml    theme.yaml       │
└───────────┬─────────────────────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                         1. YAML PARSING                                   │
│   yaml.parse() → Raw JavaScript objects with source locations             │
└───────────┬───────────────────────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                      2. SCHEMA VALIDATION                                 │
│   Zod.safeParse() → Typed objects or structured errors                    │
└───────────┬───────────────────────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                     3. REFERENCE RESOLUTION                               │
│   Resolve $token refs, component refs → Linked IR graph                   │
└───────────┬───────────────────────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                    4. SEMANTIC VALIDATION                                 │
│   Cross-component checks, circular deps, consistency                      │
└───────────┬───────────────────────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                        VALIDATED IR                                       │
│   TokenIR + ComponentIR[] + PatternIR[] → Ready for generation            │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Component Schema Design

### 2.1 Basic Component Structure

```yaml
# components/button.yaml
$schema: "https://dspec.dev/schemas/component/v1.json"
schemaVersion: 1

name: Button
description: "Primary interactive element for user actions"
category: action

# Component metadata
meta:
  figmaUrl: "https://figma.com/file/xxx"
  status: stable  # draft | experimental | stable | deprecated
  since: "1.0.0"
  deprecatedIn: null

# Props define the component's public API
props:
  - name: variant
    type: enum
    values: [primary, secondary, tertiary, destructive]
    default: primary
    description: "Visual style variant"
    
  - name: size
    type: enum
    values: [sm, md, lg]
    default: md
    description: "Button size"
    
  - name: disabled
    type: boolean
    default: false
    description: "Whether the button is disabled"
    
  - name: loading
    type: boolean
    default: false
    description: "Show loading spinner"
    
  - name: icon
    type: slot
    position: start | end
    description: "Optional icon element"
    
  - name: children
    type: slot
    required: true
    description: "Button content"

# Styling maps props to tokens
styling:
  base:
    display: inline-flex
    alignItems: center
    justifyContent: center
    fontFamily: $typography.fontFamily.sans
    fontWeight: $typography.fontWeight.medium
    borderRadius: $radii.md
    cursor: pointer
    transition: "all 150ms ease"
    
  variants:
    variant:
      primary:
        backgroundColor: $colors.interactive.primary
        color: $colors.text.onPrimary
        border: none
      secondary:
        backgroundColor: transparent
        color: $colors.interactive.primary
        border: "1px solid $colors.border.default"
      tertiary:
        backgroundColor: transparent
        color: $colors.interactive.primary
        border: none
      destructive:
        backgroundColor: $colors.semantic.error
        color: $colors.text.onPrimary
        border: none
        
    size:
      sm:
        height: $spacing.8
        paddingInline: $spacing.3
        fontSize: $typography.fontSize.sm
        gap: $spacing.1
      md:
        height: $spacing.10
        paddingInline: $spacing.4
        fontSize: $typography.fontSize.base
        gap: $spacing.2
      lg:
        height: $spacing.12
        paddingInline: $spacing.6
        fontSize: $typography.fontSize.lg
        gap: $spacing.2

# States define interactive behaviors
states:
  hover:
    condition: ":hover:not(:disabled)"
    styling:
      variants:
        variant:
          primary:
            backgroundColor: $colors.interactive.primaryHover
          secondary:
            backgroundColor: $colors.surface.hover
          tertiary:
            backgroundColor: $colors.surface.hover
          destructive:
            backgroundColor: $colors.semantic.errorHover
            
  focus:
    condition: ":focus-visible"
    styling:
      base:
        outline: "2px solid $colors.border.focus"
        outlineOffset: "2px"
        
  active:
    condition: ":active:not(:disabled)"
    styling:
      base:
        transform: "scale(0.98)"
        
  disabled:
    condition: ":disabled, [aria-disabled='true']"
    styling:
      base:
        opacity: 0.5
        cursor: not-allowed
        pointerEvents: none

# Accessibility requirements
accessibility:
  role: button
  focusable: true
  ariaProps:
    - name: aria-disabled
      bindTo: disabled
    - name: aria-busy
      bindTo: loading
  keyboardNav:
    - key: Enter
      action: activate
    - key: Space
      action: activate
```

### 2.2 Prop Types

```yaml
# All supported prop types

props:
  # Primitive types
  - name: label
    type: string
    required: true
    minLength: 1
    maxLength: 100
    pattern: "^[A-Z].*"  # Regex validation
    
  - name: count
    type: number
    min: 0
    max: 100
    step: 1  # Integer only
    
  - name: isOpen
    type: boolean
    default: false
    
  # Enum type
  - name: variant
    type: enum
    values: [primary, secondary]
    default: primary
    
  # Union type
  - name: size
    type: union
    members:
      - type: enum
        values: [sm, md, lg]
      - type: number
        min: 0
        
  # Object type (complex props)
  - name: config
    type: object
    properties:
      - name: animate
        type: boolean
        default: true
      - name: duration
        type: number
        default: 300
        
  # Array type
  - name: items
    type: array
    items:
      type: string
    minItems: 1
    maxItems: 10
    
  # Slot type (children/render props)
  - name: children
    type: slot
    required: true
    
  - name: renderHeader
    type: slot
    description: "Custom header renderer"
    
  # Ref type
  - name: inputRef
    type: ref
    element: HTMLInputElement
    
  # Function/callback type
  - name: onChange
    type: callback
    parameters:
      - name: value
        type: string
      - name: event
        type: Event
    returns: void
    
  # Component reference type
  - name: as
    type: componentRef
    accepts: [button, a, div]
    default: button
```

### 2.3 Compound Components

```yaml
# components/select.yaml
$schema: "https://dspec.dev/schemas/component/v1.json"
schemaVersion: 1

name: Select
description: "Dropdown selection component"
category: input

# Compound component definition
compound: true

# Root component props
props:
  - name: value
    type: string
    description: "Currently selected value"
    
  - name: onChange
    type: callback
    parameters:
      - name: value
        type: string
    
  - name: placeholder
    type: string
    default: "Select an option"
    
  - name: disabled
    type: boolean
    default: false

# Child component definitions
parts:
  - name: Trigger
    description: "The button that opens the dropdown"
    props:
      - name: children
        type: slot
        required: true
    styling:
      base:
        display: flex
        alignItems: center
        justifyContent: space-between
        padding: $spacing.3
        border: "1px solid $colors.border.default"
        borderRadius: $radii.md
        backgroundColor: $colors.surface.default
        cursor: pointer
        minWidth: "200px"
    states:
      hover:
        condition: ":hover:not(:disabled)"
        styling:
          base:
            borderColor: $colors.border.hover
      focus:
        condition: ":focus-visible"
        styling:
          base:
            outline: "2px solid $colors.border.focus"
            outlineOffset: "2px"
            
  - name: Content
    description: "The dropdown panel containing options"
    props:
      - name: children
        type: slot
        required: true
      - name: align
        type: enum
        values: [start, center, end]
        default: start
      - name: side
        type: enum
        values: [top, bottom]
        default: bottom
    styling:
      base:
        position: absolute
        zIndex: $zIndex.dropdown
        backgroundColor: $colors.surface.elevated
        border: "1px solid $colors.border.default"
        borderRadius: $radii.md
        boxShadow: $shadows.md
        padding: $spacing.1
        minWidth: "200px"
        
  - name: Item
    description: "An individual selectable option"
    props:
      - name: value
        type: string
        required: true
      - name: disabled
        type: boolean
        default: false
      - name: children
        type: slot
        required: true
    styling:
      base:
        display: flex
        alignItems: center
        padding: "$spacing.2 $spacing.3"
        borderRadius: $radii.sm
        cursor: pointer
    states:
      hover:
        condition: ":hover:not([data-disabled])"
        styling:
          base:
            backgroundColor: $colors.surface.hover
      selected:
        condition: "[data-selected]"
        styling:
          base:
            backgroundColor: $colors.interactive.primarySubtle
            color: $colors.interactive.primary
      disabled:
        condition: "[data-disabled]"
        styling:
          base:
            opacity: 0.5
            cursor: not-allowed
            
  - name: Separator
    description: "Visual separator between groups"
    styling:
      base:
        height: "1px"
        backgroundColor: $colors.border.default
        margin: "$spacing.1 0"
        
  - name: Group
    description: "Groups related items with a label"
    props:
      - name: label
        type: string
        required: true
      - name: children
        type: slot
        required: true
    styling:
      base:
        padding: "$spacing.1 0"
      label:
        padding: "$spacing.1 $spacing.3"
        fontSize: $typography.fontSize.xs
        fontWeight: $typography.fontWeight.semibold
        color: $colors.text.muted
        textTransform: uppercase
        letterSpacing: "0.05em"

# Context shared between compound parts
context:
  - name: selectedValue
    type: string
    providedBy: root
  - name: onSelect
    type: callback
    providedBy: root
  - name: isOpen
    type: boolean
    providedBy: root

# Accessibility for compound component
accessibility:
  role: listbox
  parts:
    Trigger:
      role: combobox
      ariaProps:
        - name: aria-expanded
          bindTo: isOpen
        - name: aria-haspopup
          value: listbox
    Content:
      role: listbox
    Item:
      role: option
      ariaProps:
        - name: aria-selected
          bindTo: isSelected
```

### 2.4 Component Variants (Alternative Approach)

```yaml
# components/badge.yaml
$schema: "https://dspec.dev/schemas/component/v1.json"
schemaVersion: 1

name: Badge
description: "Small status indicator"
category: feedback

props:
  - name: variant
    type: enum
    values: [default, success, warning, error, info]
    default: default
    
  - name: size
    type: enum
    values: [sm, md]
    default: md
    
  - name: dot
    type: boolean
    default: false
    description: "Show as dot indicator only"
    
  - name: children
    type: slot

styling:
  base:
    display: inline-flex
    alignItems: center
    justifyContent: center
    fontWeight: $typography.fontWeight.medium
    borderRadius: $radii.full
    whiteSpace: nowrap
    
  variants:
    variant:
      default:
        backgroundColor: $colors.surface.muted
        color: $colors.text.default
      success:
        backgroundColor: $colors.semantic.successSubtle
        color: $colors.semantic.success
      warning:
        backgroundColor: $colors.semantic.warningSubtle
        color: $colors.semantic.warning
      error:
        backgroundColor: $colors.semantic.errorSubtle
        color: $colors.semantic.error
      info:
        backgroundColor: $colors.semantic.infoSubtle
        color: $colors.semantic.info
        
    size:
      sm:
        height: $spacing.5
        paddingInline: $spacing.2
        fontSize: $typography.fontSize.xs
      md:
        height: $spacing.6
        paddingInline: $spacing.2\.5
        fontSize: $typography.fontSize.sm

  # Compound variants (when multiple props combine)
  compoundVariants:
    - when:
        dot: true
      styling:
        paddingInline: 0
        
    - when:
        dot: true
        size: sm
      styling:
        width: $spacing.2
        height: $spacing.2
        
    - when:
        dot: true
        size: md
      styling:
        width: $spacing.2\.5
        height: $spacing.2\.5
```

---

## 3. Token Schema Design

### 3.1 Token File Structure

```yaml
# tokens/index.yaml
$schema: "https://dspec.dev/schemas/tokens/v1.json"
schemaVersion: 1

meta:
  name: "Acme Design Tokens"
  version: "2.0.0"
  description: "Core design tokens for Acme products"

# Import other token files
imports:
  - ./colors.yaml
  - ./typography.yaml
  - ./spacing.yaml
  - ./effects.yaml

# Theme configuration
themes:
  default: light
  available:
    - name: light
      file: ./themes/light.yaml
    - name: dark
      file: ./themes/dark.yaml
```

### 3.2 Color Tokens

```yaml
# tokens/colors.yaml
$schema: "https://dspec.dev/schemas/tokens/v1.json"
schemaVersion: 1

colors:
  # Primitive colors (raw values)
  primitives:
    gray:
      50: "#fafafa"
      100: "#f4f4f5"
      200: "#e4e4e7"
      300: "#d4d4d8"
      400: "#a1a1aa"
      500: "#71717a"
      600: "#52525b"
      700: "#3f3f46"
      800: "#27272a"
      900: "#18181b"
      950: "#09090b"
      
    blue:
      50: "#eff6ff"
      100: "#dbeafe"
      200: "#bfdbfe"
      300: "#93c5fd"
      400: "#60a5fa"
      500: "#3b82f6"
      600: "#2563eb"
      700: "#1d4ed8"
      800: "#1e40af"
      900: "#1e3a8a"
      950: "#172554"
      
    # ... other color scales
    
  # Semantic colors (reference primitives)
  semantic:
    error: $colors.primitives.red.500
    errorHover: $colors.primitives.red.600
    errorSubtle: $colors.primitives.red.50
    
    warning: $colors.primitives.amber.500
    warningHover: $colors.primitives.amber.600
    warningSubtle: $colors.primitives.amber.50
    
    success: $colors.primitives.green.500
    successHover: $colors.primitives.green.600
    successSubtle: $colors.primitives.green.50
    
    info: $colors.primitives.blue.500
    infoHover: $colors.primitives.blue.600
    infoSubtle: $colors.primitives.blue.50
    
  # Contextual colors (can be themed)
  text:
    default: $colors.primitives.gray.900
    muted: $colors.primitives.gray.500
    subtle: $colors.primitives.gray.400
    onPrimary: "#ffffff"
    onDark: "#ffffff"
    
  surface:
    default: "#ffffff"
    muted: $colors.primitives.gray.100
    subtle: $colors.primitives.gray.50
    elevated: "#ffffff"
    hover: $colors.primitives.gray.100
    
  border:
    default: $colors.primitives.gray.200
    muted: $colors.primitives.gray.100
    hover: $colors.primitives.gray.300
    focus: $colors.primitives.blue.500
    
  interactive:
    primary: $colors.primitives.blue.600
    primaryHover: $colors.primitives.blue.700
    primarySubtle: $colors.primitives.blue.50
```

### 3.3 Typography Tokens

```yaml
# tokens/typography.yaml
$schema: "https://dspec.dev/schemas/tokens/v1.json"
schemaVersion: 1

typography:
  fontFamily:
    sans: "'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif"
    mono: "'JetBrains Mono', 'Fira Code', monospace"
    
  fontSize:
    xs: "0.75rem"      # 12px
    sm: "0.875rem"     # 14px
    base: "1rem"       # 16px
    lg: "1.125rem"     # 18px
    xl: "1.25rem"      # 20px
    2xl: "1.5rem"      # 24px
    3xl: "1.875rem"    # 30px
    4xl: "2.25rem"     # 36px
    5xl: "3rem"        # 48px
    
  fontWeight:
    normal: 400
    medium: 500
    semibold: 600
    bold: 700
    
  lineHeight:
    none: 1
    tight: 1.25
    snug: 1.375
    normal: 1.5
    relaxed: 1.625
    loose: 2
    
  letterSpacing:
    tighter: "-0.05em"
    tight: "-0.025em"
    normal: "0"
    wide: "0.025em"
    wider: "0.05em"
    widest: "0.1em"
    
  # Composite text styles
  textStyles:
    h1:
      fontFamily: $typography.fontFamily.sans
      fontSize: $typography.fontSize.4xl
      fontWeight: $typography.fontWeight.bold
      lineHeight: $typography.lineHeight.tight
      letterSpacing: $typography.letterSpacing.tight
      
    h2:
      fontFamily: $typography.fontFamily.sans
      fontSize: $typography.fontSize.3xl
      fontWeight: $typography.fontWeight.semibold
      lineHeight: $typography.lineHeight.tight
      
    body:
      fontFamily: $typography.fontFamily.sans
      fontSize: $typography.fontSize.base
      fontWeight: $typography.fontWeight.normal
      lineHeight: $typography.lineHeight.normal
      
    caption:
      fontFamily: $typography.fontFamily.sans
      fontSize: $typography.fontSize.sm
      fontWeight: $typography.fontWeight.normal
      lineHeight: $typography.lineHeight.normal
      color: $colors.text.muted
```

### 3.4 Spacing & Layout Tokens

```yaml
# tokens/spacing.yaml
$schema: "https://dspec.dev/schemas/tokens/v1.json"
schemaVersion: 1

spacing:
  # Scale based on 4px base unit
  0: "0"
  px: "1px"
  0.5: "0.125rem"   # 2px
  1: "0.25rem"      # 4px
  1.5: "0.375rem"   # 6px
  2: "0.5rem"       # 8px
  2.5: "0.625rem"   # 10px
  3: "0.75rem"      # 12px
  3.5: "0.875rem"   # 14px
  4: "1rem"         # 16px
  5: "1.25rem"      # 20px
  6: "1.5rem"       # 24px
  7: "1.75rem"      # 28px
  8: "2rem"         # 32px
  9: "2.25rem"      # 36px
  10: "2.5rem"      # 40px
  11: "2.75rem"     # 44px
  12: "3rem"        # 48px
  14: "3.5rem"      # 56px
  16: "4rem"        # 64px
  20: "5rem"        # 80px
  24: "6rem"        # 96px
  28: "7rem"        # 112px
  32: "8rem"        # 128px

radii:
  none: "0"
  sm: "0.125rem"    # 2px
  md: "0.375rem"    # 6px
  lg: "0.5rem"      # 8px
  xl: "0.75rem"     # 12px
  2xl: "1rem"       # 16px
  3xl: "1.5rem"     # 24px
  full: "9999px"

zIndex:
  base: 0
  dropdown: 1000
  sticky: 1100
  fixed: 1200
  modalBackdrop: 1300
  modal: 1400
  popover: 1500
  tooltip: 1600

breakpoints:
  sm: "640px"
  md: "768px"
  lg: "1024px"
  xl: "1280px"
  2xl: "1536px"
```

### 3.5 Effects Tokens

```yaml
# tokens/effects.yaml
$schema: "https://dspec.dev/schemas/tokens/v1.json"
schemaVersion: 1

shadows:
  none: "none"
  sm: "0 1px 2px 0 rgb(0 0 0 / 0.05)"
  md: "0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1)"
  lg: "0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1)"
  xl: "0 20px 25px -5px rgb(0 0 0 / 0.1), 0 8px 10px -6px rgb(0 0 0 / 0.1)"
  2xl: "0 25px 50px -12px rgb(0 0 0 / 0.25)"
  inner: "inset 0 2px 4px 0 rgb(0 0 0 / 0.05)"

transitions:
  durations:
    instant: "0ms"
    fast: "100ms"
    normal: "150ms"
    slow: "300ms"
    slower: "500ms"
    
  easings:
    linear: "linear"
    easeIn: "cubic-bezier(0.4, 0, 1, 1)"
    easeOut: "cubic-bezier(0, 0, 0.2, 1)"
    easeInOut: "cubic-bezier(0.4, 0, 0.2, 1)"
    bounce: "cubic-bezier(0.68, -0.55, 0.265, 1.55)"

animations:
  spin:
    keyframes:
      from:
        transform: "rotate(0deg)"
      to:
        transform: "rotate(360deg)"
    duration: "1s"
    timing: linear
    iteration: infinite
    
  fadeIn:
    keyframes:
      from:
        opacity: 0
      to:
        opacity: 1
    duration: $transitions.durations.normal
    timing: $transitions.easings.easeOut
    
  slideInUp:
    keyframes:
      from:
        opacity: 0
        transform: "translateY(10px)"
      to:
        opacity: 1
        transform: "translateY(0)"
    duration: $transitions.durations.normal
    timing: $transitions.easings.easeOut
```

---

## 4. Complete IR TypeScript Types

### 4.1 Core Types

```typescript
// src/ir/types/core.ts

/**
 * Schema version for migration support
 */
export interface SchemaVersion {
  major: number;
  minor: number;
  patch: number;
}

/**
 * Source location for error reporting
 */
export interface SourceLocation {
  file: string;
  line: number;
  column: number;
  endLine?: number;
  endColumn?: number;
}

/**
 * Base for all IR nodes
 */
export interface IRNode {
  /** Unique identifier within the IR */
  id: string;
  /** Source location for error reporting */
  source?: SourceLocation;
  /** Original raw value before transformation */
  raw?: unknown;
}

/**
 * Validation severity levels
 */
export type Severity = 'error' | 'warning' | 'info';

/**
 * Validation issue with context
 */
export interface ValidationIssue {
  severity: Severity;
  code: string;
  message: string;
  path: string[];
  source?: SourceLocation;
  suggestions?: string[];
}

/**
 * Result of parsing/validation
 */
export interface ParseResult<T> {
  success: boolean;
  data?: T;
  errors: ValidationIssue[];
  warnings: ValidationIssue[];
}
```

### 4.2 Token IR Types

```typescript
// src/ir/types/tokens.ts

import { IRNode, SourceLocation } from './core';

/**
 * Token reference (e.g., "$colors.primary")
 */
export interface TokenRef {
  type: 'ref';
  path: string[];
  raw: string;
  source?: SourceLocation;
}

/**
 * Resolved token value or reference
 */
export type TokenValue = string | number | TokenRef;

/**
 * Base token interface
 */
export interface BaseToken extends IRNode {
  name: string;
  description?: string;
  deprecated?: boolean;
  deprecatedMessage?: string;
}

/**
 * Color token (hex, rgb, hsl, or reference)
 */
export interface ColorToken extends BaseToken {
  type: 'color';
  value: string | TokenRef;
  /** Computed hex value after resolution */
  resolved?: string;
}

/**
 * Dimension token (spacing, sizing)
 */
export interface DimensionToken extends BaseToken {
  type: 'dimension';
  value: string | number | TokenRef;
  unit?: 'px' | 'rem' | 'em' | '%';
  /** Computed value with unit */
  resolved?: string;
}

/**
 * Font family token
 */
export interface FontFamilyToken extends BaseToken {
  type: 'fontFamily';
  value: string | string[] | TokenRef;
  resolved?: string;
}

/**
 * Font size token
 */
export interface FontSizeToken extends BaseToken {
  type: 'fontSize';
  value: string | number | TokenRef;
  resolved?: string;
}

/**
 * Font weight token
 */
export interface FontWeightToken extends BaseToken {
  type: 'fontWeight';
  value: number | string | TokenRef;
  resolved?: number;
}

/**
 * Line height token
 */
export interface LineHeightToken extends BaseToken {
  type: 'lineHeight';
  value: number | string | TokenRef;
  resolved?: string | number;
}

/**
 * Letter spacing token
 */
export interface LetterSpacingToken extends BaseToken {
  type: 'letterSpacing';
  value: string | TokenRef;
  resolved?: string;
}

/**
 * Border radius token
 */
export interface RadiusToken extends BaseToken {
  type: 'radius';
  value: string | number | TokenRef;
  resolved?: string;
}

/**
 * Shadow token
 */
export interface ShadowToken extends BaseToken {
  type: 'shadow';
  value: string | ShadowValue | TokenRef;
  resolved?: string;
}

export interface ShadowValue {
  x: string | number;
  y: string | number;
  blur: string | number;
  spread: string | number;
  color: string | TokenRef;
  inset?: boolean;
}

/**
 * Z-index token
 */
export interface ZIndexToken extends BaseToken {
  type: 'zIndex';
  value: number | TokenRef;
  resolved?: number;
}

/**
 * Duration token (for transitions/animations)
 */
export interface DurationToken extends BaseToken {
  type: 'duration';
  value: string | number | TokenRef;
  resolved?: string;
}

/**
 * Easing/timing function token
 */
export interface EasingToken extends BaseToken {
  type: 'easing';
  value: string | TokenRef;
  resolved?: string;
}

/**
 * Composite text style token
 */
export interface TextStyleToken extends BaseToken {
  type: 'textStyle';
  fontFamily?: string | TokenRef;
  fontSize?: string | TokenRef;
  fontWeight?: number | TokenRef;
  lineHeight?: string | number | TokenRef;
  letterSpacing?: string | TokenRef;
  color?: string | TokenRef;
  resolved?: {
    fontFamily?: string;
    fontSize?: string;
    fontWeight?: number;
    lineHeight?: string | number;
    letterSpacing?: string;
    color?: string;
  };
}

/**
 * Animation keyframe
 */
export interface AnimationKeyframe {
  offset: number | string; // 0, 0.5, 1 or "from", "to"
  properties: Record<string, string | TokenRef>;
}

/**
 * Animation token
 */
export interface AnimationToken extends BaseToken {
  type: 'animation';
  keyframes: AnimationKeyframe[];
  duration?: string | TokenRef;
  timing?: string | TokenRef;
  delay?: string | TokenRef;
  iteration?: number | 'infinite';
  direction?: 'normal' | 'reverse' | 'alternate' | 'alternate-reverse';
  fillMode?: 'none' | 'forwards' | 'backwards' | 'both';
}

/**
 * Union of all token types
 */
export type Token =
  | ColorToken
  | DimensionToken
  | FontFamilyToken
  | FontSizeToken
  | FontWeightToken
  | LineHeightToken
  | LetterSpacingToken
  | RadiusToken
  | ShadowToken
  | ZIndexToken
  | DurationToken
  | EasingToken
  | TextStyleToken
  | AnimationToken;

/**
 * Token category groupings
 */
export interface TokenCategories {
  colors: {
    primitives: Record<string, Record<string, ColorToken>>;
    semantic: Record<string, ColorToken>;
    text: Record<string, ColorToken>;
    surface: Record<string, ColorToken>;
    border: Record<string, ColorToken>;
    interactive: Record<string, ColorToken>;
  };
  typography: {
    fontFamily: Record<string, FontFamilyToken>;
    fontSize: Record<string, FontSizeToken>;
    fontWeight: Record<string, FontWeightToken>;
    lineHeight: Record<string, LineHeightToken>;
    letterSpacing: Record<string, LetterSpacingToken>;
    textStyles: Record<string, TextStyleToken>;
  };
  spacing: Record<string, DimensionToken>;
  radii: Record<string, RadiusToken>;
  shadows: Record<string, ShadowToken>;
  zIndex: Record<string, ZIndexToken>;
  transitions: {
    durations: Record<string, DurationToken>;
    easings: Record<string, EasingToken>;
  };
  animations: Record<string, AnimationToken>;
  breakpoints: Record<string, DimensionToken>;
}

/**
 * Theme override for tokens
 */
export interface ThemeOverride {
  name: string;
  description?: string;
  tokens: Partial<TokenCategories>;
}

/**
 * Complete Token IR
 */
export interface TokenIR extends IRNode {
  schemaVersion: number;
  meta: {
    name: string;
    version: string;
    description?: string;
  };
  tokens: TokenCategories;
  themes: ThemeOverride[];
}
```

### 4.3 Component IR Types

```typescript
// src/ir/types/components.ts

import { IRNode, SourceLocation, TokenRef } from './core';

/**
 * Component status lifecycle
 */
export type ComponentStatus = 'draft' | 'experimental' | 'stable' | 'deprecated';

/**
 * Component category
 */
export type ComponentCategory = 
  | 'action'      // Button, Link, IconButton
  | 'input'       // Input, Select, Checkbox, Radio
  | 'display'     // Card, Badge, Avatar
  | 'feedback'    // Alert, Toast, Progress
  | 'navigation'  // Tabs, Breadcrumb, Menu
  | 'layout'      // Stack, Grid, Container
  | 'overlay'     // Modal, Drawer, Popover
  | 'utility';    // VisuallyHidden, Portal

/**
 * Component metadata
 */
export interface ComponentMeta {
  figmaUrl?: string;
  status: ComponentStatus;
  since: string;
  deprecatedIn?: string;
  deprecatedMessage?: string;
  tags?: string[];
}

/**
 * Primitive prop types
 */
export type PrimitivePropType = 'string' | 'number' | 'boolean';

/**
 * Enum prop definition
 */
export interface EnumPropType {
  type: 'enum';
  values: string[];
}

/**
 * Union prop definition
 */
export interface UnionPropType {
  type: 'union';
  members: PropType[];
}

/**
 * Object prop definition
 */
export interface ObjectPropType {
  type: 'object';
  properties: PropDefinition[];
}

/**
 * Array prop definition
 */
export interface ArrayPropType {
  type: 'array';
  items: PropType;
  minItems?: number;
  maxItems?: number;
}

/**
 * Slot (children/render prop) definition
 */
export interface SlotPropType {
  type: 'slot';
  position?: 'start' | 'end';
}

/**
 * Ref prop definition
 */
export interface RefPropType {
  type: 'ref';
  element: string; // e.g., "HTMLInputElement"
}

/**
 * Callback/function prop definition
 */
export interface CallbackPropType {
  type: 'callback';
  parameters: Array<{
    name: string;
    type: PropType;
  }>;
  returns?: PropType | 'void';
}

/**
 * Component reference prop (polymorphic "as" prop)
 */
export interface ComponentRefPropType {
  type: 'componentRef';
  accepts: string[];
}

/**
 * All prop types
 */
export type PropType =
  | PrimitivePropType
  | EnumPropType
  | UnionPropType
  | ObjectPropType
  | ArrayPropType
  | SlotPropType
  | RefPropType
  | CallbackPropType
  | ComponentRefPropType;

/**
 * Prop validation rules
 */
export interface PropValidation {
  // String validations
  minLength?: number;
  maxLength?: number;
  pattern?: string;
  
  // Number validations
  min?: number;
  max?: number;
  step?: number;
}

/**
 * Single prop definition
 */
export interface PropDefinition extends IRNode {
  name: string;
  type: PropType;
  description?: string;
  required?: boolean;
  default?: unknown;
  validation?: PropValidation;
  deprecated?: boolean;
  deprecatedMessage?: string;
}

/**
 * CSS property value (literal or token ref)
 */
export type StyleValue = string | number | TokenRef;

/**
 * Style properties map
 */
export type StyleProperties = Record<string, StyleValue>;

/**
 * Variant styling (maps variant value to styles)
 */
export type VariantStyles = Record<string, StyleProperties>;

/**
 * Compound variant condition
 */
export interface CompoundVariant {
  when: Record<string, unknown>;
  styling: StyleProperties;
}

/**
 * Component styling definition
 */
export interface ComponentStyling {
  base: StyleProperties;
  variants?: Record<string, VariantStyles>;
  compoundVariants?: CompoundVariant[];
}

/**
 * Interactive state definition
 */
export interface StateDefinition extends IRNode {
  name: string;
  condition: string; // CSS selector or pseudo-class
  styling: ComponentStyling;
}

/**
 * ARIA prop binding
 */
export interface AriaBinding {
  name: string;
  bindTo?: string;  // Prop name to bind to
  value?: string;   // Static value
}

/**
 * Keyboard navigation handler
 */
export interface KeyboardHandler {
  key: string;
  modifiers?: Array<'ctrl' | 'alt' | 'shift' | 'meta'>;
  action: string;
  preventDefault?: boolean;
}

/**
 * Accessibility configuration
 */
export interface AccessibilityConfig {
  role?: string;
  focusable?: boolean;
  ariaProps?: AriaBinding[];
  keyboardNav?: KeyboardHandler[];
}

/**
 * Compound component part (child component)
 */
export interface ComponentPart extends IRNode {
  name: string;
  description?: string;
  props: PropDefinition[];
  styling: ComponentStyling;
  states?: StateDefinition[];
  accessibility?: AccessibilityConfig;
}

/**
 * Shared context between compound parts
 */
export interface ComponentContext {
  name: string;
  type: PropType;
  providedBy: 'root' | string;
}

/**
 * Complete Component IR
 */
export interface ComponentIR extends IRNode {
  schemaVersion: number;
  name: string;
  description?: string;
  category: ComponentCategory;
  meta: ComponentMeta;
  
  // Props
  props: PropDefinition[];
  
  // Styling
  styling: ComponentStyling;
  states?: StateDefinition[];
  
  // Accessibility
  accessibility?: AccessibilityConfig;
  
  // Compound component support
  compound?: boolean;
  parts?: ComponentPart[];
  context?: ComponentContext[];
  
  // Resolved references (populated during resolution)
  resolvedTokens?: Map<string, string>;
}
```

### 4.4 Pattern IR Types

```typescript
// src/ir/types/patterns.ts

import { IRNode, TokenRef } from './core';
import { ComponentIR, PropDefinition, StyleProperties } from './components';

/**
 * Pattern category
 */
export type PatternCategory =
  | 'layout'      // Page layouts, grid systems
  | 'navigation'  // Nav bars, sidebars, breadcrumbs
  | 'form'        // Form layouts, field groups
  | 'content'     // Cards, lists, media objects
  | 'feedback'    // Empty states, loading, errors
  | 'marketing';  // Hero, features, pricing

/**
 * Component instance in a pattern
 */
export interface ComponentInstance extends IRNode {
  component: string;  // Component name reference
  props?: Record<string, unknown>;
  children?: PatternNode[];
  slot?: string;  // Named slot placement
}

/**
 * HTML element in a pattern
 */
export interface ElementNode extends IRNode {
  element: string;  // HTML tag name
  props?: Record<string, unknown>;
  styling?: StyleProperties;
  children?: PatternNode[];
}

/**
 * Conditional rendering
 */
export interface ConditionalNode extends IRNode {
  type: 'conditional';
  condition: string;  // Prop or expression
  then: PatternNode[];
  else?: PatternNode[];
}

/**
 * Loop/iteration
 */
export interface LoopNode extends IRNode {
  type: 'loop';
  items: string;  // Prop name containing items
  itemAs: string;  // Variable name for each item
  indexAs?: string;  // Variable name for index
  template: PatternNode[];
}

/**
 * Slot placeholder
 */
export interface SlotNode extends IRNode {
  type: 'slot';
  name: string;
  fallback?: PatternNode[];
}

/**
 * Pattern node types
 */
export type PatternNode =
  | ComponentInstance
  | ElementNode
  | ConditionalNode
  | LoopNode
  | SlotNode;

/**
 * Pattern variant
 */
export interface PatternVariant extends IRNode {
  name: string;
  description?: string;
  props?: Record<string, unknown>;  // Prop overrides
  tree?: PatternNode[];  // Structure overrides
}

/**
 * Complete Pattern IR
 */
export interface PatternIR extends IRNode {
  schemaVersion: number;
  name: string;
  description?: string;
  category: PatternCategory;
  
  // Pattern props (passed through to components)
  props: PropDefinition[];
  
  // Component tree structure
  tree: PatternNode[];
  
  // Named variants
  variants?: PatternVariant[];
  
  // Components referenced by this pattern
  components: string[];
  
  // Resolved component references
  resolvedComponents?: Map<string, ComponentIR>;
}
```

### 4.5 Complete IR Type

```typescript
// src/ir/types/index.ts

export * from './core';
export * from './tokens';
export * from './components';
export * from './patterns';

import { ParseResult, ValidationIssue } from './core';
import { TokenIR } from './tokens';
import { ComponentIR } from './components';
import { PatternIR } from './patterns';

/**
 * Generator configuration
 */
export interface GeneratorConfig {
  outputDir: string;
  target: 'react' | 'vue' | 'svelte' | 'css' | 'figma';
  typescript: boolean;
  
  // React-specific
  react?: {
    importReact: boolean;
    cssStrategy: 'css-modules' | 'css-in-js' | 'tailwind' | 'vanilla-extract';
  };
  
  // CSS-specific
  css?: {
    prefix?: string;
    useCustomProperties: boolean;
    minify: boolean;
  };
}

/**
 * Complete design system IR
 */
export interface DesignSystemIR {
  /** Schema version for this IR */
  schemaVersion: number;
  
  /** Design system metadata */
  meta: {
    name: string;
    version: string;
    description?: string;
  };
  
  /** Token definitions */
  tokens: TokenIR;
  
  /** Component definitions */
  components: ComponentIR[];
  
  /** Pattern definitions */
  patterns: PatternIR[];
  
  /** Validation issues encountered */
  issues: ValidationIssue[];
}

/**
 * Input for generators
 */
export interface GeneratorInput {
  ir: DesignSystemIR;
  config: GeneratorConfig;
}

/**
 * Generator output
 */
export interface GeneratorOutput {
  files: GeneratedFile[];
  issues: ValidationIssue[];
}

/**
 * Single generated file
 */
export interface GeneratedFile {
  path: string;
  content: string;
  /** Generation provenance */
  provenance: {
    generator: string;
    version: string;
    specHash: string;
    timestamp: string;
    model?: string;
    promptVersion?: string;
  };
}
```

---

## 5. JSON Schema Definitions

### 5.1 Zod Schema Source (Single Source of Truth)

```typescript
// src/schemas/tokens.schema.ts

import { z } from 'zod';

/**
 * Token reference pattern: $category.path.to.token
 */
const tokenRefPattern = /^\$[a-zA-Z][a-zA-Z0-9]*(\.[a-zA-Z][a-zA-Z0-9]*)+$/;

export const TokenRefSchema = z.string().regex(tokenRefPattern, {
  message: 'Token reference must be in format: $category.path.to.token',
});

export const ColorValueSchema = z.union([
  z.string().regex(/^#([0-9a-fA-F]{3}|[0-9a-fA-F]{6}|[0-9a-fA-F]{8})$/, {
    message: 'Color must be a valid hex color',
  }),
  z.string().regex(/^rgb\(/, { message: 'Invalid RGB color' }),
  z.string().regex(/^hsl\(/, { message: 'Invalid HSL color' }),
  TokenRefSchema,
]);

export const DimensionValueSchema = z.union([
  z.string().regex(/^-?\d+(\.\d+)?(px|rem|em|%|vh|vw)?$/, {
    message: 'Invalid dimension value',
  }),
  z.number(),
  TokenRefSchema,
]);

export const ColorTokenSchema = z.object({
  type: z.literal('color').optional(),
  value: ColorValueSchema,
  description: z.string().optional(),
  deprecated: z.boolean().optional(),
});

export const ColorScaleSchema = z.record(
  z.union([z.string(), z.number()]).transform(String),
  ColorValueSchema
);

export const ColorsSchema = z.object({
  primitives: z.record(z.string(), ColorScaleSchema).optional(),
  semantic: z.record(z.string(), ColorValueSchema).optional(),
  text: z.record(z.string(), ColorValueSchema).optional(),
  surface: z.record(z.string(), ColorValueSchema).optional(),
  border: z.record(z.string(), ColorValueSchema).optional(),
  interactive: z.record(z.string(), ColorValueSchema).optional(),
});

export const TypographySchema = z.object({
  fontFamily: z.record(z.string(), z.string()),
  fontSize: z.record(z.string(), z.union([z.string(), z.number()])),
  fontWeight: z.record(z.string(), z.number()),
  lineHeight: z.record(z.string(), z.union([z.string(), z.number()])),
  letterSpacing: z.record(z.string(), z.string()).optional(),
  textStyles: z.record(z.string(), z.object({
    fontFamily: z.union([z.string(), TokenRefSchema]).optional(),
    fontSize: z.union([z.string(), TokenRefSchema]).optional(),
    fontWeight: z.union([z.number(), TokenRefSchema]).optional(),
    lineHeight: z.union([z.string(), z.number(), TokenRefSchema]).optional(),
    letterSpacing: z.union([z.string(), TokenRefSchema]).optional(),
    color: z.union([z.string(), TokenRefSchema]).optional(),
  })).optional(),
});

export const SpacingSchema = z.record(
  z.union([z.string(), z.number()]).transform(String),
  z.union([z.string(), z.number()])
);

export const RadiiSchema = z.record(z.string(), z.union([z.string(), z.number()]));

export const ShadowsSchema = z.record(z.string(), z.string());

export const ZIndexSchema = z.record(z.string(), z.number());

export const TransitionsSchema = z.object({
  durations: z.record(z.string(), z.string()),
  easings: z.record(z.string(), z.string()),
});

export const BreakpointsSchema = z.record(z.string(), z.string());

export const ThemeOverrideSchema = z.object({
  name: z.string().min(1),
  description: z.string().optional(),
  file: z.string().optional(),
});

export const TokensFileSchema = z.object({
  $schema: z.string().optional(),
  schemaVersion: z.number().int().positive(),
  meta: z.object({
    name: z.string().min(1),
    version: z.string().regex(/^\d+\.\d+\.\d+$/),
    description: z.string().optional(),
  }).optional(),
  imports: z.array(z.string()).optional(),
  colors: ColorsSchema.optional(),
  typography: TypographySchema.optional(),
  spacing: SpacingSchema.optional(),
  radii: RadiiSchema.optional(),
  shadows: ShadowsSchema.optional(),
  zIndex: ZIndexSchema.optional(),
  transitions: TransitionsSchema.optional(),
  breakpoints: BreakpointsSchema.optional(),
  animations: z.record(z.string(), z.any()).optional(),
  themes: z.object({
    default: z.string(),
    available: z.array(ThemeOverrideSchema),
  }).optional(),
});

export type TokensFile = z.infer<typeof TokensFileSchema>;
```

### 5.2 Component Schema

```typescript
// src/schemas/component.schema.ts

import { z } from 'zod';

const TokenRefSchema = z.string().regex(/^\$[a-zA-Z][a-zA-Z0-9]*(\.[a-zA-Z][a-zA-Z0-9]*)+$/);

// Prop type definitions
const PrimitiveTypeSchema = z.enum(['string', 'number', 'boolean']);

const EnumTypeSchema = z.object({
  type: z.literal('enum'),
  values: z.array(z.string()).min(1),
});

const SlotTypeSchema = z.object({
  type: z.literal('slot'),
  position: z.enum(['start', 'end']).optional(),
});

const RefTypeSchema = z.object({
  type: z.literal('ref'),
  element: z.string(),
});

const CallbackParamSchema = z.object({
  name: z.string(),
  type: z.union([PrimitiveTypeSchema, z.string()]),
});

const CallbackTypeSchema = z.object({
  type: z.literal('callback'),
  parameters: z.array(CallbackParamSchema).optional(),
  returns: z.string().optional(),
});

const ComponentRefTypeSchema = z.object({
  type: z.literal('componentRef'),
  accepts: z.array(z.string()),
});

const ObjectTypeSchema: z.ZodType<any> = z.object({
  type: z.literal('object'),
  properties: z.array(z.lazy(() => PropDefinitionSchema)),
});

const ArrayTypeSchema: z.ZodType<any> = z.object({
  type: z.literal('array'),
  items: z.lazy(() => PropTypeSchema),
  minItems: z.number().int().nonnegative().optional(),
  maxItems: z.number().int().positive().optional(),
});

const UnionTypeSchema: z.ZodType<any> = z.object({
  type: z.literal('union'),
  members: z.array(z.lazy(() => PropTypeSchema)),
});

const PropTypeSchema: z.ZodType<any> = z.union([
  PrimitiveTypeSchema,
  EnumTypeSchema,
  SlotTypeSchema,
  RefTypeSchema,
  CallbackTypeSchema,
  ComponentRefTypeSchema,
  ObjectTypeSchema,
  ArrayTypeSchema,
  UnionTypeSchema,
]);

const PropValidationSchema = z.object({
  minLength: z.number().int().nonnegative().optional(),
  maxLength: z.number().int().positive().optional(),
  pattern: z.string().optional(),
  min: z.number().optional(),
  max: z.number().optional(),
  step: z.number().optional(),
});

const PropDefinitionSchema = z.object({
  name: z.string().min(1).regex(/^[a-z][a-zA-Z0-9]*$/, {
    message: 'Prop name must be camelCase',
  }),
  type: PropTypeSchema,
  description: z.string().optional(),
  required: z.boolean().optional(),
  default: z.any().optional(),
  validation: PropValidationSchema.optional(),
  deprecated: z.boolean().optional(),
  deprecatedMessage: z.string().optional(),
});

// Style value (string, number, or token ref)
const StyleValueSchema = z.union([
  z.string(),
  z.number(),
  TokenRefSchema,
]);

const StylePropertiesSchema = z.record(z.string(), StyleValueSchema);

const VariantStylesSchema = z.record(z.string(), StylePropertiesSchema);

const CompoundVariantSchema = z.object({
  when: z.record(z.string(), z.any()),
  styling: StylePropertiesSchema,
});

const ComponentStylingSchema = z.object({
  base: StylePropertiesSchema,
  variants: z.record(z.string(), VariantStylesSchema).optional(),
  compoundVariants: z.array(CompoundVariantSchema).optional(),
});

const AriaBindingSchema = z.object({
  name: z.string().regex(/^aria-[a-z]+$/),
  bindTo: z.string().optional(),
  value: z.string().optional(),
});

const KeyboardHandlerSchema = z.object({
  key: z.string(),
  modifiers: z.array(z.enum(['ctrl', 'alt', 'shift', 'meta'])).optional(),
  action: z.string(),
  preventDefault: z.boolean().optional(),
});

const AccessibilityConfigSchema = z.object({
  role: z.string().optional(),
  focusable: z.boolean().optional(),
  ariaProps: z.array(AriaBindingSchema).optional(),
  keyboardNav: z.array(KeyboardHandlerSchema).optional(),
});

const StateDefinitionSchema = z.object({
  condition: z.string(),
  styling: ComponentStylingSchema,
});

const ComponentPartSchema = z.object({
  name: z.string().min(1).regex(/^[A-Z][a-zA-Z0-9]*$/, {
    message: 'Part name must be PascalCase',
  }),
  description: z.string().optional(),
  props: z.array(PropDefinitionSchema).optional(),
  styling: ComponentStylingSchema,
  states: z.record(z.string(), StateDefinitionSchema).optional(),
  accessibility: AccessibilityConfigSchema.optional(),
});

const ComponentContextSchema = z.object({
  name: z.string(),
  type: PropTypeSchema,
  providedBy: z.union([z.literal('root'), z.string()]),
});

export const ComponentFileSchema = z.object({
  $schema: z.string().optional(),
  schemaVersion: z.number().int().positive(),
  
  name: z.string().min(1).regex(/^[A-Z][a-zA-Z0-9]*$/, {
    message: 'Component name must be PascalCase',
  }),
  description: z.string().optional(),
  category: z.enum([
    'action', 'input', 'display', 'feedback', 
    'navigation', 'layout', 'overlay', 'utility'
  ]),
  
  meta: z.object({
    figmaUrl: z.string().url().optional(),
    status: z.enum(['draft', 'experimental', 'stable', 'deprecated']),
    since: z.string().regex(/^\d+\.\d+\.\d+$/),
    deprecatedIn: z.string().regex(/^\d+\.\d+\.\d+$/).nullable().optional(),
    deprecatedMessage: z.string().optional(),
    tags: z.array(z.string()).optional(),
  }),
  
  props: z.array(PropDefinitionSchema),
  
  styling: ComponentStylingSchema,
  
  states: z.record(z.string(), StateDefinitionSchema).optional(),
  
  accessibility: AccessibilityConfigSchema.optional(),
  
  // Compound component fields
  compound: z.boolean().optional(),
  parts: z.array(ComponentPartSchema).optional(),
  context: z.array(ComponentContextSchema).optional(),
});

export type ComponentFile = z.infer<typeof ComponentFileSchema>;
```

### 5.3 JSON Schema Generation

```typescript
// src/schemas/generate-json-schemas.ts

import { zodToJsonSchema } from 'zod-to-json-schema';
import { writeFileSync, mkdirSync } from 'fs';
import { join } from 'path';

import { TokensFileSchema } from './tokens.schema';
import { ComponentFileSchema } from './component.schema';

const SCHEMA_DIR = join(__dirname, '../../schemas');

// Ensure schema directory exists
mkdirSync(SCHEMA_DIR, { recursive: true });

// Generate token schema
const tokenJsonSchema = zodToJsonSchema(TokensFileSchema, {
  name: 'TokensFile',
  $refStrategy: 'none', // Inline all definitions for better IDE support
  definitions: {},
});

tokenJsonSchema.$id = 'https://dspec.dev/schemas/tokens/v1.json';
tokenJsonSchema.title = 'dspec Token Schema';
tokenJsonSchema.description = 'Schema for dspec design token files';

writeFileSync(
  join(SCHEMA_DIR, 'tokens.v1.schema.json'),
  JSON.stringify(tokenJsonSchema, null, 2)
);

// Generate component schema
const componentJsonSchema = zodToJsonSchema(ComponentFileSchema, {
  name: 'ComponentFile',
  $refStrategy: 'none',
  definitions: {},
});

componentJsonSchema.$id = 'https://dspec.dev/schemas/component/v1.json';
componentJsonSchema.title = 'dspec Component Schema';
componentJsonSchema.description = 'Schema for dspec component specification files';

writeFileSync(
  join(SCHEMA_DIR, 'component.v1.schema.json'),
  JSON.stringify(componentJsonSchema, null, 2)
);

console.log('✅ JSON Schemas generated successfully');
```

---

## 6. Reference Resolution

### 6.1 Reference Resolver

```typescript
// src/ir/resolver.ts

import { TokenRef, TokenIR, ComponentIR, DesignSystemIR, ValidationIssue } from './types';

/**
 * Token reference pattern
 */
const TOKEN_REF_REGEX = /\$([a-zA-Z][a-zA-Z0-9]*(?:\.[a-zA-Z][a-zA-Z0-9]*)+)/g;

/**
 * Result of reference resolution
 */
export interface ResolutionResult {
  resolved: DesignSystemIR;
  issues: ValidationIssue[];
}

/**
 * Reference resolution context
 */
interface ResolutionContext {
  tokens: TokenIR;
  components: Map<string, ComponentIR>;
  visited: Set<string>;
  path: string[];
}

/**
 * Parse a token reference string into parts
 */
export function parseTokenRef(ref: string): TokenRef | null {
  if (!ref.startsWith('$')) return null;
  
  const path = ref.slice(1).split('.');
  if (path.length < 2) return null;
  
  return {
    type: 'ref',
    path,
    raw: ref,
  };
}

/**
 * Resolve a token reference to its value
 */
export function resolveTokenRef(
  ref: TokenRef,
  tokens: TokenIR,
  visited: Set<string> = new Set()
): string | number | ValidationIssue {
  const refKey = ref.raw;
  
  // Circular reference detection
  if (visited.has(refKey)) {
    return {
      severity: 'error',
      code: 'CIRCULAR_REF',
      message: `Circular token reference detected: ${refKey}`,
      path: ref.path,
    };
  }
  
  visited.add(refKey);
  
  // Navigate to the token value
  let current: any = tokens.tokens;
  for (const segment of ref.path) {
    if (current === undefined || current === null) {
      return {
        severity: 'error',
        code: 'UNRESOLVED_REF',
        message: `Cannot resolve token reference: ${refKey}`,
        path: ref.path,
      };
    }
    current = current[segment];
  }
  
  if (current === undefined) {
    return {
      severity: 'error',
      code: 'UNRESOLVED_REF',
      message: `Token not found: ${refKey}`,
      path: ref.path,
    };
  }
  
  // If the value is itself a reference, resolve recursively
  if (typeof current === 'string' && current.startsWith('$')) {
    const nestedRef = parseTokenRef(current);
    if (nestedRef) {
      return resolveTokenRef(nestedRef, tokens, visited);
    }
  }
  
  // If it's a token object with a value property
  if (typeof current === 'object' && 'value' in current) {
    const value = current.value;
    if (typeof value === 'string' && value.startsWith('$')) {
      const nestedRef = parseTokenRef(value);
      if (nestedRef) {
        return resolveTokenRef(nestedRef, tokens, visited);
      }
    }
    return value;
  }
  
  return current;
}

/**
 * Extract and resolve all token references in a string
 */
export function resolveStringRefs(
  value: string,
  tokens: TokenIR
): string | ValidationIssue[] {
  const issues: ValidationIssue[] = [];
  
  const resolved = value.replace(TOKEN_REF_REGEX, (match, refPath) => {
    const ref = parseTokenRef('$' + refPath);
    if (!ref) return match;
    
    const result = resolveTokenRef(ref, tokens);
    if (typeof result === 'object' && 'severity' in result) {
      issues.push(result);
      return match; // Keep original on error
    }
    
    return String(result);
  });
  
  return issues.length > 0 ? issues : resolved;
}

/**
 * Resolve all references in a style properties object
 */
export function resolveStyleProperties(
  styles: Record<string, any>,
  tokens: TokenIR
): { resolved: Record<string, string | number>; issues: ValidationIssue[] } {
  const resolved: Record<string, string | number> = {};
  const issues: ValidationIssue[] = [];
  
  for (const [key, value] of Object.entries(styles)) {
    if (typeof value === 'string') {
      const result = resolveStringRefs(value, tokens);
      if (Array.isArray(result)) {
        issues.push(...result);
        resolved[key] = value;
      } else {
        resolved[key] = result;
      }
    } else if (typeof value === 'number') {
      resolved[key] = value;
    } else if (typeof value === 'object' && value?.type === 'ref') {
      const result = resolveTokenRef(value as TokenRef, tokens);
      if (typeof result === 'object' && 'severity' in result) {
        issues.push(result);
        resolved[key] = value.raw;
      } else {
        resolved[key] = result;
      }
    }
  }
  
  return { resolved, issues };
}

/**
 * Resolve all references in a component
 */
export function resolveComponentRefs(
  component: ComponentIR,
  ctx: ResolutionContext
): { resolved: ComponentIR; issues: ValidationIssue[] } {
  const issues: ValidationIssue[] = [];
  const resolvedTokens = new Map<string, string>();
  
  // Resolve base styling
  const { resolved: baseStyles, issues: baseIssues } = resolveStyleProperties(
    component.styling.base,
    ctx.tokens
  );
  issues.push(...baseIssues);
  
  // Resolve variant styling
  const resolvedVariants: Record<string, Record<string, Record<string, string | number>>> = {};
  if (component.styling.variants) {
    for (const [variantKey, variantMap] of Object.entries(component.styling.variants)) {
      resolvedVariants[variantKey] = {};
      for (const [valueKey, styles] of Object.entries(variantMap)) {
        const { resolved, issues: variantIssues } = resolveStyleProperties(styles, ctx.tokens);
        resolvedVariants[variantKey][valueKey] = resolved;
        issues.push(...variantIssues);
      }
    }
  }
  
  // Resolve state styling
  const resolvedStates: typeof component.states = {};
  if (component.states) {
    for (const [stateName, state] of Object.entries(component.states)) {
      const { resolved: stateBase, issues: stateIssues } = resolveStyleProperties(
        state.styling.base,
        ctx.tokens
      );
      issues.push(...stateIssues);
      
      resolvedStates[stateName] = {
        ...state,
        styling: {
          ...state.styling,
          base: stateBase,
        },
      };
    }
  }
  
  return {
    resolved: {
      ...component,
      styling: {
        ...component.styling,
        base: baseStyles,
        variants: Object.keys(resolvedVariants).length > 0 ? resolvedVariants as any : undefined,
      },
      states: Object.keys(resolvedStates).length > 0 ? resolvedStates : undefined,
      resolvedTokens,
    },
    issues,
  };
}

/**
 * Resolve component references in patterns
 */
export function resolvePatternComponentRefs(
  ir: DesignSystemIR
): ValidationIssue[] {
  const issues: ValidationIssue[] = [];
  const componentMap = new Map(ir.components.map(c => [c.name, c]));
  
  for (const pattern of ir.patterns) {
    pattern.resolvedComponents = new Map();
    
    for (const componentName of pattern.components) {
      const component = componentMap.get(componentName);
      if (!component) {
        issues.push({
          severity: 'error',
          code: 'UNRESOLVED_COMPONENT',
          message: `Pattern "${pattern.name}" references unknown component: ${componentName}`,
          path: ['patterns', pattern.name, 'components'],
        });
      } else {
        pattern.resolvedComponents.set(componentName, component);
      }
    }
  }
  
  return issues;
}

/**
 * Main resolution function
 */
export function resolveAllReferences(ir: DesignSystemIR): ResolutionResult {
  const issues: ValidationIssue[] = [];
  
  const ctx: ResolutionContext = {
    tokens: ir.tokens,
    components: new Map(ir.components.map(c => [c.name, c])),
    visited: new Set(),
    path: [],
  };
  
  // Resolve component token references
  const resolvedComponents = ir.components.map(component => {
    const { resolved, issues: componentIssues } = resolveComponentRefs(component, ctx);
    issues.push(...componentIssues);
    return resolved;
  });
  
  // Resolve pattern component references
  const patternIssues = resolvePatternComponentRefs({
    ...ir,
    components: resolvedComponents,
  });
  issues.push(...patternIssues);
  
  return {
    resolved: {
      ...ir,
      components: resolvedComponents,
      issues: [...ir.issues, ...issues],
    },
    issues,
  };
}
```

### 6.2 Dependency Graph

```typescript
// src/ir/dependency-graph.ts

import { DesignSystemIR, ComponentIR, PatternIR, ValidationIssue } from './types';

/**
 * Node in the dependency graph
 */
interface DependencyNode {
  type: 'token' | 'component' | 'pattern';
  name: string;
  dependencies: Set<string>;
  dependents: Set<string>;
}

/**
 * Dependency graph for the design system
 */
export class DependencyGraph {
  private nodes: Map<string, DependencyNode> = new Map();
  
  constructor(ir: DesignSystemIR) {
    this.buildGraph(ir);
  }
  
  private nodeKey(type: string, name: string): string {
    return `${type}:${name}`;
  }
  
  private buildGraph(ir: DesignSystemIR): void {
    // Add component nodes
    for (const component of ir.components) {
      const key = this.nodeKey('component', component.name);
      this.nodes.set(key, {
        type: 'component',
        name: component.name,
        dependencies: this.extractComponentDeps(component),
        dependents: new Set(),
      });
    }
    
    // Add pattern nodes
    for (const pattern of ir.patterns) {
      const key = this.nodeKey('pattern', pattern.name);
      const deps = new Set<string>();
      
      for (const componentName of pattern.components) {
        deps.add(this.nodeKey('component', componentName));
      }
      
      this.nodes.set(key, {
        type: 'pattern',
        name: pattern.name,
        dependencies: deps,
        dependents: new Set(),
      });
    }
    
    // Build reverse dependencies (dependents)
    for (const [key, node] of this.nodes) {
      for (const depKey of node.dependencies) {
        const depNode = this.nodes.get(depKey);
        if (depNode) {
          depNode.dependents.add(key);
        }
      }
    }
  }
  
  private extractComponentDeps(component: ComponentIR): Set<string> {
    const deps = new Set<string>();
    
    // Token dependencies are tracked separately
    // Here we track component dependencies (e.g., from parts or extensions)
    
    if (component.parts) {
      // Compound components don't create external dependencies
      // Parts are internal to the component
    }
    
    return deps;
  }
  
  /**
   * Get all components that depend on a token
   */
  getTokenDependents(tokenPath: string): string[] {
    // This would require token-level tracking
    // For now, return all components as potentially affected
    return Array.from(this.nodes.entries())
      .filter(([_, node]) => node.type === 'component')
      .map(([_, node]) => node.name);
  }
  
  /**
   * Get regeneration order for components
   */
  getRegenerationOrder(changedComponents: string[]): string[] {
    const toRegenerate = new Set<string>();
    const queue = changedComponents.map(c => this.nodeKey('component', c));
    
    while (queue.length > 0) {
      const key = queue.shift()!;
      if (toRegenerate.has(key)) continue;
      
      toRegenerate.add(key);
      
      const node = this.nodes.get(key);
      if (node) {
        for (const depKey of node.dependents) {
          queue.push(depKey);
        }
      }
    }
    
    // Topological sort
    return this.topologicalSort(toRegenerate);
  }
  
  /**
   * Detect circular dependencies
   */
  detectCycles(): ValidationIssue[] {
    const issues: ValidationIssue[] = [];
    const visited = new Set<string>();
    const recursionStack = new Set<string>();
    
    const dfs = (key: string, path: string[]): boolean => {
      visited.add(key);
      recursionStack.add(key);
      
      const node = this.nodes.get(key);
      if (!node) return false;
      
      for (const depKey of node.dependencies) {
        if (!visited.has(depKey)) {
          if (dfs(depKey, [...path, key])) return true;
        } else if (recursionStack.has(depKey)) {
          issues.push({
            severity: 'error',
            code: 'CIRCULAR_DEPENDENCY',
            message: `Circular dependency detected: ${[...path, key, depKey].join(' -> ')}`,
            path: path,
          });
          return true;
        }
      }
      
      recursionStack.delete(key);
      return false;
    };
    
    for (const key of this.nodes.keys()) {
      if (!visited.has(key)) {
        dfs(key, []);
      }
    }
    
    return issues;
  }
  
  private topologicalSort(nodes: Set<string>): string[] {
    const result: string[] = [];
    const visited = new Set<string>();
    
    const visit = (key: string) => {
      if (visited.has(key) || !nodes.has(key)) return;
      visited.add(key);
      
      const node = this.nodes.get(key);
      if (node) {
        for (const depKey of node.dependencies) {
          visit(depKey);
        }
      }
      
      result.push(node?.name || key);
    };
    
    for (const key of nodes) {
      visit(key);
    }
    
    return result;
  }
}
```

---

## 7. Schema Versioning & Migration

### 7.1 Version Management

```typescript
// src/ir/versioning.ts

import { SchemaVersion, ValidationIssue } from './types';

/**
 * Current schema version
 */
export const CURRENT_SCHEMA_VERSION: SchemaVersion = {
  major: 1,
  minor: 0,
  patch: 0,
};

/**
 * Minimum supported schema version
 */
export const MIN_SUPPORTED_VERSION: SchemaVersion = {
  major: 1,
  minor: 0,
  patch: 0,
};

/**
 * Parse version string
 */
export function parseVersion(version: string | number): SchemaVersion {
  if (typeof version === 'number') {
    return { major: version, minor: 0, patch: 0 };
  }
  
  const parts = version.split('.').map(Number);
  return {
    major: parts[0] || 1,
    minor: parts[1] || 0,
    patch: parts[2] || 0,
  };
}

/**
 * Compare versions
 */
export function compareVersions(a: SchemaVersion, b: SchemaVersion): number {
  if (a.major !== b.major) return a.major - b.major;
  if (a.minor !== b.minor) return a.minor - b.minor;
  return a.patch - b.patch;
}

/**
 * Check if version is supported
 */
export function isVersionSupported(version: SchemaVersion): boolean {
  return compareVersions(version, MIN_SUPPORTED_VERSION) >= 0 &&
         compareVersions(version, CURRENT_SCHEMA_VERSION) <= 0;
}

/**
 * Version check result
 */
export interface VersionCheckResult {
  supported: boolean;
  needsMigration: boolean;
  fromVersion: SchemaVersion;
  toVersion: SchemaVersion;
  issues: ValidationIssue[];
}

/**
 * Check schema version compatibility
 */
export function checkVersion(schemaVersion: number | string): VersionCheckResult {
  const version = parseVersion(schemaVersion);
  const issues: ValidationIssue[] = [];
  
  if (compareVersions(version, MIN_SUPPORTED_VERSION) < 0) {
    issues.push({
      severity: 'error',
      code: 'VERSION_TOO_OLD',
      message: `Schema version ${schemaVersion} is no longer supported. Minimum: ${MIN_SUPPORTED_VERSION.major}.${MIN_SUPPORTED_VERSION.minor}.${MIN_SUPPORTED_VERSION.patch}`,
      path: ['schemaVersion'],
    });
  }
  
  if (compareVersions(version, CURRENT_SCHEMA_VERSION) > 0) {
    issues.push({
      severity: 'error',
      code: 'VERSION_TOO_NEW',
      message: `Schema version ${schemaVersion} is newer than supported. Current: ${CURRENT_SCHEMA_VERSION.major}.${CURRENT_SCHEMA_VERSION.minor}.${CURRENT_SCHEMA_VERSION.patch}`,
      path: ['schemaVersion'],
    });
  }
  
  return {
    supported: issues.length === 0,
    needsMigration: compareVersions(version, CURRENT_SCHEMA_VERSION) < 0,
    fromVersion: version,
    toVersion: CURRENT_SCHEMA_VERSION,
    issues,
  };
}
```

### 7.2 Migration System

```typescript
// src/ir/migrations/index.ts

import { ValidationIssue } from '../types';

/**
 * Migration function signature
 */
export type MigrationFn = (data: any) => {
  migrated: any;
  issues: ValidationIssue[];
};

/**
 * Migration definition
 */
export interface Migration {
  fromVersion: number;
  toVersion: number;
  description: string;
  changes: string[];
  migrate: MigrationFn;
}

/**
 * Registry of all migrations
 */
const migrations: Migration[] = [];

/**
 * Register a migration
 */
export function registerMigration(migration: Migration): void {
  migrations.push(migration);
  migrations.sort((a, b) => a.fromVersion - b.fromVersion);
}

/**
 * Get migrations needed for a version upgrade
 */
export function getMigrationsForUpgrade(from: number, to: number): Migration[] {
  return migrations.filter(m => m.fromVersion >= from && m.toVersion <= to);
}

/**
 * Apply migrations sequentially
 */
export function applyMigrations(
  data: any,
  fromVersion: number,
  toVersion: number
): { migrated: any; issues: ValidationIssue[] } {
  const applicableMigrations = getMigrationsForUpgrade(fromVersion, toVersion);
  const allIssues: ValidationIssue[] = [];
  
  let current = data;
  for (const migration of applicableMigrations) {
    const { migrated, issues } = migration.migrate(current);
    current = migrated;
    allIssues.push(...issues);
  }
  
  // Update schema version
  current.schemaVersion = toVersion;
  
  return { migrated: current, issues: allIssues };
}

// Example migration (for future use)
registerMigration({
  fromVersion: 1,
  toVersion: 2,
  description: 'Add accessibility.focusRing, deprecate styling.focus',
  changes: [
    'Added: component.accessibility.focusRing',
    'Deprecated: component.styling.focus (use accessibility.focusRing)',
  ],
  migrate: (data) => {
    const issues: ValidationIssue[] = [];
    
    // Example migration logic
    if (data.components) {
      for (const component of data.components) {
        if (component.styling?.focus) {
          // Migrate focus styling to accessibility.focusRing
          if (!component.accessibility) {
            component.accessibility = {};
          }
          component.accessibility.focusRing = component.styling.focus;
          
          issues.push({
            severity: 'warning',
            code: 'DEPRECATED_FIELD',
            message: `Component "${component.name}": styling.focus is deprecated, migrated to accessibility.focusRing`,
            path: ['components', component.name, 'styling', 'focus'],
          });
        }
      }
    }
    
    return { migrated: data, issues };
  },
});
```

### 7.3 Migration CLI Command

```typescript
// src/cli/commands/migrate.ts

import { Command } from 'commander';
import { readFileSync, writeFileSync } from 'fs';
import { parse, stringify } from 'yaml';
import { checkVersion } from '../ir/versioning';
import { applyMigrations } from '../ir/migrations';

export const migrateCommand = new Command('migrate')
  .description('Migrate spec files to the current schema version')
  .argument('<files...>', 'Spec files to migrate')
  .option('--dry-run', 'Show what would be migrated without writing')
  .option('--from <version>', 'Source version (auto-detected if not specified)')
  .option('--to <version>', 'Target version (defaults to current)')
  .action(async (files: string[], options) => {
    for (const file of files) {
      console.log(`\nProcessing: ${file}`);
      
      const content = readFileSync(file, 'utf-8');
      const data = parse(content);
      
      const versionCheck = checkVersion(data.schemaVersion);
      
      if (!versionCheck.needsMigration) {
        console.log('  ✓ Already at current version');
        continue;
      }
      
      const fromVersion = options.from 
        ? parseInt(options.from, 10) 
        : versionCheck.fromVersion.major;
      const toVersion = options.to 
        ? parseInt(options.to, 10) 
        : versionCheck.toVersion.major;
      
      console.log(`  Migrating from v${fromVersion} to v${toVersion}...`);
      
      const { migrated, issues } = applyMigrations(data, fromVersion, toVersion);
      
      for (const issue of issues) {
        const prefix = issue.severity === 'error' ? '✗' : '⚠';
        console.log(`  ${prefix} ${issue.message}`);
      }
      
      if (options.dryRun) {
        console.log('  [Dry run - no changes written]');
        continue;
      }
      
      const output = stringify(migrated, {
        lineWidth: 120,
        defaultKeyType: 'PLAIN',
        defaultStringType: 'QUOTE_DOUBLE',
      });
      
      writeFileSync(file, output);
      console.log(`  ✓ Migrated successfully`);
    }
  });
```

---

## 8. Validation Pipeline

### 8.1 Complete Validation Pipeline

```typescript
// src/ir/validate.ts

import { parse as parseYaml, parseDocument } from 'yaml';
import { ZodError } from 'zod';
import { TokensFileSchema } from '../schemas/tokens.schema';
import { ComponentFileSchema } from '../schemas/component.schema';
import { 
  ParseResult, 
  ValidationIssue, 
  DesignSystemIR, 
  TokenIR, 
  ComponentIR 
} from './types';
import { checkVersion, CURRENT_SCHEMA_VERSION } from './versioning';
import { applyMigrations } from './migrations';
import { resolveAllReferences, DependencyGraph } from './resolver';

/**
 * Convert Zod error to validation issues
 */
function zodErrorToIssues(error: ZodError, file: string): ValidationIssue[] {
  return error.issues.map(issue => ({
    severity: 'error',
    code: 'SCHEMA_VALIDATION',
    message: issue.message,
    path: issue.path.map(String),
    source: {
      file,
      line: 0,  // Would need YAML AST for accurate line numbers
      column: 0,
    },
  }));
}

/**
 * Parse and validate a token file
 */
export async function parseTokenFile(
  content: string,
  filePath: string
): Promise<ParseResult<TokenIR>> {
  const issues: ValidationIssue[] = [];
  
  // 1. Parse YAML
  let raw: any;
  try {
    const doc = parseDocument(content);
    if (doc.errors.length > 0) {
      return {
        success: false,
        errors: doc.errors.map(e => ({
          severity: 'error',
          code: 'YAML_PARSE',
          message: e.message,
          path: [],
          source: { file: filePath, line: e.linePos?.[0]?.line || 0, column: e.linePos?.[0]?.col || 0 },
        })),
        warnings: [],
      };
    }
    raw = doc.toJS();
  } catch (e: any) {
    return {
      success: false,
      errors: [{
        severity: 'error',
        code: 'YAML_PARSE',
        message: e.message,
        path: [],
        source: { file: filePath, line: 0, column: 0 },
      }],
      warnings: [],
    };
  }
  
  // 2. Check version
  const versionCheck = checkVersion(raw.schemaVersion || 1);
  if (!versionCheck.supported) {
    return {
      success: false,
      errors: versionCheck.issues,
      warnings: [],
    };
  }
  
  // 3. Apply migrations if needed
  if (versionCheck.needsMigration) {
    const { migrated, issues: migrationIssues } = applyMigrations(
      raw,
      versionCheck.fromVersion.major,
      versionCheck.toVersion.major
    );
    raw = migrated;
    issues.push(...migrationIssues);
  }
  
  // 4. Schema validation
  const result = TokensFileSchema.safeParse(raw);
  if (!result.success) {
    return {
      success: false,
      errors: zodErrorToIssues(result.error, filePath),
      warnings: issues.filter(i => i.severity === 'warning'),
    };
  }
  
  // 5. Build TokenIR
  const tokenIR: TokenIR = {
    id: filePath,
    schemaVersion: CURRENT_SCHEMA_VERSION.major,
    meta: result.data.meta || { name: 'Tokens', version: '1.0.0' },
    tokens: result.data as any,  // Type assertion for now
    themes: [],
    source: { file: filePath, line: 1, column: 1 },
  };
  
  return {
    success: true,
    data: tokenIR,
    errors: issues.filter(i => i.severity === 'error'),
    warnings: issues.filter(i => i.severity === 'warning'),
  };
}

/**
 * Parse and validate a component file
 */
export async function parseComponentFile(
  content: string,
  filePath: string
): Promise<ParseResult<ComponentIR>> {
  const issues: ValidationIssue[] = [];
  
  // 1. Parse YAML
  let raw: any;
  try {
    const doc = parseDocument(content);
    if (doc.errors.length > 0) {
      return {
        success: false,
        errors: doc.errors.map(e => ({
          severity: 'error',
          code: 'YAML_PARSE',
          message: e.message,
          path: [],
          source: { file: filePath, line: e.linePos?.[0]?.line || 0, column: e.linePos?.[0]?.col || 0 },
        })),
        warnings: [],
      };
    }
    raw = doc.toJS();
  } catch (e: any) {
    return {
      success: false,
      errors: [{
        severity: 'error',
        code: 'YAML_PARSE',
        message: e.message,
        path: [],
        source: { file: filePath, line: 0, column: 0 },
      }],
      warnings: [],
    };
  }
  
  // 2. Check version
  const versionCheck = checkVersion(raw.schemaVersion || 1);
  if (!versionCheck.supported) {
    return {
      success: false,
      errors: versionCheck.issues,
      warnings: [],
    };
  }
  
  // 3. Apply migrations if needed
  if (versionCheck.needsMigration) {
    const { migrated, issues: migrationIssues } = applyMigrations(
      raw,
      versionCheck.fromVersion.major,
      versionCheck.toVersion.major
    );
    raw = migrated;
    issues.push(...migrationIssues);
  }
  
  // 4. Schema validation
  const result = ComponentFileSchema.safeParse(raw);
  if (!result.success) {
    return {
      success: false,
      errors: zodErrorToIssues(result.error, filePath),
      warnings: issues.filter(i => i.severity === 'warning'),
    };
  }
  
  // 5. Build ComponentIR
  const componentIR: ComponentIR = {
    id: `component:${result.data.name}`,
    schemaVersion: CURRENT_SCHEMA_VERSION.major,
    name: result.data.name,
    description: result.data.description,
    category: result.data.category,
    meta: result.data.meta,
    props: result.data.props as any,
    styling: result.data.styling as any,
    states: result.data.states as any,
    accessibility: result.data.accessibility as any,
    compound: result.data.compound,
    parts: result.data.parts as any,
    context: result.data.context as any,
    source: { file: filePath, line: 1, column: 1 },
  };
  
  return {
    success: true,
    data: componentIR,
    errors: issues.filter(i => i.severity === 'error'),
    warnings: issues.filter(i => i.severity === 'warning'),
  };
}

/**
 * Semantic validation after parsing
 */
export function semanticValidation(ir: DesignSystemIR): ValidationIssue[] {
  const issues: ValidationIssue[] = [];
  
  // Build dependency graph
  const graph = new DependencyGraph(ir);
  
  // Check for circular dependencies
  issues.push(...graph.detectCycles());
  
  // Validate prop types reference valid tokens
  for (const component of ir.components) {
    // Check that enum props have at least one value
    for (const prop of component.props) {
      if (typeof prop.type === 'object' && prop.type.type === 'enum') {
        if (prop.type.values.length === 0) {
          issues.push({
            severity: 'error',
            code: 'EMPTY_ENUM',
            message: `Prop "${prop.name}" in component "${component.name}" has empty enum values`,
            path: ['components', component.name, 'props', prop.name],
          });
        }
      }
    }
    
    // Check styling variants match props
    if (component.styling.variants) {
      for (const variantKey of Object.keys(component.styling.variants)) {
        const prop = component.props.find(p => p.name === variantKey);
        if (!prop) {
          issues.push({
            severity: 'warning',
            code: 'ORPHAN_VARIANT',
            message: `Styling variant "${variantKey}" in component "${component.name}" has no matching prop`,
            path: ['components', component.name, 'styling', 'variants', variantKey],
          });
        }
      }
    }
  }
  
  return issues;
}

/**
 * Full validation pipeline
 */
export async function validateDesignSystem(
  tokenFiles: { path: string; content: string }[],
  componentFiles: { path: string; content: string }[],
  patternFiles: { path: string; content: string }[] = []
): Promise<ParseResult<DesignSystemIR>> {
  const errors: ValidationIssue[] = [];
  const warnings: ValidationIssue[] = [];
  
  // Parse tokens
  let tokens: TokenIR | undefined;
  for (const { path, content } of tokenFiles) {
    const result = await parseTokenFile(content, path);
    errors.push(...result.errors);
    warnings.push(...result.warnings);
    if (result.success && result.data) {
      tokens = result.data;  // Use last token file (would merge in practice)
    }
  }
  
  if (!tokens) {
    return {
      success: false,
      errors: [...errors, {
        severity: 'error',
        code: 'NO_TOKENS',
        message: 'No valid token files found',
        path: [],
      }],
      warnings,
    };
  }
  
  // Parse components
  const components: ComponentIR[] = [];
  for (const { path, content } of componentFiles) {
    const result = await parseComponentFile(content, path);
    errors.push(...result.errors);
    warnings.push(...result.warnings);
    if (result.success && result.data) {
      components.push(result.data);
    }
  }
  
  // Build initial IR
  const ir: DesignSystemIR = {
    schemaVersion: CURRENT_SCHEMA_VERSION.major,
    meta: tokens.meta,
    tokens,
    components,
    patterns: [],
    issues: [],
  };
  
  // Resolve references
  const { resolved, issues: refIssues } = resolveAllReferences(ir);
  errors.push(...refIssues.filter(i => i.severity === 'error'));
  warnings.push(...refIssues.filter(i => i.severity === 'warning'));
  
  // Semantic validation
  const semanticIssues = semanticValidation(resolved);
  errors.push(...semanticIssues.filter(i => i.severity === 'error'));
  warnings.push(...semanticIssues.filter(i => i.severity === 'warning'));
  
  return {
    success: errors.length === 0,
    data: resolved,
    errors,
    warnings,
  };
}
```

---

## Appendix A: VS Code Configuration

```json
// .vscode/settings.json
{
  "yaml.schemas": {
    "./schemas/tokens.v1.schema.json": ["tokens/**/*.yaml", "tokens.yaml"],
    "./schemas/component.v1.schema.json": ["components/**/*.yaml"]
  },
  "yaml.validate": true,
  "yaml.completion": true,
  "yaml.hover": true,
  "[yaml]": {
    "editor.defaultFormatter": "redhat.vscode-yaml",
    "editor.autoIndent": "full",
    "editor.tabSize": 2
  }
}
```

## Appendix B: Recommended Directory Structure

```
project/
├── specs/
│   ├── tokens/
│   │   ├── index.yaml           # Main token file with imports
│   │   ├── colors.yaml
│   │   ├── typography.yaml
│   │   ├── spacing.yaml
│   │   └── themes/
│   │       ├── light.yaml
│   │       └── dark.yaml
│   ├── components/
│   │   ├── button.yaml
│   │   ├── input.yaml
│   │   └── select.yaml
│   └── patterns/
│       └── form-field.yaml
├── schemas/
│   ├── tokens.v1.schema.json    # Generated JSON Schema
│   └── component.v1.schema.json
├── src/
│   ├── ir/
│   │   ├── types/
│   │   │   ├── core.ts
│   │   │   ├── tokens.ts
│   │   │   ├── components.ts
│   │   │   └── patterns.ts
│   │   ├── resolver.ts
│   │   ├── validate.ts
│   │   └── versioning.ts
│   └── schemas/
│       ├── tokens.schema.ts     # Zod schemas (source of truth)
│       └── component.schema.ts
└── .vscode/
    └── settings.json
```

---

## Summary

This architecture provides:

1. **Complete TypeScript Types** for TokenIR, ComponentIR, and PatternIR with full IDE support
2. **Zod Schemas** as the single source of truth for validation and type inference
3. **JSON Schema Generation** for IDE autocomplete via YAML Language Server
4. **Reference Resolution** for token refs (`$colors.primary`) and component refs
5. **Schema Versioning** with explicit migration support
6. **Comprehensive Validation Pipeline** from YAML parsing through semantic validation

The design follows the plan's requirements for:
- Phase 1.2-1.3: Schema & Parser, IR Specification
- Support for Phase 2's compound components
- Extensibility for multi-platform generation
