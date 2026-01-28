# dspec: Revised Plan v2

**Based on feedback from:** Product Designer, Product Engineer, Sr. Staff Engineer, CTO, Engineering Manager, AI Skeptic

---

## What Changed

| Original | Revised | Why |
|----------|---------|-----|
| 10-week timeline | 24-week timeline (6 months) | Realistic scoping |
| AI generation default | AI generation opt-in | Reduce risk, prove static first |
| Start with components | Start with tokens only | Validate incrementally |
| Build vs buy assumed | Explicit buy vs build analysis | Answer "why not Style Dictionary?" |
| Debugging hand-waved | Explicit debugging workflow | Operational clarity |
| Visual regression in Phase 5 | Visual regression in Phase 1 | Safety net from day 1 |
| Designer YAML authoring | Acknowledge gap, defer visual tooling | Honest about limitations |
| No kill criteria | Explicit go/no-go gates | Fail fast if not working |
| No business metrics | Success metrics defined | Prove ROI |

---

## Phase 0: Validation (2 weeks)

**Goal:** Answer critical questions before building anything.

### 0.1 Buy vs Build Analysis

**Evaluate existing tools:**

| Tool | What It Does | Evaluate |
|------|--------------|----------|
| Style Dictionary | Token compilation | Can it do what we need? |
| Tokens Studio | Figma ↔ tokens sync | Integration story? |
| Supernova | Full design system management | Cost vs build? |
| Specify | Token pipeline SaaS | Maintenance burden? |

**Output:** Written analysis with recommendation. If existing tool works, stop here.

### 0.2 Spec Expressiveness Gap Analysis

**Test:** Can YAML specs express existing Rosetta components?

```bash
# Pick 3 components of varying complexity
1. Button (simple)
2. Input (medium - states, validation)
3. Select (complex - compound component)
```

For each:
1. Export current component's full API surface
2. Write equivalent YAML spec (draft format)
3. Identify what spec CAN'T express
4. Estimate override percentage needed

**Kill criteria:** If >30% of component logic requires overrides, the spec format is too limited. Redesign or abort.

### 0.3 AI Generation Prototype

**Test:** Can AI reliably generate one component?

1. Write a Button spec
2. Craft an initial prompt
3. Generate Button 10 times
4. Measure:
   - Pass rate (TypeScript + lint)
   - Semantic correctness (manual review)
   - Consistency (diff between runs)

**Kill criteria:** If <80% pass rate or significant variance, AI generation is not ready.

### 0.4 Cost Projection

| Item | Estimate |
|------|----------|
| AI API (monthly, 50 components) | $X |
| Engineering (6 months) | $X |
| Ongoing maintenance (annual) | $X |

Compare to: Status quo cost (manual maintenance).

**Output:** ROI projection with payback period.

### Phase 0 Deliverables

- [ ] Buy vs build recommendation
- [ ] Gap analysis for 3 components
- [ ] AI prototype results
- [ ] Cost/ROI projection
- [ ] Go/No-Go decision

**Gate:** Leadership review before proceeding.

---

## Phase 1: Token Foundation (6 weeks)

**Goal:** Prove value with zero AI risk.

### Scope

- Token compilation only (colors, typography, spacing, radii, shadows)
- Static generation only (Handlebars templates)
- No AI in this phase
- Output: CSS variables, TypeScript constants, Figma tokens JSON

### 1.1 Project Setup (Week 1)

- Repository structure
- TypeScript, build system (tsup)
- CLI scaffold (Commander)
- CI/CD pipeline (GitHub Actions)

### 1.2 Token Schema & Parser (Week 2)

```yaml
# specs/tokens/colors.yaml
colors:
  background:
    primary:
      value: "#FFFFFF"
      description: Primary canvas
```

- JSON Schema for tokens
- YAML parser with validation
- Reference resolution (`$colors.background.primary`)

### 1.3 Static Generators (Week 3-4)

| Output | Template |
|--------|----------|
| `tokens.css` | CSS custom properties |
| `tokens.ts` | TypeScript constants + types |
| `tokens.json` | Raw JSON for tooling |
| `figma-tokens.json` | Figma Tokens plugin format |

### 1.4 Pipeline System (Week 5)

- Pipeline YAML format
- Step execution (static only)
- `dspec generate --target tokens`
- Watch mode for iteration

### 1.5 Publishing (Week 6)

- npm package generation
- Version management
- `@org/design-tokens` published

### Phase 1 Deliverables

- [ ] `dspec` CLI with token generation
- [ ] Published `@org/design-tokens` package
- [ ] Figma tokens synced
- [ ] Documentation

### Phase 1 Success Metrics

