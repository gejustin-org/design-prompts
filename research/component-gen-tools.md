# Component Code Generation Tools Research

> Research report for dspec: Finding the best open-source tools for generating React components from design specs.

## Executive Summary

After researching scaffolding tools, template engines, AST manipulation libraries, and CLI approaches used by major projects, here are the key findings:

| Approach | Best For | Recommended Tool |
|----------|----------|------------------|
| Template Engine | Simple token replacement | **Handlebars** |
| Scaffolding | Multi-file generation with prompts | **Hygen** |
| AST Manipulation | Code modification/analysis | **ts-morph** |
| CLI Distribution | Component distribution | **shadcn-style registry** |

**Bottom Line:** For dspec, a hybrid approach combining **Handlebars templates** + **ts-morph for validation/AST ops** + a **custom CLI inspired by shadcn** gives us the best balance of flexibility and maintainability.

---

## Tool Comparison Matrix

| Tool | License | Active? | Learning Curve | dspec Fit |
|------|---------|---------|----------------|-----------|
| Plop.js | MIT | ✅ Yes | Low | ⭐⭐⭐⭐ |
| Hygen | MIT | ✅ Yes | Low | ⭐⭐⭐⭐⭐ |
| Yeoman | BSD | ⚠️ Maintained | Medium | ⭐⭐ |
| ts-morph | MIT | ✅ Yes | Medium-High | ⭐⭐⭐⭐ |
| @babel/generator | MIT | ✅ Yes | High | ⭐⭐ |
| Handlebars | MIT | ✅ Yes | Low | ⭐⭐⭐⭐⭐ |
| EJS | Apache-2.0 | ✅ Yes | Low | ⭐⭐⭐⭐ |
| Mustache | MIT | ✅ Yes | Very Low | ⭐⭐⭐ |
| Recast | MIT | ✅ Yes | Medium | ⭐⭐⭐ |

---

## 1. Plop.js

**License:** MIT  
**GitHub:** https://github.com/plopjs/plop  
**Latest:** v4.0.5

### What It Does Well
- **Inquirer integration** — Built on inquirer.js for interactive prompts
- **Handlebars native** — Templates use Handlebars out of the box
- **TypeScript support** — First-class TS plopfiles (Node 22+)
- **Action types** — `add`, `modify`, `addMany`, custom actions
- **Injection** — Can inject code into existing files
- **CLI bypass** — Can pass answers via CLI args

### Limitations
- **Local focus** — Designed for project-local generators, not distribution
- **No registry** — No built-in way to share/distribute generators
- **Simple templating** — Handlebars can be limiting for complex logic

### How We Could Use It
```javascript
// plopfile.ts
export default function (plop) {
  plop.setGenerator("component", {
    description: "Generate a dspec component",
    prompts: [
      { type: "input", name: "name", message: "Component name:" },
      { type: "list", name: "style", choices: ["neo-brutalism", "minimal", "saas"] }
    ],
    actions: [
      {
        type: "add",
        path: "src/components/{{pascalCase name}}/{{pascalCase name}}.tsx",
        templateFile: "templates/component.hbs"
      },
      {
        type: "add", 
        path: "src/components/{{pascalCase name}}/{{pascalCase name}}.stories.tsx",
        templateFile: "templates/story.hbs"
      }
    ]
  });
}
```

### Verdict for dspec
**Good choice if:** We want simple local scaffolding with prompts  
**Not ideal if:** We need a distributed registry or complex transformations

---

## 2. Hygen

**License:** MIT  
**GitHub:** https://github.com/jondot/hygen  
**Used by:** Wix, Airbnb, Mercedes-Benz, Accenture

### What It Does Well
- **Zero config** — Templates in `_templates/`, just works
- **EJS templates** — Full JavaScript in templates
- **Frontmatter metadata** — `to:`, `inject:`, `skip_if:` in templates
- **Injection support** — Insert into existing files with regex patterns
- **Shell commands** — Can run shell commands as actions
- **Case changing** — Built-in `h.changeCase.*` helpers
- **Subcommands** — `hygen component new:styled` targets specific templates
- **Embeddable** — Can be used programmatically

### Limitations
- **EJS only** — No Handlebars (though EJS is more powerful)
- **Flat prompts** — No complex conditional flows like Plop
- **No registry** — Still project-local focused

