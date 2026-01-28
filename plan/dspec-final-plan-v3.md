# dspec: Final Plan v3

**Incorporates feedback from:** CTO, Product Designer, Product Engineer, Sr. Staff Engineer, Engineering Manager, AI Skeptic

---

## Executive Summary

**What:** A tool that compiles YAML design specs into publishable component libraries.

**Approach:** Static-first, AI-optional. Prove value with deterministic token compilation before introducing AI for component generation.

**Timeline:** 24 weeks (6 months) to production-ready. Phase 0 validation in 2 weeks.

**Investment:** 3.0 FTE for 6 months.

**Kill criteria:** Explicit gates at each phase. "AI Not Worth It" is an acceptable outcome.

---

## Changes from v2

| v2 | v3 | Source |
|----|----|----|
| 2.5 FTE | **3.0 FTE** | EM: bus factor |
| 80% AI pass rate threshold | **95%** | AI Skeptic |
| Compound components "open question" | **Resolved in Phase 0** | Staff, Engineer |
| No eject command | **`dspec eject` added** | Engineer |
| No IR specification | **Required before Phase 1** | Staff |
| Baseline metrics assumed | **Measured before Phase 0** | CTO |
| Designer tooling "future" | **Committed follow-on (3 months post-Phase 3)** | Designer |
| No static-only decision point | **Explicit gate after Phase 2** | AI Skeptic |
| No incident response | **Playbook required before Phase 3** | EM |

---

## Pre-Phase 0: Baseline Establishment (1 week)

**Before Phase 0 kicks off, measure:**

| Metric | How | Owner |
|--------|-----|-------|
| Current component bug rate | Last 90 days of Rosetta bugs | Tech Lead |
| Current time-to-publish | Time from PR merge to npm | Tech Lead |
| Current developer satisfaction | Survey (1-10 scale) | EM |
| Designer workflow pain points | Interview 2-3 designers | Tech Lead |
| Manual maintenance hours/month | Engineering time tracking | EM |

**Deliverables:**
- [ ] Baseline metrics document
- [ ] Designer stakeholder interview notes
- [ ] ROI calculation methodology (formula, not just template)

---

## Phase 0: Validation (2 weeks)

### 0.1 Buy vs Build Analysis

**Evaluate:**

| Tool | Evaluate For | Kill If |
|------|--------------|---------|
| Style Dictionary | Token compilation | Covers 90%+ of token needs |
| Tokens Studio | Figma ↔ token sync | Full bidirectional works |
| Supernova | Full design system | TCO < custom build |
| Specify | Token pipeline SaaS | Maintenance-free option |

**Output:** Written analysis with **Total Cost of Ownership**, not just feature comparison.

**Reviewed by:** Someone outside the project team.

### 0.2 Spec Expressiveness Gap Analysis

**Test with 3 components:**
1. Button (simple)
2. Input (medium — states, validation)
3. Select (complex — **compound component**)

For each:
1. Export current API surface
2. Write equivalent YAML spec
3. Document what spec CAN'T express
4. Calculate override percentage needed

**Compound component hypothesis:** Propose a spec format for `<Select><SelectItem/></Select>` patterns. Test against real Select component.

**Kill criteria:**
- If >30% requires overrides → spec format insufficient
- If compound component pattern fails → redesign schema or abort

### 0.3 AI Generation Prototype

**Test Button generation:**
1. Write Button spec
2. Craft initial prompt
3. Generate **20 times** (not 10)
4. Measure:
   - Pass rate (TypeScript + lint + tests run)
   - Semantic correctness (manual review)
   - Cross-run diff (determinism)

**Reproducibility strategy:** Document approach:
- Response caching with content hash
- Pinned model version in generation metadata
- CI uses cached responses, regeneration requires explicit flag

**Kill criteria:** If <90% pass rate (raised from 80%), AI generation not ready.

### 0.4 Cost Projection (Real Numbers)

| Item | Estimate | Calculation |
|------|----------|-------------|
| AI API (monthly, 50 components) | $____ | [tokens × price × frequency] |
| Engineering (6 months, 3 FTE) | $____ | [salary + burden] |
| Ongoing maintenance (annual) | $____ | [% of FTE] |
| Prompt maintenance (annual) | $____ | [hours × rate] |
| **Total Year 1** | $____ | |

**Baseline comparison:**
| Current manual maintenance | $____ /year | [hours × rate] |

**ROI:** If Year 1 cost > 2x current maintenance, reconsider.

### Phase 0 Deliverables

