# Rosetta — AI Context

> Handshake's core UI library. Composable, token-driven, styled-components. All new features must use Rosetta.

## Tokens (CSS Variables)

All tokens prefixed `--rosetta-`. Use via `var(--rosetta-*)` in CSS or `rosettaVariables["--rosetta-*"]` in TS.

### Colors — Background

| Token                | Value                 | Usage              |
| -------------------- | --------------------- | ------------------ |
| `bg-primary`         | `rgba(255,255,255,1)` | Page background    |
| `bg-secondary`       | `#f6f6f6`             | Card/section fill  |
| `bg-surface`         | `#fcfcfd`             | Elevated surface   |
| `bg-inverted`        | `rgba(18,18,18,1)`    | Dark backgrounds   |
| `bg-selected`        | `rgba(234,234,234,1)` | Selected row/item  |
| `bg-disabled`        | `rgba(238,238,238,1)` | Disabled elements  |
| `bg-status-info`     | `rgba(204,219,255,1)` | Info background    |
| `bg-status-positive` | `rgba(165,224,159,1)` | Success background |
| `bg-status-warning`  | `rgba(255,217,153,1)` | Warning background |
| `bg-status-negative` | `rgba(255,194,196,1)` | Error background   |
| `bg-status-neutral`  | `rgba(234,234,234,1)` | Neutral background |

### Colors — Foreground

| Token                | Value                 | Usage                |
| -------------------- | --------------------- | -------------------- |
| `fg-primary`         | `rgba(18,18,18,1)`    | Default text         |
| `fg-secondary`       | `rgba(18,18,18,0.7)`  | Secondary/muted text |
| `fg-inverted`        | `rgba(255,255,255,1)` | Text on dark bg      |
| `fg-disabled`        | `rgba(18,18,18,0.4)`  | Disabled text        |
| `fg-border`          | `rgba(31,32,44,0.2)`  | Borders/dividers     |
| `fg-status-info`     | `rgba(0,59,204,1)`    | Info text/icon       |
| `fg-status-positive` | `rgba(50,125,15,1)`   | Success text/icon    |
| `fg-status-warning`  | `rgba(163,103,0,1)`   | Warning text/icon    |
| `fg-status-negative` | `rgba(187,54,67,1)`   | Error text/icon      |

### Colors — Interactive

| Token                          | Value                 | Usage                           |
| ------------------------------ | --------------------- | ------------------------------- |
| `interactive-primary`          | `rgba(4,25,27,1)`     | Primary interactive (dark teal) |
| `interactive-hovered`          | `rgba(18,18,18,0.06)` | Hover overlay                   |
| `interactive-pressed`          | `rgba(18,18,18,0.12)` | Press overlay                   |
| `interactive-focusOutline`     | `rgba(0,59,204,1)`    | Focus ring (blue)               |
| `interactive-inverted-hovered` | `rgba(19,39,41,1)`    | Hover on dark bg                |
| `interactive-inverted-pressed` | `rgba(34,53,54,1)`    | Press on dark bg                |

### Spacing (4px base)

| Token                  | Value |
| ---------------------- | ----- |
| `size-spacing-half`    | 2px   |
| `size-spacing-one`     | 4px   |
| `size-spacing-two`     | 8px   |
| `size-spacing-three`   | 12px  |
| `size-spacing-four`    | 16px  |
| `size-spacing-five`    | 20px  |
| `size-spacing-six`     | 24px  |
| `size-spacing-eight`   | 32px  |
| `size-spacing-twelve`  | 48px  |
| `size-spacing-sixteen` | 64px  |

### Radius

| Token               | Value  | Usage                   |
| ------------------- | ------ | ----------------------- |
| `size-radius-zero`  | 0px    | Sharp corners           |
| `size-radius-one`   | 4px    | Default (inputs, cards) |
| `size-radius-two`   | 8px    | Larger cards, modals    |
| `size-radius-three` | 12px   | Large containers        |
| `size-radius-four`  | 16px   | Extra-large surfaces    |
| `size-radius-max`   | 9999px | Pills, avatars          |

### Shadows