### Template Example
```ejs
---
to: src/components/<%= name %>/<%= name %>.tsx
---
import { cva, type VariantProps } from "class-variance-authority";
import { cn } from "@/lib/utils";

const <%= h.changeCase.camelCase(name) %>Variants = cva(
  "<%= baseClasses %>",
  {
    variants: {
      variant: {
        default: "<%= variants.default %>",
        <%_ for (const [key, value] of Object.entries(variants)) { _%>
        <%= key %>: "<%= value %>",
        <%_ } _%>
      },
    },
    defaultVariants: {
      variant: "default",
    },
  }
);

export interface <%= name %>Props
  extends React.HTMLAttributes<HTMLDivElement>,
    VariantProps<typeof <%= h.changeCase.camelCase(name) %>Variants> {}

export function <%= name %>({ className, variant, ...props }: <%= name %>Props) {
  return (
    <div
      className={cn(<%= h.changeCase.camelCase(name) %>Variants({ variant }), className)}
      {...props}
    />
  );
}
```

### Programmatic Usage
```javascript
const { runner } = require('hygen');

await runner(['component', 'new', 'Button'], {
  templates: path.join(__dirname, '_templates'),
  cwd: process.cwd(),
  createPrompter: () => require('enquirer'),
});
```

### Verdict for dspec
**Excellent choice** — Best balance of simplicity and power. EJS templates handle complex CVA generation well.

---

## 3. Yeoman

**License:** BSD  
**GitHub:** https://github.com/yeoman/generator  
**Status:** Still maintained but less popular than before

### What It Does Well
- **Full lifecycle** — initializing, prompting, configuring, writing, conflicts, install, end
- **Composability** — Generators can compose other generators
- **File utilities** — `this.fs.copyTpl()`, `this.fs.extendJSON()`
- **Conflict resolution** — Built-in file conflict handling
- **Sub-generators** — `yo myapp:component`

### Limitations
- **Heavy** — Lots of boilerplate for simple tasks
- **Outdated patterns** — Class-based API feels dated
- **Slow startup** — Heavier than Plop/Hygen
- **Overkill** — More suited for full project scaffolds than component generation

### Verdict for dspec
**Not recommended** — Too heavy for our use case. Use Plop or Hygen instead.

---

## 4. ts-morph

**License:** MIT  
**GitHub:** https://github.com/dsherret/ts-morph  
**Docs:** https://ts-morph.com

### What It Does Well
- **TypeScript AST manipulation** — Full access to TS compiler API
- **Navigation** — Easy traversal of syntax trees
- **Modification** — Add/remove/modify statements, classes, interfaces
- **Code generation** — Generate TS/JS code from AST
- **Formatting** — Auto-format generated code
- **Type checking** — Access to TypeScript's type system

### Key Operations
```typescript
import { Project, VariableDeclarationKind } from "ts-morph";

const project = new Project();
const sourceFile = project.createSourceFile("button.tsx", "");

// Add imports
sourceFile.addImportDeclaration({
  namedImports: ["cva", "VariantProps"],
  moduleSpecifier: "class-variance-authority",
});

// Add variable (CVA config)
sourceFile.addVariableStatement({
  declarationKind: VariableDeclarationKind.Const,
  declarations: [{
    name: "buttonVariants",
    initializer: `cva("inline-flex items-center", {
      variants: {
        variant: {
          default: "bg-primary text-primary-foreground",
          destructive: "bg-destructive text-destructive-foreground",
        },
      },
    })`,
  }],
});

// Add function
sourceFile.addFunction({
  name: "Button",
  isExported: true,
  parameters: [{ name: "props", type: "ButtonProps" }],
  statements: "return <button {...props} />;",
});

// Save
await project.save();
```

### Limitations
- **Learning curve** — Need to understand TypeScript AST
- **Verbose** — More code than template-based approaches
- **Overkill for simple gen** — Better for modification than creation

### Use Cases for dspec
1. **Validation** — Verify generated code is valid TypeScript
2. **Modification** — Inject into existing component files
3. **Analysis** — Parse design system to extract patterns
4. **Type generation** — Generate TypeScript interfaces from specs

### Verdict for dspec
**Use as complement** — Not for primary generation, but excellent for validation and modification tasks.

---

## 5. @babel/generator

**License:** MIT  
**Docs:** https://babeljs.io/docs/babel-generator

### What It Does Well
- **AST to code** — Converts Babel AST back to source code
- **Source maps** — Automatic source map generation
- **JSX support** — Handles JSX/TSX syntax
- **Formatting options** — Minification, comments, whitespace

### Basic Usage
```javascript
import { parse } from "@babel/parser";
import { generate } from "@babel/generator";

const ast = parse(code, { plugins: ["jsx", "typescript"] });
// ... modify AST ...
const output = generate(ast).code;
```

### Limitations
- **Low-level** — Requires understanding Babel AST structure
- **No TypeScript types** — Works with syntax, not types
- **Complex for generation** — Better for transformation than creation

### Verdict for dspec
**Not recommended as primary tool** — ts-morph is better for TypeScript. Babel is more for transpilation pipelines.

---

## 6. Template Engines Comparison

### Handlebars

**License:** MIT  
**Best for:** Logic-less templates with helpers

