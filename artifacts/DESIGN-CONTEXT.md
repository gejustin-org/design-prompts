# Rosetta — AI Context

> Professional clarity with human warmth. Handshake's design system for a platform where students and employers connect. Readable, accessible, trustworthy — never cold or generic.

## The Vibe

**Clean. Approachable. Trustworthy. Modern.** Like a well-designed coworking space — organized and professional but not intimidating. Content is the hero; UI recedes.

**This design is NOT:** playful/whimsical · dark/moody · decorative/ornate · cold/clinical · trendy/experimental

## DNA (What Makes Rosetta Rosetta)

1. **Dark Teal, Not Blue** — `#04191B` is the signature. Sophisticated, distinctive, avoids "generic SaaS blue"
2. **Noi Grotesk + Stylistic Alternates** — `font-feature-settings: "ss03" on, "ss11" on` gives subtle character
3. **Shadow-First Elevation** — Soft shadows over heavy borders. Cards float, not sit
4. **4px Grid Discipline** — Everything snaps to 4px. Spacing: 2/4/8/12/16/20/24/32/48/64
5. **Semantic Color Restraint** — Color means something. Don't use color decoratively
6. **White Space as Feature** — Generous padding/margins. The interface breathes

## Tokens (`--rosetta-*`)

Use via `var(--rosetta-*)` in CSS or `rosettaVariables["--rosetta-*"]` in TS.

### Colors

| Token                      | Value                 | Usage                |
| -------------------------- | --------------------- | -------------------- |
| `bg-primary`               | `#FFF`                | Page background      |
| `bg-secondary`             | `#F6F6F6`             | Card/section fill    |
| `bg-surface`               | `#FCFCFD`             | Elevated surface     |
| `bg-inverted`              | `#121212`             | Dark backgrounds     |
| `bg-selected`              | `#EAEAEA`             | Selected state       |
| `bg-disabled`              | `#EEEEEE`             | Disabled elements    |
| `fg-primary`               | `#121212`             | Default text         |
| `fg-secondary`             | `rgba(18,18,18,0.7)`  | Muted text           |
| `fg-inverted`              | `#FFF`                | Text on dark bg      |
| `fg-disabled`              | `rgba(18,18,18,0.4)`  | Disabled text        |
| `fg-border`                | `rgba(31,32,44,0.2)`  | Borders/dividers     |
| `interactive-primary`      | `#04191B`             | Buttons, key actions |
| `interactive-hovered`      | `rgba(18,18,18,0.06)` | Hover overlay        |
| `interactive-pressed`      | `rgba(18,18,18,0.12)` | Press overlay        |
| `interactive-focusOutline` | `#003BCC`             | Focus ring (blue)    |

**Status Colors:** Each has bg (tint) + fg (saturated) pair:
Info `#CCDBFF`/`#003BCC` · Positive `#A5E09F`/`#327D0F` · Warning `#FFD999`/`#A36700` · Negative `#FFC2C4`/`#BB3643` · Neutral `#EAEAEA`

### Spacing (4px base)

`half=2` · `one=4` · `two=8` · `three=12` · `four=16` · `five=20` · `six=24` · `eight=32` · `twelve=48` · `sixteen=64`

### Radius

| Token         | Value  | When                                                                      |
| ------------- | ------ | ------------------------------------------------------------------------- |
| `radius-one`  | 4px    | Interactive elements (inputs, buttons)                                    |
| `radius-two`  | 8px    | Small interactive where shape trends circular (checkboxes, small avatars) |
| `radius-four` | 16px   | Large surfaces (cards, containers)                                        |
| `radius-max`  | 9999px | Pills, round avatars                                                      |

### Shadows (Elevation)

`elevation-zero`: none · `elevation-one`: subtle lift (cards) · `elevation-two`: dropdowns · `elevation-three`: modals/dialogs

### Typography

**Fonts:** Sans=`"Noi Grotesk", system-ui, sans-serif` · Brand=`"Sansplomb", sans-serif` · Mono=`"IBM Plex Mono", monospace`
**Weights:** regular=400, semiBold=500, bold=700

| Style          | Size | Weight  | LH  | Letter Spacing |
| -------------- | ---- | ------- | --- | -------------- |
| heading.xlarge | 44px | 500     | 1.0 | -0.04em        |
| heading.large  | 32px | 500     | 1.2 | -0.03em        |
| heading.medium | 28px | 500     | 1.2 | -0.03em        |
| heading.small  | 24px | 500     | 1.2 | -0.02em        |
| body.large     | 17px | 400/500 | 1.4 | -0.02em        |
| body.medium    | 15px | 400/500 | 1.2 | 0              |
| body.small     | 13px | 400/500 | 1.2 | 0.01em         |

```tsx
const Title = styled.h1`
  ${Typography.heading.large()}
