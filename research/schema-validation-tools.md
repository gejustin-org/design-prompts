# Schema Validation and YAML Tooling Research

> Research report for free/open-source tools for YAML schema validation, parsing, and IDE integration.

---

## Executive Summary

This report evaluates tools for validating YAML files against schemas and providing IDE autocomplete for spec authors. The recommended stack is:

| Purpose | Recommended Tool | Alternative |
|---------|------------------|-------------|
| **YAML Parser** | `yaml` (eemeli) | `js-yaml` |
| **Schema Validation** | Ajv + JSON Schema | Zod |
| **IDE Autocomplete** | YAML Language Server | Custom JSON Schema |
| **Schema Generation** | `zod-to-json-schema` | Manual JSON Schema |

---

## Tool Analysis

### 1. JSON Schema

**What it is:** A vocabulary for annotating and validating JSON documents (and by extension, YAML).

**License:** Open specification (no license required for use)

**Key Features:**
- Industry standard for schema definition
- Supported by most validators and IDEs
- Versions: draft-04, draft-06, draft-07, 2019-09, 2020-12
- Native support in VS Code via YAML Language Server

**How to validate YAML against JSON Schema:**
1. Parse YAML to JavaScript object (using `yaml` or `js-yaml`)
2. Validate the object using a JSON Schema validator (Ajv)

```typescript
import { parse } from 'yaml';
import Ajv from 'ajv';

const yaml = parse(yamlString);
const ajv = new Ajv();
const validate = ajv.compile(jsonSchema);
const valid = validate(yaml);
```

**Integration Complexity:** Low - well-documented, mature ecosystem

**Verdict:** ✅ **Foundation for all validation** - use JSON Schema as the schema format

---

### 2. Ajv (Another JSON Validator)

**What it is:** The fastest JSON Schema validator for Node.js and browsers.

**License:** MIT

**GitHub:** https://github.com/ajv-validator/ajv

**Key Features:**
- Supports JSON Schema draft-04/06/07/2019-09/2020-12
- Supports JSON Type Definition (RFC8927)
- **50% faster** than the next fastest validator
- Code generation for optimized validation functions
- Comprehensive error messages with paths
- OpenAPI extensions (discriminator, nullable)
- Default value assignment and type coercion
- Custom keywords and formats support
- i18n error messages via `ajv-i18n`

**Performance:**
- Fastest according to multiple benchmarks (json-schema-benchmark, jsck, z-schema)
- Generates optimized validation code for v8

**How we could use it:**
```typescript
import Ajv from 'ajv';
import { parse } from 'yaml';

const ajv = new Ajv({ 
  allErrors: true,      // Report all errors, not just first
  useDefaults: true,    // Fill in default values
  coerceTypes: true     // Coerce types when possible
});

const schema = {
  type: 'object',
  properties: {
    name: { type: 'string', minLength: 1 },
    version: { type: 'string', pattern: '^\\d+\\.\\d+\\.\\d+$' }
  },
  required: ['name', 'version']
};

const validate = ajv.compile(schema);
const data = parse(yamlContent);
const valid = validate(data);

if (!valid) {
  console.log(validate.errors);
  // [{ instancePath: '/name', message: 'must be string', ... }]
}
```

**Integration Complexity:** Low - straightforward API, excellent docs

**Verdict:** ✅ **Best choice for runtime validation** - fast, feature-rich, industry standard

---

### 3. Zod

**What it is:** TypeScript-first schema validation with static type inference.

**License:** MIT

**GitHub:** https://github.com/colinhacks/zod

**Key Features:**
- Zero external dependencies
- Tiny: 2kb core bundle (gzipped)
- **Built-in JSON Schema conversion** (as of v4)
- TypeScript type inference from schemas
- Immutable API
- Works with plain JS too
- Extensive ecosystem

**YAML Compatibility:**
- No native YAML support, but works perfectly with parsed YAML
- Parse YAML → validate with Zod → get typed result