```handlebars
{{!-- component.hbs --}}
import { cva, type VariantProps } from "class-variance-authority";

const {{camelCase name}}Variants = cva(
  "{{baseClasses}}",
  {
    variants: {
      {{#each variants}}
      {{@key}}: {
        {{#each this}}
        {{@key}}: "{{this}}",
        {{/each}}
      },
      {{/each}}
    },
  }
);
```

**Pros:**
- Clean syntax, easy to read
- Custom helpers for transforms (`{{pascalCase name}}`)
- Partials for reuse
- Used by Plop.js natively

**Cons:**
- No inline JavaScript logic
- Verbose for complex conditionals

### EJS

**License:** Apache-2.0  
**Best for:** Templates needing JavaScript logic

```ejs
<%# component.ejs %>
import { cva, type VariantProps } from "class-variance-authority";

const <%= toCamelCase(name) %>Variants = cva(
  "<%= baseClasses %>",
  {
    variants: {
      <% for (const [variantName, options] of Object.entries(variants)) { %>
      <%= variantName %>: {
        <% for (const [optionName, classes] of Object.entries(options)) { %>
        <%= optionName %>: "<%= classes %>",
        <% } %>
      },
      <% } %>
    },
  }
);
```

**Pros:**
- Full JavaScript in templates
- Familiar syntax (like PHP/ERB)
- Easy debugging (JS exceptions)

**Cons:**
- Can become messy with too much logic
- Less readable than Handlebars

### Mustache

**License:** MIT  
**Best for:** Extremely simple templates

```mustache
{{! component.mustache }}
import { cva } from "class-variance-authority";

const {{name}}Variants = cva("{{baseClasses}}", {
  variants: {
    {{#variants}}
    {{name}}: "{{classes}}",
    {{/variants}}
  },
});
```

**Pros:**
- Minimal syntax
- Logic-less (forces clean data)
- Works in any language

**Cons:**
- Too limited for complex generation
- No helpers or transforms

### Recommendation for dspec

**Use Handlebars** for these reasons:
1. Clean, readable templates
2. Custom helpers for case transforms (`pascalCase`, `kebabCase`)
3. Partials for component sections (variants, props, etc.)
4. Native integration with Plop.js
5. Can extend with custom helpers for dspec-specific logic

---

## 7. CVA (class-variance-authority)

**License:** Apache-2.0  
**GitHub:** https://github.com/joe-bell/cva  
**Docs:** https://cva.style/docs

### How Others Generate CVA Code

CVA code follows a predictable pattern:

```typescript
const componentVariants = cva(
  "base-classes here",  // Base styles
  {
    variants: {
      variant: {
        default: "variant-default-classes",
        secondary: "variant-secondary-classes",
      },
      size: {
        sm: "size-sm-classes",
        md: "size-md-classes", 
        lg: "size-lg-classes",
      },
    },
    compoundVariants: [
      { variant: "default", size: "lg", class: "compound-classes" },
    ],
    defaultVariants: {
      variant: "default",
      size: "md",
    },
  }
);
```

### Generation Approach for dspec

Since CVA has a consistent structure, we can:

1. **Define variant schema:**
```typescript
interface CVASpec {
  base: string[];
  variants: Record<string, Record<string, string[]>>;
  compoundVariants?: CompoundVariant[];
  defaultVariants: Record<string, string>;
}
```

2. **Transform to CVA code:**
```typescript
function generateCVA(spec: CVASpec): string {
  return `cva(
  "${spec.base.join(' ')}",
  {
    variants: {
${Object.entries(spec.variants)
  .map(([name, options]) => 
    `      ${name}: {\n${Object.entries(options)
      .map(([opt, classes]) => `        ${opt}: "${classes.join(' ')}"`)
      .join(',\n')}\n      }`)
  .join(',\n')}
    },
    defaultVariants: {
${Object.entries(spec.defaultVariants)
  .map(([k, v]) => `      ${k}: "${v}"`)
  .join(',\n')}
    },
  }
)`;
}
```

---

## 8. shadcn/ui CLI Approach

**License:** MIT  
**GitHub:** https://github.com/shadcn-ui/ui

### How It Works

shadcn/ui pioneered a **registry-based distribution model**:

1. **No npm package** — Components are copied into your project
2. **Registry JSON** — Components defined in a registry with metadata
3. **CLI adds components** — `npx shadcn@latest add button`
4. **Local ownership** — You own and can modify the code

### CLI Architecture

```
shadcn/
├── src/
│   ├── commands/
│   │   ├── init.ts      # Initialize project config
│   │   ├── add.ts       # Add components
│   │   ├── build.ts     # Build registry JSON
│   │   └── ...
│   ├── utils/
│   │   ├── registry/    # Registry fetching/parsing
│   │   ├── transformers/ # Code transforms
│   │   └── ...
│   └── index.ts
└── package.json
```