| Token                    | Usage                                                                              |
| ------------------------ | ---------------------------------------------------------------------------------- |
| `shadow-elevation-zero`  | None                                                                               |
| `shadow-elevation-one`   | Cards, low lift                                                                    |
| `shadow-elevation-two`   | Dropdowns, popovers                                                                |
| `shadow-elevation-three` | Modals, dialogs                                                                    |
| `shadow-elevation-four`  | Popovers (same as three)                                                           |
| `shadow-elevation-five`  | Highest elevation (same as three)                                                  |
| `shadow-one`             | Legacy: `0px 1px 2px 0px rgba(31,32,44,0.12)`                                      |
| `shadow-two`             | Legacy: `0px 2px 10px 0px rgba(31,32,44,0.1), 0px 0px 2px 0px rgba(31,32,44,0.04)` |
| `shadow-three`           | Legacy: `0px 8px 32px 0px rgba(31,32,44,0.1), 0px 0px 2px 0px rgba(31,32,44,0.04)` |

### Standalone Sizes (touch targets, icons)

| xs   | sm   | md   | lg   | xl   |
| ---- | ---- | ---- | ---- | ---- |
| 24px | 36px | 48px | 60px | 72px |

### Contained Sizes (icons inside containers)

| xxs  | xs   | sm   | md   | lg   | xl   |
| ---- | ---- | ---- | ---- | ---- | ---- |
| 12px | 16px | 20px | 24px | 28px | 32px |

### Typography

| Token               | Value                                |
| ------------------- | ------------------------------------ |
| `font-family-sans`  | "Noi Grotesk", system-ui, sans-serif |
| `font-family-brand` | "Sansplomb", sans-serif              |
| `font-family-mono`  | "IBM Plex Mono", monospace           |
| Font weights        | regular=400, semiBold=500, bold=700  |
| Display weights     | regular=400, bold=700                |

**Font Size Scale:** fs1=12px, fs2=14px, fs3=16px, fs4=20px, fs5=24px, fs6=32px, fs7=40px, fs8=46px, fs9=58px (base=16px)

**Line Height Scale:** lh1=4px, lh2=8px, lh3=12px, lh4=16px, lh5=20px, lh6=24px, lh7=28px, lh8=32px

**Type Scale** (use Typography mixins, not raw vars):

| Style          | Size | Weight  | Line Height |
| -------------- | ---- | ------- | ----------- |
| heading.xlarge | 44px | 500     | 1.0         |
| heading.large  | 32px | 500     | 1.2         |
| heading.medium | 28px | 500     | 1.2         |
| heading.small  | 24px | 500     | 1.2         |
| body.xlarge    | 19px | 400/500 | 1.2         |
| body.large     | 17px | 400/500 | 1.4         |
| body.medium    | 15px | 400/500 | 1.2         |
| body.small     | 13px | 400/500 | 1.2         |
| mono.1         | 17px | 400/500 | 1.25        |
| mono.2         | 15px | 400/500 | 1.25        |
| mono.3         | 13px | 400/500 | 1.2/1.25    |
| mono.4         | 11px | 400/500 | 1.2/1.25    |

**Letter Spacing:**

| Style          | Spacing |
| -------------- | ------- |
| brand-large    | -0.02em |
| brand-small    | -0.01em |
| heading-xlarge | -0.04em |
| heading-large  | -0.03em |
| heading-medium | -0.03em |
| heading-small  | -0.02em |
| body-xlarge    | -0.02em |
| body-large     | -0.02em |
| body-medium    | 0       |
| body-small     | 0.01em  |
| mono-1         | -0.01em |
| mono-2         | -0.01em |
| mono-3         | 0       |
| mono-4         | 0       |

**Brand Typography:**

| Style       | Size  | Weight | Line Height |
| ----------- | ----- | ------ | ----------- |
| brand-large | 122px | 600    | 0.8         |
| brand-small | 60px  | 600    | 0.9         |

**Font Families (display/text aliases):**

- `typography-font-family-display` = "Noi Grotesk", system-ui, sans-serif
- `typography-font-family-text` = "Noi Grotesk", system-ui, sans-serif

```tsx
// Use Typography mixins from @joinhandshake/rosetta
const Title = styled.h1`
  ${Typography.heading.large()}
`;
const Text = styled.p`
  ${Typography.body.medium({ semibold: true })}
`;
```

**Default body typography:** Body Medium (15px, weight 400, line-height 1.2, letter-spacing 0)
**OpenType features:** `font-feature-settings: "ss03" on, "ss11" on;`

## Principles

- **Composable**: Build complex UI from small, single-purpose components
- **Token-driven**: All color/spacing/type from CSS variables, never hardcode
- **Accessible first**: WCAG AA (4.5:1 contrast), keyboard nav, aria-labels required
- **Responsive**: 5 breakpoints (XS:0-480, SM:481-768, M:769-1024, L:1025-1440, XL:1441+)
- **Styled-components**: Use Typography mixins + CSS vars, not raw CSS values

## Component Patterns

