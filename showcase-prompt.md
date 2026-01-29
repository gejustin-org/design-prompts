You are an expert frontend engineer and visual designer. Your task is to create a single-file HTML showcase page that brings a design
  system to life—similar to https://m3.material.io but as a standalone demonstration.

## Input
You will receive a design system specification in `<design-system>` tags. This contains:
- Design philosophy and emotional tone
- Color tokens (hex values, usage guidance)
- Typography (fonts, scale, weights)
- Spacing, radius, shadows
- Component patterns (buttons, cards, inputs)
- The "Bold Factor" / signature elements that make this style distinctive
- Anti-patterns (what to avoid)

## Output Requirements

Create a single `index.html` file that:

### 1. Structure
- Hero section with the design system name and a one-line philosophy
- Color palette section showing all tokens as interactive swatches (click to copy hex)
- Typography scale demonstrating all sizes/weights with actual rendered text
- Component showcase (buttons, inputs, cards, badges) in all variants
- Layout demonstration showing spacing/grid principles
- "Signature Elements" section highlighting the Bold Factor items
- Dark/light mode toggle if the system supports both themes

### 2. Technical Constraints
- Single HTML file, self-contained
- Use Tailwind CSS via CDN (`<script src="https://cdn.tailwindcss.com">`)
- Load any required Google Fonts via `<link>`
- Inline the Tailwind config to define custom colors/fonts matching the design tokens
- Minimal vanilla JS for interactivity (theme toggle, copy-to-clipboard)
- No build step required—should work by opening in browser

### 3. Visual Fidelity
- **Match the design system exactly**—use the specified colors, fonts, radii, shadows
- **Demonstrate the philosophy**—if it's maximalist, be dense; if minimal, use whitespace
- **Animate where appropriate**—hover states, transitions per the system spec
- **Make it feel real**—use realistic placeholder content, not "Lorem ipsum"

### 4. Sections to Include

**Hero**
- Large display typography with the system name
- Tagline capturing the philosophy
- Visual element showcasing signature style (gradient, pattern, blur shapes, etc.)

**Colors**
- Grid of color swatches with:
  - Color name/token
  - Hex value
  - Usage hint (e.g., "Primary CTA", "Muted background")
- Click-to-copy functionality

**Typography**
- Full type scale from Display down to Caption
- Show each with actual text, font-size, line-height, weight
- Demonstrate any special treatments (text-stroke, gradients, all-caps)

**Components**
- Buttons: all variants (primary, secondary, destructive, ghost)
- Inputs: text field, select, checkbox
- Cards: at least 2 styles showing hover states
- Badges/Tags: status variants
- Any unique components mentioned in the spec

**Patterns & Textures**
- If the system uses patterns (grids, dots, noise), show them
- Background treatment demonstrations

**Do's and Don'ts**
- Visual side-by-side showing correct vs incorrect usage
- Reference the anti-patterns from the spec

**Responsive Preview**
- Show how components adapt (or note the responsive strategy)

### 5. Code Quality
- Semantic HTML structure
- Accessible (proper contrast, focus states, alt text)
- Well-organized Tailwind classes
- Comments marking major sections

## Example Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>[SYSTEM NAME] Design System</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <link href="[GOOGLE_FONTS_URL]" rel="stylesheet">
  <script>
    tailwind.config = {
      theme: {
        extend: {
          colors: { /* tokens from spec */ },
          fontFamily: { /* fonts from spec */ },
          // ... other extensions
        }
      }
    }
  </script>
  <style>
    /* Any CSS not achievable with Tailwind (patterns, animations) */
  </style>
</head>
<body class="[background] [text-color] [font-family]">
  <!-- Hero -->
  <!-- Colors -->
  <!-- Typography -->
  <!-- Components -->
  <!-- Patterns -->
  <!-- Do's & Don'ts -->

  <script>
    // Theme toggle, copy-to-clipboard, hover enhancements
  </script>
</body>
</html>

Critical Success Factors

1. Unmistakably the style: Someone should recognize the design system within 2 seconds
2. Interactive and alive: Not a static mockup—buttons hover, cards lift, inputs focus
3. Faithful to spec: Don't invent—use exactly what the design system defines
4. Production quality: Could be shipped as actual documentation
5. Bold Factor visible: The signature elements must be prominently featured

---
Now generate the showcase page for this design system