### Key Patterns

1. **Registry schema:**
```typescript
interface RegistryItem {
  name: string;
  type: "components:ui" | "components:example";
  dependencies?: string[];
  devDependencies?: string[];
  registryDependencies?: string[];
  files: {
    path: string;
    content: string;
    type: "registry:ui" | "registry:lib";
  }[];
  tailwind?: { config?: object };
  cssVars?: Record<string, Record<string, string>>;
}
```

2. **Component resolution:**
   - Fetches from registry URL
   - Resolves dependencies (other components)
   - Transforms import paths for target project
   - Writes files to configured directory

3. **Configuration:**
```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "default",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.js",
    "css": "app/globals.css",
    "baseColor": "slate",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils"
  }
}
```

### What We Can Learn

1. **Registry model** — Perfect for distributing design system components
2. **Local-first** — Users own their code, can customize
3. **Dependency resolution** — Components can depend on other components
4. **Transform pipeline** — Adapt code to user's project structure
5. **CSS variables** — Style theming through CSS custom properties

---

## 9. AI-Assisted Code Gen (Free Options)

### Available Free Tools

1. **Ollama + Local Models**
   - Run LLMs locally
   - No API costs
   - Models: CodeLlama, DeepSeek Coder, StarCoder

2. **Codeium**
   - Free tier available
   - IDE extensions
   - Not programmatically accessible

3. **TabNine Basic**
   - Local code completion
   - Free tier limited

### Practical AI Integration

For dspec, AI could help with:
- Generating Tailwind class combinations
- Suggesting variant names
- Creating component descriptions
- Generating Storybook stories

**Approach:**
```typescript
// Using local Ollama
async function suggestVariants(component: string, style: string) {
  const response = await fetch('http://localhost:11434/api/generate', {
    method: 'POST',
    body: JSON.stringify({
      model: 'codellama',
      prompt: `Generate CVA variants for a ${component} in ${style} style...`,
    }),
  });
  return response.json();
}
```

### Verdict
AI is supplementary, not core. Focus on deterministic template generation first.

---

## Recommendations for dspec

### 1. Best Template Engine: **Handlebars**

**Why:**
- Clean syntax for component templates
- Custom helpers for case transforms
- Partials for reusable sections
- Works with Plop.js if we want prompts

### 2. Best Scaffolding Approach: **Hygen-inspired Custom CLI**

**Why:**
- EJS templates are powerful enough for CVA generation
- Frontmatter pattern is elegant
- Can embed in our own CLI
- Project-local templates for customization

### 3. Architecture Recommendation

```
dspec/
├── cli/
│   ├── commands/
│   │   ├── init.ts         # Initialize dspec in project
│   │   ├── generate.ts     # Generate from spec
│   │   └── add.ts          # Add from registry
│   └── index.ts
├── registry/
│   ├── schemas/            # JSON schemas for specs
│   └── components/         # Pre-built component specs
├── templates/
│   ├── component.hbs       # Main component template
│   ├── variants.hbs        # CVA variants partial
│   ├── types.hbs           # TypeScript types partial
│   └── story.hbs           # Storybook story template
├── lib/
│   ├── generator.ts        # Template engine wrapper
│   ├── transformer.ts      # Code transforms (ts-morph)
│   └── validator.ts        # Spec validation
└── package.json
```

### 4. What We Can Reuse vs Build

| Component | Reuse | Build |
|-----------|-------|-------|
| Template engine | Handlebars | Custom helpers |
| AST manipulation | ts-morph | dspec-specific transforms |
| CLI framework | Commander.js | Commands & prompts |
| Schema validation | Zod | dspec schemas |
| Code formatting | Prettier | Config only |
| File operations | fs-extra | - |
| Case transforms | change-case | - |

### 5. Phased Implementation

**Phase 1: Core Generation**
- Handlebars templates for components
- CLI with `generate` command
- Local spec files

**Phase 2: Registry**
- shadcn-style registry
- `add` command for pre-built components
- Dependency resolution

**Phase 3: Intelligence**
- ts-morph for code analysis
- Style detection from existing components
- Validation and suggestions

---

## Appendix: License Summary

| Tool | License | Commercial OK? |
|------|---------|----------------|
| Handlebars | MIT | ✅ Yes |
| Hygen | MIT | ✅ Yes |
| Plop.js | MIT | ✅ Yes |
| ts-morph | MIT | ✅ Yes |
| EJS | Apache-2.0 | ✅ Yes |
| Mustache | MIT | ✅ Yes |
| CVA | Apache-2.0 | ✅ Yes |
| shadcn/ui | MIT | ✅ Yes |
| Recast | MIT | ✅ Yes |
| @babel/generator | MIT | ✅ Yes |
| Yeoman | BSD | ✅ Yes |

All tools are permissively licensed and safe for commercial use.