- [ ] Buy vs build recommendation with TCO
- [ ] Gap analysis for 3 components (including compound)
- [ ] Compound component spec proposal
- [ ] AI prototype results (20 runs)
- [ ] AI reproducibility strategy document
- [ ] Cost projection with real numbers
- [ ] Designer interview summary
- [ ] Go/No-Go decision

**Gate:** Leadership + designer stakeholder review.

---

## Phase 1: Token Foundation (6 weeks)

### Scope
- Token compilation only
- **Zero AI** — pure static generation
- Output: CSS, TypeScript, Figma tokens JSON

### 1.1 Project Setup (Week 1)
- Repository, TypeScript, build system
- CLI scaffold
- CI/CD pipeline

### 1.2 Schema & Parser (Week 2)
- JSON Schema for tokens
- YAML parser with validation
- Reference resolution

### 1.3 IR Specification (Week 2-3)

**Required before proceeding:**

```typescript
// Documented IR types
interface TokenIR {
  colors: Record<string, ColorToken>;
  typography: Record<string, TypographyToken>;
  spacing: Record<string, number>;
  // ... complete type definitions
}

interface ParseResult {
  tokens: TokenIR;
  errors: ValidationError[];
  warnings: Warning[];
}

interface GeneratorInput {
  ir: TokenIR;
  config: GeneratorConfig;
}
```

**Deliverable:** `docs/architecture/ir-specification.md` (2-3 pages)

### 1.4 Static Generators (Week 3-4)
- `tokens.css` — CSS custom properties
- `tokens.ts` — TypeScript constants + types
- `figma-tokens.json` — Figma Tokens plugin format

### 1.5 Visual Token Preview (Week 4)
- Simple HTML page rendering all tokens
- Auto-generated from token specs
- Designers can view without CLI

### 1.6 Pipeline System (Week 5)
- Pipeline YAML format
- Step execution (static only)
- Watch mode

### 1.7 Publishing (Week 6)
- npm package generation
- `@org/design-tokens` published

### Phase 1 Deliverables
- [ ] `dspec` CLI with token generation
- [ ] IR specification document
- [ ] Visual token preview page
- [ ] Published `@org/design-tokens`
- [ ] 2+ teams using package

### Phase 1 Success Metrics

| Metric | Target |
|--------|--------|
| Token sync time | < 5 minutes |
| Adoption | 2+ teams |
| Bug rate | 0 (deterministic) |

**Gate:** Team retrospective. Designer feedback on token preview.

---

## Phase 2: First Component — Static Then AI (5 weeks)

### Philosophy: Static-First, AI-Assist

**Week 1-2:** Static scaffold only (no AI)
**Week 3:** Add AI-assisted completion (opt-in)
**Week 4:** Quality gates + visual regression
**Week 5:** Evaluate: Is AI worth it?

### 2.1 Component Schema (Week 1)

```yaml
name: Button
props:
  - name: variant
    type: enum
    values: [primary, secondary, tertiary, destructive]
styling:
  variants:
    primary:
      backgroundColor: $colors.interactive.primary
```

**Compound component format:** Use proposal from Phase 0.

### 2.2 Static Scaffold Generator (Week 2)

Generate via templates (NO AI):
- `Button.tsx` structure (imports, interface, basic component)
- `Button.test.tsx` scaffold (describe blocks, test shells)
- `Button.stories.tsx` scaffold (meta, story shells)
- Type definitions
- Barrel exports

**Output:** Functional but incomplete. Variant styling and test implementations are stubs.

**Deliverable:** Working static-only Button.

### 2.3 Static-Only Decision Point (End of Week 2)

**Explicit evaluation:**

> "The static scaffold provides ~70% of the component. Is the remaining 30% worth the AI complexity?"

**Options:**
1. **Proceed with AI** — Remaining 30% justifies complexity
2. **Ship static-only** — Engineers fill stubs manually, simpler system
3. **Hybrid** — AI for some components, manual for complex ones

**This is a real decision, not a formality.** Document rationale.

### 2.4 AI-Assisted Completion (Week 3, if proceeding)

Opt-in AI steps:
- CVA variant implementations
- Test case implementations
- Story example implementations

**Generation provenance in every file:**
```typescript
/**
 * Generated by dspec v1.2.0
 * Spec: components/button.yaml@abc123
 * Prompt: react/cva-variants@2.1.0
 * Model: claude-sonnet-4-20250514
 * Temperature: 0
 * Generation hash: def456
 * Timestamp: 2025-03-15T10:30:00Z
 */
```

### 2.5 Quality Gates (Week 4)

- TypeScript strict
- ESLint + Prettier
- Unit tests
- **Visual regression (Chromatic/Percy)** — baseline established
- Accessibility audit (axe-core)
- Bundle size check