**How we could use it:**
```typescript
import { z } from 'zod';
import { parse } from 'yaml';

// Define schema with full TypeScript inference
const SpecSchema = z.object({
  name: z.string().min(1),
  version: z.string().regex(/^\d+\.\d+\.\d+$/),
  description: z.string().optional(),
  actions: z.array(z.object({
    id: z.string(),
    type: z.enum(['http', 'grpc', 'cli'])
  }))
});

// Infer TypeScript type from schema
type Spec = z.infer<typeof SpecSchema>;

// Parse and validate
const data = parse(yamlContent);
const result = SpecSchema.safeParse(data);

if (!result.success) {
  console.log(result.error.issues);
} else {
  const spec: Spec = result.data; // Fully typed!
}
```

**For IDE Support (generate JSON Schema):**
```typescript
import { zodToJsonSchema } from 'zod-to-json-schema';
// Note: As of Zod v4, this is built-in!

const jsonSchema = zodToJsonSchema(SpecSchema, 'Spec');
// Write to .json file for YAML Language Server
```

**Integration Complexity:** Low-Medium - requires learning Zod API, but excellent DX

**Verdict:** ✅ **Best for TypeScript projects** - type inference is killer feature

---

### 4. Yup

**What it is:** Schema builder for runtime value parsing and validation.

**License:** MIT

**GitHub:** https://github.com/jquense/yup

**Key Features:**
- Concise, expressive schema interface
- TypeScript support with type inference
- Async validation support
- Extensible with custom methods
- Rich error details
- Compatible with Standard Schema specification

**Comparison to Zod:**
| Feature | Zod | Yup |
|---------|-----|-----|
| Type inference | Excellent | Good |
| Bundle size | 2kb | ~15kb |
| JSON Schema export | Built-in (v4) | Via plugin |
| Error messages | Good | Excellent |
| Async validation | Supported | Native focus |
| API style | Chainable | Chainable |
| Maturity | Newer, rapidly growing | Older, stable |

**How we could use it:**
```typescript
import { object, string, array, InferType } from 'yup';
import { parse } from 'yaml';

const specSchema = object({
  name: string().required(),
  version: string().matches(/^\d+\.\d+\.\d+$/).required(),
  description: string(),
});

type Spec = InferType<typeof specSchema>;

const data = parse(yamlContent);
const spec = await specSchema.validate(data);
```

**Integration Complexity:** Low - similar to Zod

**Verdict:** ⚠️ **Good alternative** - prefer Zod for new projects due to better JSON Schema support

---

### 5. yaml (npm package)

**What it is:** Definitive YAML library for JavaScript.

**License:** ISC (permissive, similar to MIT)

**GitHub:** https://github.com/eemeli/yaml

**Key Features:**
- Supports YAML 1.1 and 1.2
- **Passes all yaml-test-suite tests**
- Graceful error handling (parses as much as possible)
- **Preserves comments and blank lines** when editing
- Full AST access for programmatic manipulation
- No external dependencies
- Works in Node.js and browsers
- TypeScript support (3.9+)

**API Layers:**
1. **Parse & Stringify** - Simple JSON-like API
2. **Documents** - Full AST with comments, directives
3. **Lexer/Parser/Composer** - Low-level token access

**How we could use it:**
```typescript
import { parse, stringify, parseDocument } from 'yaml';

// Simple parsing (most common)
const data = parse(yamlString);

// Preserve comments when editing
const doc = parseDocument(yamlString);
doc.set('version', '2.0.0');
const output = doc.toString(); // Comments preserved!

// Error handling
const doc = parseDocument(invalidYaml);
if (doc.errors.length > 0) {
  doc.errors.forEach(e => console.error(e.message));
}
```

**Integration Complexity:** Very Low - drop-in JSON.parse replacement

**Verdict:** ✅ **Best YAML parser** - spec-compliant, comment-preserving, modern

---

### 6. js-yaml

**What it is:** YAML 1.2 parser/writer, started as PyYAML port.

**License:** MIT

