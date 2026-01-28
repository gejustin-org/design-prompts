# Design Token Tools Landscape Research

> **Research Date:** January 2026  
> **Goal:** Identify free/open-source tools for compiling design tokens (YAML/JSON) to multiple outputs (CSS, TypeScript, Swift, Figma JSON)

---

## Executive Summary

The design token ecosystem has matured significantly. **Style Dictionary** remains the dominant open-source solution with excellent extensibility. **Terrazzo** is a newer DTCG-native alternative worth considering. The **W3C DTCG spec** is nearing finalization and should be our canonical format.

### Quick Recommendations

| Need | Recommendation |
|------|----------------|
| **Primary compiler** | Style Dictionary v4 |
| **DTCG compliance** | Adopt W3C DTCG format + use `@tokens-studio/sd-transforms` |
| **Swift output** | Terrazzo (native support) or custom Style Dictionary format |
| **Figma integration** | Tokens Studio Figma plugin (free tier) + Style Dictionary |
| **Build our own?** | No — extend existing tools instead |

---

## Tool Analysis

### 1. Style Dictionary (Amazon → Community)

**Repository:** https://github.com/style-dictionary/style-dictionary  
**License:** Apache 2.0 ✅  
**NPM:** `style-dictionary`  
**Status:** Actively maintained, v4 released

#### Overview
Style Dictionary is the de facto standard for design token compilation. Originally from Amazon, it's now a community-maintained project. It defines tokens in JSON/JS, transforms them through a pipeline, and outputs to any platform format.

#### Input Formats Supported
- JSON
- JSON5
- JavaScript modules (for computed values)
- YAML (via custom parser)

#### Output Formats (Built-in)
| Platform | Formats |
|----------|---------|
| **Web** | CSS custom properties, SCSS variables, LESS, Stylus, ES modules, CommonJS |
| **iOS** | Swift structs, Objective-C macros, JSON |
| **Android** | XML resources (colors, dimens, strings) |
| **Flutter** | Dart classes |
| **Documentation** | HTML, Markdown, JSON |

#### Extensibility
Style Dictionary's greatest strength is extensibility:

```javascript
// Custom transform
StyleDictionary.registerTransform({
  name: 'size/pxToRem',
  type: 'value',
  filter: token => token.type === 'dimension',
  transform: token => `${parseFloat(token.value) / 16}rem`
});

// Custom format
StyleDictionary.registerFormat({
  name: 'figma/json',
  format: ({ dictionary }) => {
    // Generate Figma-compatible JSON
    return JSON.stringify(figmaFormat(dictionary.allTokens));
  }
});
```

#### Strengths
- ✅ Extremely extensible (transforms, formats, filters, parsers)
- ✅ Excellent documentation
- ✅ Large community, many examples
- ✅ Platform-agnostic architecture
- ✅ Active development (v4 modernizes the API)

#### Limitations
- ⚠️ Not DTCG-native (requires transforms for DTCG compliance)
- ⚠️ No built-in Swift with modern patterns (need custom format)
- ⚠️ No native Figma JSON export (need custom format)

#### Integration Strategy
**Use directly with extensions.** Style Dictionary should be our core compiler. We'll write custom formats for:
- Figma JSON export
- Enhanced Swift output
- TypeScript type generation

---

### 2. Tokens Studio

**Website:** https://tokens.studio  
**Figma Plugin:** https://github.com/tokens-studio/figma-plugin  
**SD Transforms:** https://github.com/tokens-studio/sd-transforms  
**License:** MIT (open-source components), proprietary (Studio platform)

#### Free vs Paid

| Feature | Free | Pro (Paid) |
|---------|------|-----------|
| Figma plugin (core) | ✅ | ✅ |
| JSON token export | ✅ | ✅ |
| Git sync (GitHub, GitLab) | ✅ | ✅ |
| Basic theming | ✅ | ✅ |
| Multi-dimensional themes | ❌ | ✅ |
| Team collaboration | ❌ | ✅ |
| Studio Platform (web app) | ❌ | ✅ |

#### Key Open-Source Package: `@tokens-studio/sd-transforms`

This is the critical piece for us — it bridges Tokens Studio's format with Style Dictionary:

```javascript
import { register } from '@tokens-studio/sd-transforms';
import StyleDictionary from 'style-dictionary';

register(StyleDictionary);

const sd = new StyleDictionary({
  source: ['tokens/**/*.json'],
  preprocessors: ['tokens-studio'],
  platforms: {
    css: {
      transformGroup: 'tokens-studio',
      buildPath: 'build/css/',
      files: [{ destination: 'variables.css', format: 'css/variables' }]
    }
  }
});
```

#### What sd-transforms Provides
- Maps Tokens Studio format to DTCG format
- Math expression evaluation
- Color modifier transforms
- Typography/shadow composite expansion
- Theme permutation helpers

#### Integration Strategy
**Use the free Figma plugin + sd-transforms.** This gives us:
- Visual token management in Figma
- JSON export to Git
- Full Style Dictionary integration via sd-transforms