### 2.6 `dspec eject` Command (Week 4)

```bash
dspec eject Button
```

Produces standalone component:
- No dspec imports
- No generation metadata (optional flag to keep)
- Full ownership transferred to consumer
- Component removed from generation pipeline

### Phase 2 Deliverables
- [ ] Button component (static or AI-assisted)
- [ ] Visual regression baseline
- [ ] `dspec eject` working
- [ ] Override system documented
- [ ] Static-only decision documented

### Phase 2 Success Metrics

| Metric | Target |
|--------|--------|
| AI pass rate (if used) | > 95% |
| Manual override rate | < 20% |
| Visual regression accuracy | < 5% false positives |
| Time: spec change → published | < 30 min |

**Kill criteria:** 
- If AI pass rate < 90% → Revert to static-only
- If override rate > 30% → Spec format insufficient

**Gate:** Is AI providing enough value? Explicit decision.

---

## Phase 3: Component Library (6 weeks)

**Prerequisite:** Phase 2 AI decision made. If static-only chosen, this phase is templates only.

### Component Selection (10 total)

| Component | Complexity | AI Needed? |
|-----------|------------|------------|
| Button | Low | Per Phase 2 decision |
| Input | Medium | States, validation |
| Card | Low | Minimal |
| Badge/Tag | Low | Minimal |
| Dialog | Medium | Accessibility |
| Select | High | Compound |
| Checkbox | Medium | States |
| Switch | Medium | States |
| Tooltip | Medium | Positioning |
| Table | High | Features |

### 3.1 Incremental Generation (Week 1-4)
- 2-3 components per week
- Human review of all AI output
- Visual regression for each

### 3.2 Cross-Component Consistency (Week 5)
- Shared conventions document
- Custom ESLint rules
- Dependency graph
- Regeneration cascade testing

### 3.3 Fast Regeneration Paths (Week 5)

```bash
# Regenerate static only (< 1 second)
dspec generate --no-ai

# Regenerate specific component
dspec generate --component Button

# Use cached AI responses
dspec generate --cached
```

### 3.4 Publishing & Documentation (Week 6)
- `@org/rosetta-react` published
- Storybook deployed
- Migration guide

### Phase 3 Success Metrics

| Metric | Target |
|--------|--------|
| Components published | 10 |
| AI pass rate (if used) | > 95% |
| Override rate | < 15% |
| Bugs (30 days) | < 5 |
| Developer satisfaction | > 7/10 |

**Gate:** Developer survey. If satisfaction < 6/10, pause and diagnose.

---

## Phase 4: Stabilization (4 weeks)

### 4.1 Debugging Workflow (Documented)

```
Bug in generated component
        ↓
1. Check runtime behavior → Is it a logic bug?
        ↓
2. Check generation provenance → What produced this?
        ↓
3. Is it reproducible? → Regenerate, compare
        ↓
4. Root cause:
   - Spec bug → Fix spec, regenerate
   - Prompt bug → Fix prompt, regenerate all affected
   - AI variability → Pin to cached response, investigate
   - Correct but unwanted → Add override
```

### 4.2 Incident Response Playbook

```yaml
P1 Bug in Generated Code:
  triage:
    - Identify affected component
    - Check if override or hotfix needed
  hotfix:
    - Edit generated file directly (emergency only)
    - Deploy hotfix
    - Create ticket: "Backport to spec"
  follow-up:
    - Root cause analysis within 48h
    - Prompt or spec fix
    - Regenerate and verify
  SLA: 4 hours to hotfix

On-Call:
  - Primary: dspec tech lead
  - Secondary: senior engineer
  - Escalation: manual fix authorized after 2 hours
```

### 4.3 Rollback System

```bash
# List generation history
dspec history Button

# Rollback to specific generation
dspec rollback Button --to abc123

# Rollback all to last known good
dspec rollback --all --to 2025-03-15
```

### 4.4 Override System (Finalized)

```
src/Button/
├── Button.gen.tsx       # Generated (DO NOT EDIT)
├── Button.override.tsx  # Human patches (optional)
├── Button.tsx           # Merges both, re-exports
└── index.ts
```

**Conflict handling:**
- CI detects merge conflicts
- Build fails with clear message
- Human must update override or accept new generation

### 4.5 Monitoring Dashboard

Track:
- Generation pass rate (target: >95%)
- Override rate per component
- Time since last successful generation
- AI API costs (monthly)
- Alert if pass rate drops below 90%

### 4.6 Schema Evolution Strategy

```yaml
# specs/.schema-version
version: 2

migrations:
  - from: 1
    to: 2
    changes:
      - added: component.accessibility.focusRing
      - deprecated: component.styling.focus (use accessibility.focusRing)
    script: migrations/v1-to-v2.ts
```