### Button

- **Variants**: `PrimaryButton`, `SecondaryButton`, `TertiaryButton`, `SubduedButton` (each has `*DestructiveButton`)
- **Sizes**: `Size.small | Size.medium | Size.large` (36px, 48px, 60px)
- **Props**: `onClick`, `disabled`, `loading`, `as` (for Links)
- **Border Radius**: `radius-two` (8px), `radius-max` for icon-only

```tsx
<PrimaryButton onClick={handleClick} size={Size.small}>
  <RosettaLeftIcon><Rosetta.Add /></RosettaLeftIcon>
  <ButtonText>Add Item</ButtonText>
</PrimaryButton>

// As a link
<PrimaryButton as={Link} to="/path">
  <ButtonText>Navigate</ButtonText>
</PrimaryButton>

// Icon-only
<SubduedSimpleIconButton aria-label="Close" size={Size.small}>
  <Rosetta.Close />
</SubduedSimpleIconButton>
```

### Modal

- **Use for**: Confirmations, short forms, focused tasks
- **Sizes**: `ModalSizes.Narrow | ModalSizes.Wide`
- **Pattern**: `disclosure` prop triggers open, `<Dismiss>` wraps close elements
- **Nested**: Use `nested` prop or `useDialog({ nested: true })` for nested modals

```tsx
<Modal
  aria-label="Confirm action"
  size={ModalSizes.Narrow}
  yAxis="100px"
  disclosure={
    <PrimaryButton>
      <ButtonText>Open</ButtonText>
    </PrimaryButton>
  }
>
  <Header>
    <h1>Title</h1>
    <Dismiss>
      <SubduedSimpleIconButton aria-label="Close modal" size={Size.small}>
        <Rosetta.Close />
      </SubduedSimpleIconButton>
    </Dismiss>
  </Header>
  <Body>
    <p>Modal content goes here</p>
  </Body>
  <Footer>
    <Dismiss>
      <SecondaryButton size={Size.small}>
        <ButtonText>Cancel</ButtonText>
      </SecondaryButton>
    </Dismiss>
    <Dismiss>
      <PrimaryButton size={Size.small}>
        <ButtonText>Confirm</ButtonText>
      </PrimaryButton>
    </Dismiss>
  </Footer>
</Modal>
```

### Select

- **Sizes**: `FieldSize.Small | FieldSize.Medium`
- **Status**: `FieldStatus.Warning | .Negative | .Positive` (or undefined for default)
- **Props**: `label`, `placeholder`, `helperText`, `disabled`, `maxHeight`, `minHeight`, `placement`, `flip`

```tsx
<Select
  label="Role"
  placeholder="Select a role..."
  data-size={FieldSize.Medium}
  helperText="Choose the user's role"
  data-status={FieldStatus.Warning}
>
  <Option value="admin">Admin</Option>
  <Option value="editor">Editor</Option>
  <Option value="viewer">Viewer</Option>
</Select>

// With groups and dividers
<Select label="Grouped Options" placement={FieldPlacement.Bottom} flip={false}>
  <Option value="opt1">Option 1</Option>
  <Divider />
  <span style={{ padding: "4px 16px", fontWeight: 500 }}>Group Label</span>
  <Option value="opt2">Option 2</Option>
</Select>
```

### InlineAlert

- **Statuses**: `InlineAlertStatus.Information | .Warning | .Negative | .Positive | .Ai`
- **Props**: `title`, `data-status`, `icon` (boolean), `actionableElements`, `children`
- **Actions**: Use `useInlineAlert()` or `InlineAlertContext` for `hide()` function

```tsx
<InlineAlert
  title="Changes saved successfully"
  data-status={InlineAlertStatus.Positive}
  icon
>
  <div>Your profile has been updated.</div>
</InlineAlert>

// With action buttons
<InlineAlert
  title="New version available"
  data-status={InlineAlertStatus.Information}
  icon
  actionableElements={
    <ButtonGroup>
      <SecondaryButton size={Size.small} onClick={dismiss}>
        <ButtonText>Dismiss</ButtonText>
      </SecondaryButton>
      <PrimaryButton size={Size.small} onClick={update}>
        <ButtonText>Update Now</ButtonText>
      </PrimaryButton>
    </ButtonGroup>
  }
>
  <div>A new version is available with bug fixes and improvements.</div>
</InlineAlert>
```

### Toast

- **Statuses**: `ToastStatus.Positive | .Warning | .Negative`
- **Props**: `id` (required), `title`, `body`, `data-status`, `duration` (ms), `onComplete`
- **Must wrap in `<Toasts>` container**