---

### 3. W3C Design Tokens Community Group (DTCG) Specification

**Spec URL:** https://tr.designtokens.org/format/  
**Status:** Draft (2025.10), but stable enough for adoption  
**GitHub:** https://github.com/design-tokens/community-group

#### Overview
The DTCG spec defines a standard JSON format for design tokens. It's becoming the interchange format that tools are converging on.

#### Key Format Features

```json
{
  "color": {
    "brand": {
      "primary": {
        "$type": "color",
        "$value": "#0070d2",
        "$description": "Primary brand color"
      }
    }
  },
  "spacing": {
    "$type": "dimension",
    "small": { "$value": "8px" },
    "medium": { "$value": "16px" }
  }
}
```

#### Defined Token Types
- `color` — sRGB, P3, and other color spaces
- `dimension` — sizes with units (px, rem, etc.)
- `fontFamily`, `fontWeight`, `duration`
- `cubicBezier` — easing functions
- `number` — unitless numbers
- **Composite types:** `typography`, `shadow`, `border`, `gradient`, `transition`

#### File Extensions
- `.tokens` (preferred)
- `.tokens.json` (for editor compatibility)

#### Recommendation
**Adopt DTCG format as our canonical source.** Benefits:
- Tool interoperability (Figma, Style Dictionary, Terrazzo, etc.)
- Future-proofed as the standard solidifies
- Self-documenting with `$type` and `$description`

---

### 4. Theo (Salesforce)

**Repository:** https://github.com/salesforce-ux/theo  
**License:** MIT ✅  
**Status:** ⚠️ Maintenance mode (last significant update: 2020)

#### Overview
Theo was an early design token tool from Salesforce. It uses YAML/JSON input and can output to multiple platforms.

#### Input Formats
- YAML (primary)
- JSON

#### Output Formats
- CSS custom properties
- SCSS, Sass, Less, Stylus
- JavaScript modules (ES6, CommonJS)
- Android XML
- iOS JSON
- Aura (Salesforce-specific)
- HTML documentation

