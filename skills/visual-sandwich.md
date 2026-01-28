---
description: Convert design mockups to pixel-accurate HTML/CSS through iterative VLM refinement
---

# Visual Sandwich Workflow Skill

Convert design mockups to pixel-accurate HTML/CSS through iterative VLM refinement.

## When to Use

Use this skill when you have:
- A **reference design image** to match
- An **existing HTML/CSS implementation** to refine
- Need for **pixel-accurate** results

## The Visual Sandwich Pattern

Each iteration sends three inputs to a VLM:
1. **Reference design** (image) - the target
2. **Current screenshot** (image) - what we have now
3. **Current code** (text) - the HTML/CSS

The VLM compares and returns **specific CSS fixes**.

## Workflow Steps

### Step 1: Setup

Ensure you have:
- Reference image in `workspace/designs/`
- Initial HTML file (generate with frontend-design skill if needed)
- Playwright installed for screenshots

### Step 2: Take Screenshot

```bash
python3 << 'EOF'
from playwright.sync_api import sync_playwright
import os

html_path = os.path.abspath("path/to/index.html")
screenshot_path = "path/to/screenshot.png"

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page(viewport={"width": 1920, "height": 1080})
    page.goto(f"file://{html_path}")
    page.wait_for_timeout(2000)
    page.screenshot(path=screenshot_path)
    browser.close()
EOF
```

### Step 3: VLM Comparison

Use PAL MCP chat with both images:

```
Compare these two images:

IMAGE 1 (Reference design - the target)
IMAGE 2 (Current implementation)

Analyze the differences and provide SPECIFIC CSS fixes.

Focus on:
1. Layout/spacing differences
2. Color/glow intensity
3. Border/corner styling
4. Missing visual elements

Provide targeted CSS blocks I can apply.
```

### Step 4: Apply Fixes

Apply the CSS changes recommended by the VLM.

### Step 5: Repeat

Take a new screenshot and run another comparison. Typically converges in 2-4 iterations.

## Key CSS Techniques

### Triple-Stack Bloom (Glow Effects)
```css
filter: drop-shadow(0 0 2px var(--color))
        drop-shadow(0 0 10px var(--color))
        drop-shadow(0 0 30px var(--color));
```

### White Hot Core (Light Physics)
```css
.element {
    stroke: #ffffff;  /* White core */
    filter: drop-shadow(0 0 4px var(--neon-cyan))
            drop-shadow(0 0 8px var(--neon-cyan));
}
```

### clip-path + drop-shadow Fix
```css
/* box-shadow gets clipped! Use drop-shadow instead */
.clipped-element {
    clip-path: polygon(...);
    filter: drop-shadow(0 0 5px cyan);
}
```

### Deep Black Backgrounds
```css
/* High contrast: near-black backgrounds make colors pop */
.panel { background: #080808; }
.display { background: #000; }
```

### Floating Tick Separators
```css
.section::before {
    content: '';
    position: absolute;
    left: 0;
    top: 20%;
    bottom: 20%;
    width: 1px;
    background: linear-gradient(to bottom,
        transparent,
        var(--gold-dim),
        var(--gold-primary),
        var(--gold-dim),
        transparent
    );
}
```

### Hardware Box Containment
```css
.container {
    max-width: 1500px;
    margin: 0 auto;
    border-left: 1px solid var(--border);
    border-right: 1px solid var(--border);
}
```

### Technical Grid Background
```css
.panel {
    background-image:
        linear-gradient(rgba(255, 184, 0, 0.03) 1px, transparent 1px),
        linear-gradient(90deg, rgba(255, 184, 0, 0.03) 1px, transparent 1px);
    background-size: 20px 20px;
}
```

## Common VLM Insights

The VLM often catches:
- **Color mismatches** - white text that should be gold/colored
- **Layout drift** - elements too spread out, need containment
- **Conceptual issues** - "floating modules" vs "unified panel"
- **Missing depth** - backgrounds too light, need contrast
- **Separator style** - solid borders vs floating gradient ticks

## Iteration Tracking

Name screenshots by iteration:
- `vs_iteration1.png`
- `vs_iteration2.png`
- `vs_iteration3_final.png`

## Success Criteria

Stop iterating when:
- VLM reports only minor differences
- Visual comparison shows strong match
- Key design elements all present and styled correctly

## Example Session

```
Iteration 1: "Elements look like floating modules"
  → Add vertical separators, remove gaps

Iteration 2: "Title is white, should be gold"
  → Change color to var(--gold-primary)

Iteration 3: "Missing hardware box effect"
  → Add side borders, deepen backgrounds

Done: Converged to reference
```

## Requirements

- Python 3 with Playwright (`pip install playwright && playwright install chromium`)
- PAL MCP server for VLM comparison
- Reference design image