| Metric | Target |
|--------|--------|
| Token sync time (spec → all platforms) | < 5 minutes |
| Developer adoption | 2+ teams using tokens package |
| Bug rate | 0 (it's deterministic) |

**Gate:** Team retrospective. Is this useful? Continue?

---

## Phase 2: One Component, Static First (4 weeks)

**Goal:** Generate ONE component (Button) with minimal AI.

### Philosophy: Static-First, AI-Assist

```
Component = Static Scaffold + AI-Assisted Logic (opt-in)
```

Most of a component is boilerplate:
- Imports
- Type definitions
- CVA variant structure
- Exports
- Test scaffolding
- Story scaffolding

**Static templates handle 70%+.** AI only fills in the "creative" parts.

### 2.1 Component Schema (Week 1)

```yaml
# specs/components/button.yaml
name: Button
description: Primary interactive element

props:
  - name: variant
    type: enum
    values: [primary, secondary, tertiary, destructive]
    default: primary
  # ...

styling:
  # Token references, not raw values
  variants:
    primary:
      backgroundColor: $colors.interactive.primary
      # ...
```

### 2.2 Static Component Generator (Week 2)

Generate via templates (NO AI):
- `Button.tsx` structure
- Props interface
- CVA skeleton with token references
- `Button.test.tsx` scaffold
- `Button.stories.tsx` scaffold
- Barrel exports

**Output is functional but incomplete** — variant styling rules, test implementations, story examples are stubs.

### 2.3 AI-Assisted Completion (Week 3)

**Opt-in AI steps** to fill stubs:

```yaml
# In pipeline
- name: button-variants
  type: ai
  enabled: true  # Explicit opt-in
  prompt: prompts/react/cva-variants.md
  input: component.styling
  output: src/Button/_variants.ts
```

AI generates:
- CVA variant implementations (complex styling logic)
- Test case implementations
- Story example implementations

**Human reviews AI output before merge.**

### 2.4 Quality Gates (Week 4)

- TypeScript strict mode
- ESLint + Prettier
- Unit tests (generated + human-reviewed)
- **Visual regression (Chromatic/Percy)** ← Critical safety net
- Accessibility audit (axe-core)
- Bundle size check

### Phase 2 Deliverables

- [ ] Button component generated and published
- [ ] Visual regression baseline established
- [ ] Override system documented (ONE mechanism)
- [ ] Debugging workflow documented

### Phase 2 Success Metrics

| Metric | Target |
|--------|--------|
| AI generation pass rate | > 90% |
| Manual override rate | < 20% |
| Visual regression false positives | < 5% |
| Time from spec change → published | < 30 min |

**Gate:** Is AI-assisted generation good enough? If override rate > 30%, reconsider AI approach.

---

## Phase 3: Component Library (6 weeks)

**Goal:** Generate 10 core components.

### Component Selection

| Component | Complexity | AI Needed? |
|-----------|------------|------------|
| Button | Low | Minimal |
| Input | Medium | Yes (states) |
| Card | Low | Minimal |
| Badge/Tag | Low | Minimal |
| Dialog | Medium | Yes (a11y) |
| Select | High | Yes (compound) |
| Checkbox | Medium | Yes (states) |
| Switch | Medium | Yes (states) |
| Tooltip | Medium | Yes (positioning logic) |
| Table | High | Yes (features) |

### 3.1 Incremental Generation (Week 1-4)

- 2-3 components per week
- Each component follows Phase 2 workflow
- Visual regression for each
- Human review of AI output

### 3.2 Cross-Component Consistency (Week 5)

- Shared conventions document (injected in prompts)
- Custom ESLint rules for generated code patterns
- Component dependency graph
- Regeneration cascade testing

### 3.3 Publishing & Documentation (Week 6)

- `@org/rosetta-react` published
- Storybook deployed
- API documentation generated
- Migration guide for consumers

### Phase 3 Success Metrics

| Metric | Target |
|--------|--------|
| Components published | 10 |
| AI generation pass rate | > 90% |
| Manual override rate | < 15% |
| Bugs in generated code (30 days) | < 5 |
| Developer satisfaction (survey) | > 7/10 |

**Gate:** Is this better than hand-written? Survey team. If satisfaction < 6/10, pause and fix.

---

## Phase 4: Stabilization (4 weeks)

**Goal:** Production hardening before expanding scope.

### 4.1 Debugging Story

Document explicit workflow:

```
Something's wrong with Button
        ↓
Is it a runtime bug? → Check generated code, trace to spec
        ↓
Is it a generation bug? → Check AI output, review prompt
        ↓
Is it a spec bug? → Fix spec, regenerate
        ↓
Can't fix via spec? → Add override, document why
```

### 4.2 Rollback Strategy

- Generated code tagged with generation hash
- `dspec rollback --component Button --to <hash>`
- npm deprecation workflow for bad publishes

### 4.3 Override System (Finalized)

**ONE mechanism:** Override files with merge.

```
src/Button/
├── Button.gen.tsx       # Generated (DO NOT EDIT)
├── Button.override.tsx  # Human patches (optional)
├── Button.tsx           # Merges both, re-exports
└── index.ts
```

Override applies patches to generated code. Conflicts flagged in CI.

### 4.4 Monitoring

- Track: generation pass rate over time
- Track: override rate per component
- Track: prompt version → output quality correlation
- Alert: If pass rate drops below 85%

### 4.5 Local Development

- `dspec generate --local` uses cached AI responses
- Mock AI mode for offline development
- API key not required for static-only regeneration

### Phase 4 Deliverables

- [ ] Debugging playbook
- [ ] Rollback capability
- [ ] Override system finalized
- [ ] Monitoring dashboard
- [ ] Local dev mode

---

## Phase 5: Multi-Platform (Conditional, 6 weeks)

**Prerequisites:**
- Phase 3 metrics met
- Team satisfaction > 7/10
- Leadership approval to expand

### Option A: Figma Tokens (3 weeks)

- Extend token generator for Figma Tokens plugin
- Bidirectional sync exploration (Figma → spec)
- Designer workflow documentation

### Option B: Swift/iOS (6 weeks)

- Swift token generator
- SwiftUI component generator (static + AI)
- ONE component (Button) as proof
- iOS team adoption

**Do not attempt both simultaneously.** Pick based on business need.

---

## Phase 6: Designer Tooling (Future)

**Explicitly deferred.** Current plan is developer-centric.

### What Designers Need (Not in Scope)

- Figma plugin: Design → YAML spec export
- Visual spec editor (not YAML)
- Live preview while editing
- Visual diff for design review

**Honest acknowledgment:** This v1 requires designers to work through developers or learn YAML. Visual tooling is a separate project.

---

## Timeline Summary

| Phase | Duration | Cumulative |
|-------|----------|------------|
| 0: Validation | 2 weeks | 2 weeks |
| 1: Token Foundation | 6 weeks | 8 weeks |
| 2: One Component | 4 weeks | 12 weeks |
| 3: Component Library | 6 weeks | 18 weeks |
| 4: Stabilization | 4 weeks | 22 weeks |
| 5: Multi-Platform | 6 weeks | 28 weeks |

**MVP (Phases 0-3):** 18 weeks (~4.5 months)
**Production-Ready (Phases 0-4):** 22 weeks (~5.5 months)
**Multi-Platform (Full):** 28 weeks (~7 months)

---

## Staffing

| Role | FTE | Notes |
|------|-----|-------|
| Tech Lead | 1.0 | Owns architecture, AI quality |
| Senior Engineer | 1.0 | React generator, quality gates |
| Engineer | 0.5 | Documentation, testing |

**Total:** 2.5 FTE for 6 months

**Critical:** At least 2 people must deeply understand the system (bus factor).

---

## Kill Criteria

**Stop the project if:**

| Phase | Criteria |
|-------|----------|
| 0 | Buy analysis shows existing tool works |
| 0 | Gap analysis shows >30% override needed |
| 0 | AI prototype <80% pass rate |
| 2 | Override rate >30% for Button |
| 3 | Developer satisfaction <6/10 |
| 3 | Bug rate 2x hand-written baseline |

**Sunk cost is not a reason to continue.** Fail fast.

---

## Success Metrics (Summary)

| Metric | Phase 1 | Phase 3 | Phase 4 |
|--------|---------|---------|---------|
| Generation pass rate | 100% (static) | >90% | >95% |
| Manual override rate | 0% | <15% | <10% |
| Time to publish | <5 min | <30 min | <15 min |
| Developer satisfaction | — | >7/10 | >8/10 |
| Bugs (30 days) | 0 | <5 | <2 |

---

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| AI model deprecation | Medium | High | Abstract provider, pin versions, snapshot tests |
| Spec can't express needs | Medium | High | Phase 0 gap analysis, flexible override system |
| Team rejects tool | Medium | Medium | Involve team early, keep escape hatches |
| Debugging nightmare | Medium | High | Explicit debugging workflow, provenance comments |
| Cost overrun (AI API) | Low | Medium | Caching, static-first, cost monitoring |
| Timeline slip | High | Medium | Buffer built into estimates, clear gates |

---

## Open Questions (To Resolve in Phase 0)

1. **Style Dictionary compatibility** — Can we use it for token compilation?
2. **Figma direction** — Export from Figma or only to Figma?
3. **Compound components** — How do we spec `<Select><SelectItem/></Select>`?
4. **Responsive styling** — How do specs express breakpoints?
5. **Theme variants** — Light/dark mode in specs?

---

## What This Plan Does NOT Include

1. **Designer visual tooling** — Deferred to future project
2. **Kotlin/Android** — Only after Swift proven
3. **Full Handshake migration** — Only after production stable
4. **AI chat/interactive mode** — Nice-to-have, not MVP
5. **Pattern/workflow generation** — Future scope

---

## Approval Checklist

Before proceeding to Phase 1:

- [ ] Buy vs build analysis complete
- [ ] Gap analysis demonstrates viability
- [ ] AI prototype meets threshold
- [ ] Cost projection approved
- [ ] Team consulted and supportive
- [ ] 2.5 FTE allocated for 6 months
- [ ] Kill criteria accepted by leadership