#### Extensibility
```javascript
theo.registerFormat('array.js', `
  module.exports = [
    {{#each props as |prop|}}
    ['{{camelcase prop.name}}', '{{prop.value}}'],
    {{/each}}
  ]
`);

theo.registerValueTransform('easing/web', 
  prop => prop.get('type') === 'easing',
  prop => `cubic-bezier(${prop.get('value').join(', ')})`
);
```

#### Assessment
| Aspect | Rating |
|--------|--------|
| Capabilities | ⭐⭐⭐ Good but dated |
| Maintenance | ⚠️ Stale |
| Community | ⚠️ Declining |
| DTCG Support | ❌ None |

#### Recommendation
**Do not use.** Theo is effectively abandoned. Style Dictionary is the spiritual successor with better maintenance and features.

---

### 5. Terrazzo (formerly Cobalt UI)

**Repository:** https://github.com/terrazzoapp/terrazzo  
**Website:** https://terrazzo.app  
**License:** MIT ✅  
**Status:** Active development

#### Overview
Terrazzo is a newer DTCG-native token compiler. It's opinionated about following the DTCG spec strictly, which can be both a strength and limitation.

#### Input Formats
- DTCG JSON only (strict compliance)

#### Output Formats (via plugins)
- CSS (`@terrazzo/plugin-css`)
- Sass (`@terrazzo/plugin-sass`)
- JavaScript/TypeScript (`@terrazzo/plugin-js`)
- **Swift** (`@terrazzo/plugin-swift`) ✅
- Tailwind CSS (`@terrazzo/plugin-tailwind`)

#### Usage
```bash
npm i -D @terrazzo/cli @terrazzo/plugin-css @terrazzo/plugin-swift
npx tz init
npx tz build
```

```javascript
// terrazzo.config.js
import css from '@terrazzo/plugin-css';
import swift from '@terrazzo/plugin-swift';

export default {
  tokens: './tokens.json',
  outDir: './build',
  plugins: [
    css(),
    swift()
  ]
};
```

#### Strengths
- ✅ DTCG-native (no transforms needed)
- ✅ Built-in Swift support
- ✅ Modern architecture
- ✅ CLI-first design

#### Limitations
- ⚠️ Smaller community than Style Dictionary
- ⚠️ Less flexible for non-standard tokens
- ⚠️ Fewer integrations/examples

#### Integration Strategy
**Consider for Swift output.** Terrazzo's Swift plugin may be better than building our own Style Dictionary format. Could use both:
- Style Dictionary for CSS/TS/Figma
- Terrazzo for Swift (if needed)

---

### 6. Other Tools Evaluated

#### Diez
**Status:** ❌ Discontinued  
Was acquired and shut down. Do not use.

#### Specify
**Status:** Paid SaaS  
Not open-source. Skip.

#### Supernova
**Status:** Paid SaaS  
Enterprise design system platform. Not relevant for our open-source needs.

---

## Format Comparison

| Tool | DTCG Native | Custom Format | JSON | YAML |
|------|-------------|---------------|------|------|
| Style Dictionary | Via transforms | ✅ | ✅ | Via parser |
| Terrazzo | ✅ | ❌ | ✅ | ❌ |
| Theo | ❌ | ✅ | ✅ | ✅ |
| Tokens Studio | Via sd-transforms | ✅ | ✅ | ❌ |

---

## Output Coverage Matrix

| Output | Style Dictionary | Terrazzo | Theo | Custom Needed |
|--------|-----------------|----------|------|---------------|
| CSS Variables | ✅ Built-in | ✅ Plugin | ✅ | ❌ |
| SCSS | ✅ Built-in | ✅ Plugin | ✅ | ❌ |
| TypeScript | ✅ Built-in | ✅ Plugin | ✅ | Maybe (types) |
| Swift | ⚠️ Basic | ✅ Plugin | ❌ | Possibly |
| Figma JSON | ❌ | ❌ | ❌ | **Yes** |
| Android XML | ✅ Built-in | ❌ | ✅ | ❌ |

---

## Recommendations

### 1. Primary Stack

```
┌─────────────────────────────────────────────────────────┐
│                    TOKEN SOURCE                          │
│  ┌─────────────┐        ┌──────────────────────────┐   │
│  │ Figma       │───────▶│ tokens.json (DTCG format)│   │
│  │ (Tokens     │        │ stored in Git            │   │
│  │  Studio)    │        └──────────────────────────┘   │
│  └─────────────┘                    │                   │
└─────────────────────────────────────│───────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────┐
│                  COMPILATION (Style Dictionary)          │
│  ┌──────────────────────────────────────────────────┐  │
│  │ @tokens-studio/sd-transforms (DTCG transforms)   │  │
│  │ Custom transforms (if needed)                     │  │
│  │ Custom formats (Figma JSON, enhanced Swift)       │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
              ┌──────────┐    ┌──────────┐     ┌──────────┐
              │   CSS    │    │    TS    │     │  Swift   │
              │variables │    │  types   │     │ structs  │
              └──────────┘    └──────────┘     └──────────┘
```

### 2. What to Use Directly

| Tool | Use For |
|------|---------|
| **Style Dictionary v4** | Core compilation engine |
| **@tokens-studio/sd-transforms** | DTCG transforms, Tokens Studio compatibility |
| **Tokens Studio Figma plugin** | Visual token management (free tier) |

### 3. What to Wrap/Extend

| Gap | Solution |
|-----|----------|
| **Figma JSON export** | Write custom Style Dictionary format |
| **TypeScript types** | Extend JS format to generate `.d.ts` |
| **Enhanced Swift** | Either custom SD format OR use Terrazzo's Swift plugin |

### 4. What Requires Custom Code

| Need | Approach | Effort |
|------|----------|--------|
| **Figma JSON format** | Custom SD format (~100 LOC) | Low |
| **Token validation** | Use DTCG schema + custom rules | Low |
| **Multi-brand theming** | Use sd-transforms `permutateThemes` | Low |
| **CI/CD integration** | npm scripts + GitHub Actions | Low |

---

## Implementation Roadmap

### Phase 1: Foundation
1. Set up Style Dictionary v4 with sd-transforms
2. Define DTCG-compliant token structure
3. Implement CSS and TypeScript outputs

### Phase 2: Platform Outputs
4. Create custom Figma JSON format
5. Evaluate Swift output (SD custom vs Terrazzo)
6. Add Android XML if needed

### Phase 3: Workflow Integration
7. Integrate Tokens Studio Figma plugin
8. Set up Git sync workflow
9. Add CI/CD token compilation

---

## Cost Analysis

| Item | Cost |
|------|------|
| Style Dictionary | Free (Apache 2.0) |
| sd-transforms | Free (MIT) |
| Terrazzo | Free (MIT) |
| Tokens Studio Figma plugin (free tier) | Free |
| Tokens Studio Pro | $15/editor/month (optional) |
| Custom development | Internal time only |

**Total cost for full implementation: $0** (using free tiers and open-source)

---

## Conclusion

The design token tooling ecosystem is mature enough that we don't need to build from scratch. **Style Dictionary + sd-transforms** covers 90% of our needs. We'll write small custom formats for Figma JSON and potentially Swift, but the heavy lifting is already done.

**Key decision: Adopt DTCG format now.** It's the emerging standard, and both Style Dictionary and Terrazzo support it. This future-proofs our token system.

---

## References

- [Style Dictionary Documentation](https://styledictionary.com)
- [DTCG Specification](https://tr.designtokens.org/format/)
- [sd-transforms GitHub](https://github.com/tokens-studio/sd-transforms)
- [Terrazzo Documentation](https://terrazzo.app/docs)
- [Tokens Studio](https://tokens.studio)