**GitHub:** https://github.com/nodeca/js-yaml

**Key Features:**
- YAML 1.2 compliant
- Fast parsing
- CLI tool included
- Custom type support
- Mature and widely used

**Comparison to `yaml` package:**

| Feature | yaml (eemeli) | js-yaml |
|---------|---------------|---------|
| YAML 1.1 + 1.2 | ✅ | 1.2 only |
| Comment preservation | ✅ | ❌ |
| AST manipulation | ✅ | ❌ |
| Error recovery | ✅ | ❌ |
| Test suite pass | 100% | ~95% |
| Bundle size | ~65kb | ~50kb |
| Weekly downloads | ~30M | ~60M |

**How we could use it:**
```typescript
import yaml from 'js-yaml';
import fs from 'fs';

const doc = yaml.load(fs.readFileSync('config.yaml', 'utf8'));
const output = yaml.dump(doc);
```

**Integration Complexity:** Very Low

**Verdict:** ⚠️ **Good but dated** - use `yaml` package instead for new projects

---

### 7. YAML Language Server

**What it is:** Language Server Protocol implementation for YAML with JSON Schema support.

**License:** MIT

**GitHub:** https://github.com/redhat-developer/yaml-language-server

**Maintained by:** Red Hat

**Key Features:**
- JSON Schema 7 support
- **Auto-completion** from schema
- **Hover documentation** from schema descriptions
- **Validation** against JSON Schema
- Document outlining
- Supports custom tags
- Schema Store integration
- Kubernetes CRD support
- Works with any LSP-compatible editor

**Configuration Options:**
```json
{
  "yaml.schemas": {
    "./my-schema.json": "*.spec.yaml"
  },
  "yaml.validate": true,
  "yaml.completion": true,
  "yaml.hover": true,
  "yaml.schemaStore.enable": true
}
```

**Schema Association Methods:**
1. **Settings** - `yaml.schemas` configuration
2. **Modeline** - `# yaml-language-server: $schema=./schema.json`
3. **Schema Store** - automatic for known file patterns

**How we could use it for IDE autocomplete:**

1. Create JSON Schema for your spec format
2. Configure VS Code settings:
```json
{
  "yaml.schemas": {
    "./spec-schema.json": "*.spec.yaml"
  }
}
```

3. Or add modeline to YAML files:
```yaml
# yaml-language-server: $schema=./spec-schema.json
name: my-spec
version: 1.0.0
```

**Integration Complexity:** Low - just provide JSON Schema

**Verdict:** ✅ **Essential for IDE support** - de facto standard for YAML editing

---

### 8. Redocly CLI

**What it is:** All-in-one OpenAPI/AsyncAPI documentation and linting utility.

**License:** MIT (community edition)

**GitHub:** https://github.com/redocly/redocly-cli

**Key Features:**
- OpenAPI 2.0, 3.0, 3.1, 3.2 support
- AsyncAPI 2.6, 3.0 support
- Arazzo 1.0 support
- **Linting** with configurable rules
- **Bundling** split schemas
- **Documentation generation** (Redoc)
- **API contract testing**
- Custom decorators for transformation

**Relevant for our use case:**
- **Linting infrastructure** - could adapt rule system
- **Bundle command** - handling $ref resolution
- **Plugin architecture** - extensible validation

**How we could use it:**
- Probably **not directly applicable** unless building OpenAPI tools
- Could study their **rule/plugin architecture** for inspiration

**Integration Complexity:** Medium - primarily for OpenAPI

**Verdict:** ⚠️ **Not directly relevant** - but good reference for linting architecture

---

## Recommendations

### Best YAML Parser: `yaml` (eemeli)

**Why:**
- 100% spec compliance
- Comment preservation (critical for editing user files)
- Graceful error handling
- Modern, well-maintained
- Full AST access when needed

```bash
npm install yaml
```

### Best Schema Validation Approach: Zod + Ajv Hybrid

**For TypeScript projects:**

