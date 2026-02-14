# /phi-type — Golden Ratio Typography

Apply a φ-based type scale to any web project's CSS. Part of the [AI Brand Studio](~/1-Projects/ai-brand-studio/).

## Invocation

```
/phi-type [path-to-css-or-project]
```

Examples:
- `/phi-type src/styles/global.css` — apply to a specific CSS file
- `/phi-type` — auto-detect CSS in current project

## Prerequisites

- A web project with CSS (plain CSS, Astro, Tailwind, etc.)

---

# EXECUTION INSTRUCTIONS

## Step 1: Parse Arguments

Extract from `$ARGUMENTS`:
- `TARGET` — path to CSS file or project directory (optional, will auto-detect)

If no target provided, search for CSS files:
1. `src/**/*.css`, `src/**/*.astro`, `app/**/*.css`
2. `styles/**/*.css`
3. `*.css` in project root

If multiple candidates, use `AskUserQuestion` to let the user pick.

## Step 2: Audit Current Typography

Read the target file(s) and catalog every `font-size` declaration:

```
CURRENT TYPE AUDIT:
- html/body base: [value]
- h1: [value]
- h2: [value]
- nav: [value]
- body/p: [value]
- small/footer/meta: [value]
- code: [value]
- other: [list]
```

Present this to the user before making changes.

## Step 3: Define the φ Scale

The golden ratio type scale uses φ = 1.618:

```css
:root {
  /* Golden ratio type scale (φ = 1.618) */
  --font-xs: 0.618rem;    /* 1/φ — footnotes, timestamps */
  --font-sm: 0.786rem;    /* √(1/φ) — nav, labels, meta */
  --font-base: 1rem;      /* base — body text */
  --font-md: 1.272rem;    /* √φ — subheadings, emphasis */
  --font-lg: 1.618rem;    /* φ — section headings */
  --font-xl: 2.618rem;    /* φ² — hero/display (use sparingly) */
}
```

The base `font-size` on `html` should be `16px` (browser default). Line-height should be between `1.618` (φ itself) and `1.75` for monospace fonts.

## Step 4: Map Elements to Scale Steps

Assign each element to the appropriate scale step:

| Scale step | CSS var | Typical elements |
|------------|---------|-----------------|
| φ² | `--font-xl` | Hero headlines, display text |
| φ | `--font-lg` | Section headings (h1, h2) |
| √φ | `--font-md` | Subheadings (h3), emphasized text |
| 1 | `--font-base` | Body paragraphs, list items |
| √(1/φ) | `--font-sm` | Navigation, labels, captions, code |
| 1/φ | `--font-xs` | Footnotes, timestamps, legal text |

Rules:
- **Never skip more than one scale step** between adjacent elements
- **Headings descend**: h1 ≥ h2 ≥ h3, each one step down
- **Body text is always base** — don't shrink paragraphs
- **Code inherits its context's size** unless it's a standalone block

## Step 5: Apply Changes

Replace hardcoded `font-size` values with `var(--font-*)` references.

Add the `:root` custom properties block if not already present.

Set `html { font-size: 16px; line-height: 1.75; }` (adjust line-height for the font family — 1.75 for monospace, 1.618 for proportional).

## Step 6: Present Changes

Show a summary:

```
φ TYPE SCALE APPLIED:

:root custom properties added:
  --font-xs through --font-xl (golden ratio steps)

Elements remapped:
  hero h1:     [old] → var(--font-lg)  [1.618rem]
  section h2:  [old] → var(--font-md)  [1.272rem]
  body:        [old] → var(--font-base) [1rem]
  nav:         [old] → var(--font-sm)  [0.786rem]
  footer:      [old] → var(--font-xs)  [0.618rem]

Line-height: [old] → 1.75
Base: 16px

The scale ratios: each step is related to its neighbors by φ (1.618).
```

---

# NOTES

- This skill modifies CSS in place. It does NOT create new files.
- For Astro projects with `<style>` blocks in `.astro` files, edit the style block directly.
- For Tailwind projects, add the scale as CSS custom properties and use them in `@apply` or direct class overrides.
- If the project already uses a type scale, present both and let the user choose.
- The φ scale works best with a single font family. Mixed font stacks may need per-family adjustments.