**Migration tooling:**
```bash
dspec migrate --from 1 --to 2
```

### 4.7 Knowledge Transfer

- **Pair rotation:** Each engineer pairs with tech lead for 1 week
- **Weekly architecture sync:** 30 min, all 3 FTE
- **Documentation review:** PRs require doc updates
- **Prompt review:** Prompt changes require 2 approvers

### Phase 4 Deliverables
- [ ] Debugging playbook
- [ ] Incident response runbook
- [ ] Rollback system
- [ ] Monitoring dashboard
- [ ] Schema migration tooling
- [ ] Knowledge transfer completed (all 3 FTE)

---

## Phase 5: Multi-Platform (Conditional, 6 weeks)

**Prerequisites:**
- Phase 3 metrics met
- Developer satisfaction > 7/10
- Leadership approval

**Choose ONE:**

### Option A: Figma Integration (4 weeks)
- Bidirectional token sync
- Figma → spec exploration
- Designer workflow documentation

### Option B: iOS/Swift (6 weeks)
- Swift token generator
- SwiftUI component generator
- Button as proof of concept

---

## Phase 6: Designer Tooling (Committed Follow-On)

**Timeline:** Starts within 3 months of Phase 3 completion.

**Allocation:** 1.0 FTE for 3 months (separate project).

**Scope:**
- Figma plugin: design → spec export
- Visual spec editor (web-based)
- Live preview while editing
- Visual diff for design review

**Commitment:** This is not "future/maybe." It's a committed follow-on with resources allocated.

---

## Timeline Summary

| Phase | Duration | Cumulative |
|-------|----------|------------|
| Pre-Phase 0: Baselines | 1 week | 1 week |
| Phase 0: Validation | 2 weeks | 3 weeks |
| Phase 1: Tokens | 6 weeks | 9 weeks |
| Phase 2: First Component | 5 weeks | 14 weeks |
| Phase 3: Component Library | 6 weeks | 20 weeks |
| Phase 4: Stabilization | 4 weeks | 24 weeks |
| Phase 5: Multi-Platform | 6 weeks | 30 weeks |

**MVP (Phases 0-3):** 20 weeks (~5 months)
**Production-Ready (Phases 0-4):** 24 weeks (6 months)

---

## Staffing

| Role | FTE | Responsibilities |
|------|-----|------------------|
| Tech Lead | 1.0 | Architecture, AI quality, prompt ownership |
| Senior Engineer | 1.0 | React generator, quality gates, testing |
| Engineer | 1.0 | Documentation, tooling, CI/CD |
| **Total** | **3.0** | |

**Prompt ownership:** Tech Lead + Senior Engineer both trained. Prompt changes require both to approve.

**Bus factor:** All 3 must understand core system by end of Phase 2.

---

## Kill Criteria

| Phase | Kill If |
|-------|---------|
| 0 | Buy analysis shows existing tool sufficient |
| 0 | Gap analysis shows >30% override needed |
| 0 | AI prototype <90% pass rate |
| 0 | Compound component pattern fails |
| 2 | Static-only decision: AI not worth complexity |
| 2 | AI override rate >30% |
| 3 | Developer satisfaction <6/10 |
| 3 | Bug rate 2x baseline |
| Any | Monthly AI costs exceed projection by 50% |

**"AI Not Worth It" is acceptable.** Ship static-only if that's the right call.

**Sunk cost is not a reason to continue.**

---

## Success Metrics (Summary)

| Metric | Phase 1 | Phase 2 | Phase 3 | Phase 4 |
|--------|---------|---------|---------|---------|
| AI pass rate | N/A | >95% | >95% | >95% |
| Override rate | 0% | <20% | <15% | <10% |
| Time to publish | <5 min | <30 min | <30 min | <15 min |
| Dev satisfaction | — | — | >7/10 | >8/10 |
| Bugs (30 days) | 0 | — | <5 | <2 |

---

## Approval Checklist

**Before Phase 0:**
- [ ] Baseline metrics measured
- [ ] ROI methodology documented
- [ ] Designer stakeholder interviewed
- [ ] 3.0 FTE allocated
- [ ] Kill criteria accepted by leadership

**Before Phase 1:**
- [ ] Buy vs build decision made
- [ ] IR specification written
- [ ] Compound component proposal validated
- [ ] AI reproducibility strategy documented

**Before Phase 3:**
- [ ] Static-only vs AI decision documented
- [ ] Incident response playbook drafted

**Before Phase 5:**
- [ ] Designer tooling project scoped and resourced