```typescript
// schemas/spec.ts - Single source of truth
import { z } from 'zod';

export const SpecSchema = z.object({
  name: z.string().min(1).describe('Unique identifier'),
  version: z.string().regex(/^\d+\.\d+\.\d+$/).describe('Semver version'),
  actions: z.array(ActionSchema).describe('List of actions')
});

export type Spec = z.infer<typeof SpecSchema>;

// Generate JSON Schema for IDE support
export const specJsonSchema = SpecSchema.toJsonSchema(); // Zod v4+
```

**Why this approach:**
1. **Single source of truth** - Zod schema defines validation AND types
2. **Type safety** - Full TypeScript inference
3. **IDE support** - Export to JSON Schema for YAML Language Server
4. **Runtime validation** - Zod handles parsing with good errors
5. **Flexibility** - Can also use Ajv for JSON Schema validation if needed

### IDE Autocomplete Strategy

**Recommended approach:**

1. **Define schemas in Zod** (or directly in JSON Schema)

2. **Generate JSON Schema file:**
```typescript
// scripts/generate-schema.ts
import { writeFileSync } from 'fs';
import { SpecSchema } from '../schemas/spec';
import { zodToJsonSchema } from 'zod-to-json-schema';

const schema = zodToJsonSchema(SpecSchema, {
  name: 'Spec',
  $refStrategy: 'none' // Inline all definitions for better IDE support
});

writeFileSync('schemas/spec.schema.json', JSON.stringify(schema, null, 2));
```

3. **Configure YAML Language Server:**
```json
// .vscode/settings.json
{
  "yaml.schemas": {
    "./schemas/spec.schema.json": ["*.spec.yaml", "specs/**/*.yaml"]
  }
}
```

4. **Add schema modeline to files (optional):**
```yaml
# yaml-language-server: $schema=../../schemas/spec.schema.json
name: example
version: 1.0.0
```

**Benefits:**
- ✅ Autocomplete for all fields
- ✅ Hover documentation from `describe()` calls
- ✅ Inline validation errors
- ✅ Works in VS Code, Neovim, any LSP client

---

## Integration Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Schema Definition                          │
│                                                                 │
│   Zod Schema (TypeScript)  ─────────► TypeScript Types          │
│         │                                                       │
│         ▼                                                       │
│   JSON Schema (generated)  ─────────► YAML Language Server      │
│         │                                  │                    │
│         │                                  ▼                    │
│         │                            IDE Autocomplete           │
│         │                            Hover Docs                 │
│         │                            Inline Validation          │
│         │                                                       │
└─────────┼───────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Runtime Validation                           │
│                                                                 │
│   YAML File ──► yaml.parse() ──► Zod.safeParse() ──► Typed Data│
│                                        │                        │
│                                        ▼                        │
│                                  Validation Errors              │
│                                  (with paths + messages)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation Checklist

- [ ] Install dependencies: `yaml`, `zod`, `zod-to-json-schema`
- [ ] Define Zod schemas with `.describe()` for docs
- [ ] Create script to generate JSON Schema files
- [ ] Add VS Code settings for YAML Language Server
- [ ] Document modeline usage for spec authors
- [ ] Set up validation in spec loading code
- [ ] Add helpful error messages with file paths

---

## Package Versions (as of research date)

| Package | Version | Weekly Downloads |
|---------|---------|------------------|
| yaml | 2.x | ~30M |
| ajv | 8.x | ~100M |
| zod | 3.x / 4.x | ~20M |
| yup | 1.x | ~8M |
| js-yaml | 4.x | ~60M |
| yaml-language-server | 1.x | N/A (editor plugin) |

---

## References

- [JSON Schema Specification](https://json-schema.org/)
- [Ajv Documentation](https://ajv.js.org/)
- [Zod Documentation](https://zod.dev/)
- [yaml Package Documentation](https://eemeli.org/yaml/)
- [YAML Language Server](https://github.com/redhat-developer/yaml-language-server)
- [Schema Store](https://www.schemastore.org/) - Public JSON Schema registry