```tsx
<Toasts>
  <Toast
    id="save-toast"
    title="Changes saved"
    body="Your changes have been saved successfully."
    data-status={ToastStatus.Positive}
    duration={5000}
    onComplete={handleDismiss}
  />
</Toasts>

// Multiple toasts
<Toasts>
  {toasts.map(({ id, title, body }, index) => (
    <Toast
      key={id}
      id={id}
      title={title}
      body={body}
      data-status={ToastStatus.Positive}
      onComplete={handleRemove(index)}
    />
  ))}
</Toasts>
```

### Tag

- **Variants**: `TagVariants.Static | .Interactive | .Dismissable`
- **Sizes**: `TagSizes.Small | .Medium`
- **Colors**: `undefined | "purple" | "teal" | "pink" | "red" | "green" | "yellow"`
- **Props**: `size`, `variant`, `color`, `disabled`, `onClick`, `icon`, `avatar`

```tsx
// Static tag
<Tag size={TagSizes.Medium} variant={TagVariants.Static}>
  <TagText>Engineering</TagText>
</Tag>

// With icon
<Tag
  size={TagSizes.Medium}
  variant={TagVariants.Interactive}
  icon={<Rosetta.Archive width={IconSize.Small} />}
  onClick={handleClick}
>
  <TagText>Archived</TagText>
</Tag>

// Dismissable with color
<Tag size={TagSizes.Medium} variant={TagVariants.Dismissable} color="purple">
  <TagText>Design</TagText>
</Tag>

// With avatar
<Tag
  size={TagSizes.Medium}
  variant={TagVariants.Static}
  avatar={
    <HumanAvatar size={ContainedAvatarSizes.Small} containerContext={ContainerContext.Contained}>
      <img src={avatarUrl} alt="User" style={{ objectFit: "cover" }} />
    </HumanAvatar>
  }
>
  <TagText>Jane Doe</TagText>
</Tag>
```

### Banner

- **Styled variants**: `InformationBanner`, `WarningBanner`, `NegativeBanner`, `PositiveBanner`, `AiBanner`
- **Props**: `title`, `bodyText`, `icon`, `rightIcon`, `actionableElements`, `fill`
- **Hook**: `useBanner()` provides `hide()` for dismiss

```tsx
<InformationBanner
  title="You messaged your top matches!"
  bodyText="We'll notify you when they respond."
  icon={<Rosetta.InformationCircle />}
  rightIcon={<Rosetta.Close />}
/>

// With actions
<WarningBanner
  title="Your subscription expires soon"
  bodyText="Renew before Feb 1 to keep your access."
  icon={<Rosetta.Warning />}
  actionableElements={<ActionButtons />}
/>
```

### Table

- **Hooks**: `useRowSelect()` → `{ selected, onRowSelect, onAllRowsSelect }`, `useSortBy(initialState)` → `{ sorting, onSort }`
- **Sizes**: `TableSize.Small | .Medium`
- **Composition**: Standard HTML `<table>/<thead>/<tbody>/<tr>/<td>` with Rosetta enhancements
- **Key elements**: `<SortableHeader>` for sortable columns, `<Checkbox>` for row selection, `<Menu>` + `<MenuItem>` for actions, `<Pagination>` for paging

```tsx
const { selected, onRowSelect, onAllRowsSelect } = useRowSelect();
const { sorting, onSort } = useSortBy({ name: Sortable.Ascending });

<Layout>
  <Header data-size={dataSize}>
    <Content data-selected={!!Object.keys(selected).length}>
      <Selection>
        {Object.keys(selected).length
          ? `${Object.keys(selected).length} selected`
          : ""}
      </Selection>
      <Toolbar>
        <Menu
          disclosure={
            <Button>
              <Rosetta.MenuHamburger />
            </Button>
          }
          data-size={MenuSize[dataSize]}
          placement={MenuPlacement.BottomEnd}
        >
          <MenuItem>Remove</MenuItem>
          <MenuItem>Export</MenuItem>
        </Menu>
      </Toolbar>
    </Content>
  </Header>

  <Table data-size={dataSize}>
    <thead>
      <tr>
        <th>
          <IndeterminateCheckbox
            id="select-all"
            indeterminate={isIndeterminate}
            checked={isAllChecked}
            onChange={onAllRowsSelect}
          />
        </th>
        <SortableHeader onClick={onSort} aria-sort={sorting.name} name="name">
          Name
        </SortableHeader>
        <SortableHeader onClick={onSort} aria-sort={sorting.title} name="title">
          Title
        </SortableHeader>
        <th>Company</th>
      </tr>
    </thead>
    <tbody>
      {rows.map(({ id, name, title, company }, i) => (
        <tr key={id}>
          <td>
            <Checkbox
              id={id}
              checked={!!selected[i]}
              onChange={onRowSelect}
              value={i}
            />
          </td>
          <td>{name}</td>
          <td>{title}</td>
          <td>{company}</td>
        </tr>
      ))}
    </tbody>
  </Table>

  <Pagination
    value={page}
    onChange={(e) => setPage(e.target.value)}
    total={totalPages}
    data-size={PaginationSize[dataSize]}
  />
</Layout>;
```