`;
const Text = styled.p`
  ${Typography.body.medium({ semibold: true })}
`;
```

**Standalone sizes** (touch targets): xs=24 sm=36 md=48 lg=60 xl=72
**Contained sizes** (icons in containers): xxs=12 xs=16 sm=20 md=24 lg=28 xl=32

## Components (`@joinhandshake/rosetta`)

### Button

Variants: `PrimaryButton`, `SecondaryButton`, `TertiaryButton`, `SubduedButton` (each has `*DestructiveButton`). Sizes: `Size.small|medium|large`.

```tsx
<PrimaryButton onClick={handle} size={Size.small}>
  <RosettaLeftIcon>
    <Rosetta.Add />
  </RosettaLeftIcon>
  <ButtonText>Add Item</ButtonText>
</PrimaryButton>
```

### Modal

Sizes: `ModalSizes.Narrow|Wide`. `disclosure` triggers open, `<Dismiss>` wraps close elements.

```tsx
<Modal
  aria-label="Confirm"
  size={ModalSizes.Narrow}
  disclosure={
    <PrimaryButton>
      <ButtonText>Open</ButtonText>
    </PrimaryButton>
  }
>
  <Header>
    <h1>Title</h1>
    <Dismiss>
      <SubduedSimpleIconButton aria-label="Close">
        <Rosetta.Close />
      </SubduedSimpleIconButton>
    </Dismiss>
  </Header>
  <Body>
    <p>Content</p>
  </Body>
  <Footer>
    <Dismiss>
      <PrimaryButton size={Size.small}>
        <ButtonText>OK</ButtonText>
      </PrimaryButton>
    </Dismiss>
  </Footer>
</Modal>
```

### Select

```tsx
<Select
  label="Role"
  placeholder="Choose..."
  data-size={FieldSize.Medium}
  helperText="Required"
>
  <Option value="admin">Admin</Option>
</Select>
```

Status: `FieldStatus.Warning|Negative|Positive`

### InlineAlert

```tsx
<InlineAlert title="Saved" data-status={InlineAlertStatus.Positive} icon>
  <div>Done.</div>
</InlineAlert>
```

Statuses: `Information|Warning|Negative|Positive|Ai`

### Toast — Wrap in `<Toasts>`. Statuses: `ToastStatus.Positive|Warning|Negative`

### Tag — Variants: `Static|Interactive|Dismissable`. Colors: `purple|teal|pink|red|green|yellow`. Wrap text in `<TagText>`.

### Banner — `InformationBanner`, `WarningBanner`, `NegativeBanner`, `PositiveBanner`, `AiBanner`

### Table — Compositional `<table>` with `<SortableHeader>`, hooks `useRowSelect()`, `useSortBy()`, `<Pagination>`

## Layouts

- **GridLayout**: 12-col responsive grid, `spans`/`itemSpans` props
- **TwoColumnLeftLayout**: Master-detail, `leftContent`/`rightContent`, `focusedColumn`

## Motion

Functional, not decorative. Standard: `transition: 0.2s ease-out`. Respect `prefers-reduced-motion`.

## Do / Don't

**Do:**

- Wrap button text in `<ButtonText>`, icons in `<RosettaLeftIcon>`/`<RosettaRightIcon>`
- Use `data-size`/`data-status` attributes for sizing/status
- Use `aria-label` on icon-only buttons and inputs without visible labels
- Use Rosetta tokens for ALL colors/spacing/radii — never hardcode
- Use `<Dismiss>` wrapper for elements that close modals
- Buttons for actions, `<Link>` for navigation
- Min touch target 44px

**Don't:**

- Hardcode colors, spacing, or font sizes
- Invent new tokens or components not in Rosetta
- Use color decoratively — color carries semantic meaning
- Put too much in modals — if scrolling needed, use a page
- Use raw HTML `<select>`/`<input>` — use Rosetta fields
- Skip labels on fields (use `aria-label` if visually hidden)
- Use inverted button variants or `--deprecated` tokens

## Accessibility

- **Contrast**: WCAG AA (4.5:1 normal text, 3:1 large text)
- **Focus**: `outline: var(--rosetta-interactive-focusOutline) solid 2px; outline-offset: 2px`
- **Keyboard**: Enter/Space activate buttons, Escape closes modals, Tab navigates
- **Modals**: Auto focus-trap, `aria-label` required
- **Forms**: All fields need labels, focus moves to first error on submit
- **Tables**: `<th>` with `scope`, `<caption>` for title, `aria-sort` on sortable headers
- **Reduced motion**: Respect `prefers-reduced-motion: reduce`

## Docs

- Rosetta: https://rosetta.joinhandshake-internal.com/
- Keystone: https://keystone.joinhandshake-internal.com/
- Playground: https://rosetta-playground.joinhandshake-internal.com/