### Card

- **Composition**: `StyledCard` > `StyledCardHeader` + `StyledCardContentBody` + `StyledCardFooter`
- **Header**: Can include avatar via `HumanAvatar`, title via `Title`, subtitle via `SubTitle`

```tsx
<StyledCard>
  <StyledCardHeader>
    <HumanAvatar size={StandaloneAvatarSizes.Medium}>
      <img src={avatarUrl} alt="User" />
    </HumanAvatar>
    <StyledCardHeaderTextContainer>
      <Title>Card Title</Title>
      <SubTitle>Supporting text</SubTitle>
    </StyledCardHeaderTextContainer>
  </StyledCardHeader>
  <StyledCardContentBody>
    <p>Card content goes here</p>
  </StyledCardContentBody>
  <StyledCardFooter>
    <TertiaryButton size={Size.small}>
      <ButtonText>Action</ButtonText>
    </TertiaryButton>
  </StyledCardFooter>
</StyledCard>
```

### Dialog

- **Props**: `visible`, `disclosure`, `children`, `nested`
- Lower-level than Modal — use Modal for most cases

```tsx
<Dialog
  disclosure={
    <PrimaryButton>
      <ButtonText>Open</ButtonText>
    </PrimaryButton>
  }
>
  Dialog content here
</Dialog>
```

### Tooltip

- **Placements**: `Placement.Top | .Bottom | .Right | .Left`
- **Props**: `title` (simple text) or `overlay` (rich content), `placementOverride`

```tsx
<Tooltip title="More information" placementOverride={Placement.Top}>
  <Rosetta.InformationCircle
    fill="var(--rosetta-fg-secondary)"
    width="24"
    height="24"
  />
</Tooltip>
```

## Layouts

- **GridLayout**: 12-column responsive grid with `spans` (column positioning per breakpoint) and `itemSpans` (item sizing). Breakpoints: XS/SM=4 cols, M=8 cols, L/XL=12 cols
- **TwoColumnLeftLayout**: Master-detail layout with `leftContent`/`rightContent`, `focusedColumn` ("left"|"right"), `verticalDivider`, `responsiveBreakpoint`, `emptyState`, toast/modal integration via props

## Do / Don't

**Do:**

- Wrap button text in `<ButtonText>`, icons in `<RosettaLeftIcon>`/`<RosettaRightIcon>`
- Use `data-size` and `data-status` attributes for sizing/status control
- Use `aria-label` on icon-only buttons and inputs without visible labels
- Use Rosetta CSS vars for all colors/spacing/radii
- Use `<Dismiss>` wrapper for elements that close modals
- Use `as={Link}` prop on buttons when they should navigate

**Don't:**

- Hardcode colors, spacing, or font sizes — use tokens
- Use Button for navigation — use `<Link>` (or `as={Link}`)
- Put too much content in modals — if scrolling needed, use a page
- Use raw HTML `<select>` / `<input>` — use Rosetta field components
- Skip `label` prop on fields — required for a11y (use `aria-label` if hidden)
- Use inverted button variants (deprecated)
- Use tokens marked `--deprecated` — use the non-deprecated equivalent

## Accessibility

- **Contrast**: 4.5:1 minimum (WCAG AA)
- **Focus**: All interactive elements must have visible focus ring (`interactive-focusOutline`)
- **Keyboard**: Enter/Space activate buttons, Escape closes modals, Tab navigates
- **Modals**: Auto focus-trap, `aria-label` required, Escape to dismiss
- **Forms**: All fields need labels (visible or `aria-label`), focus moves to first error on submit
- **Tables**: Use `<th>` with `scope`, `<caption>` for title, `aria-sort` on sortable headers

## Full Documentation

- Docs: https://rosetta.joinhandshake-internal.com/
- Keystone: https://keystone.joinhandshake-internal.com/
- Playground: https://rosetta-playground.joinhandshake-internal.com/
