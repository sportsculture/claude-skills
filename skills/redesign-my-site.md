---
description: AI-powered site audit and redesign with annotated screenshots and mockup generation
---

# Redesign My Site

AI-powered site audit and redesign skill. Analyzes screenshots against purpose, annotates problems, generates redesigned mockups.

## Usage

```
/redesign-my-site [screenshot_paths...]
```

Examples:
- `/redesign-my-site` — Interactive mode, I'll ask for screenshots
- `/redesign-my-site ~/screenshots/home.png ~/screenshots/about.png`

## Prerequisites

**Required environment variable:**
```bash
GOOGLE_API_KEY  # For Nano Banana Pro (gemini-3-pro-image-preview)
```

**Required tools:**
- Python 3 with Pillow (`pip install pillow`)

---

# EXECUTION INSTRUCTIONS

When this skill is invoked, follow these stages exactly.

---

## STAGE 1: INTAKE

### Step 1.1: Gather Screenshots

If screenshots not provided, ask user for paths.

Verify each screenshot exists and check resolution:
```bash
for img in $SCREENSHOTS; do
  if [ -f "$img" ]; then
    dimensions=$(identify -format "%wx%h" "$img" 2>/dev/null)
    echo "OK: $img ($dimensions)"
  else
    echo "NOT FOUND: $img"
  fi
done
```

### Step 1.2: Understand Purpose

Ask user:
- **Purpose**: Generate leads, sell products, inform/educate, build trust?
- **Target audience**: Who is the customer?
- **CTA**: What ONE action should visitors take?

### Step 1.3: Setup Output Directory

```bash
PROJECT="redesign"
DATE=$(date +%Y%m%d)
OUTPUT_DIR="redesign-audit/${PROJECT}-${DATE}"
mkdir -p "$OUTPUT_DIR"/{original,critique,mockups,implementation}
```

### Step 1.4: Cost Confirmation

Confirm with user before proceeding (~$1.50-3.00 for full audit).

---

## STAGE 2: ANALYSIS

### Step 2.1: Vision Analysis

For each screenshot, analyze using Claude's vision capabilities against:
1. Does the visual hierarchy guide users to the primary CTA?
2. Is the value proposition immediately clear?
3. Are there accessibility issues (contrast, text size)?
4. Is there visual clutter that distracts from the goal?
5. Does the design build trust and credibility?
6. Is the mobile experience considered?

Return structured analysis with issues, severity, and fixes.

### Step 2.2: Save Analysis

Save the JSON analysis for each screenshot:
```bash
echo "$ANALYSIS_JSON" > "$OUTPUT_DIR/critique/analysis_$(basename $IMG .png).json"
```

---

## STAGE 3: CRITIQUE (Annotation)

### Step 3.1: Generate Annotated Images

Create annotated versions highlighting issues by severity:
- **Critical**: Red overlay
- **Major**: Orange overlay
- **Minor**: Yellow overlay

Use Python with Pillow to draw issue zones and numbers.

### Step 3.2: Display Annotated Images

For each issue, explain:
- **What's wrong:** [description]
- **Why it matters:** [impact on purpose/conversion]
- **What to do:** [fix]

---

## STAGE 4: REDESIGN

### Step 4.1: Synthesize Redesign Brief

Based on the analysis, create a redesign brief addressing critical issues.

### Step 4.2: Generate Redesigned Mockups

Use Nano Banana Pro (gemini-3-pro-image-preview) to generate redesigns.

**CRITICAL: Use short, focused prompts. Long prompts timeout or produce poor results.**

Generate 2 variations:
- **Variation A:** Dashboard style with sidebar navigation
- **Variation B:** Full-page with before/after comparison

### Step 4.3: AI Evaluation

Use `mcp__pal__consensus` to evaluate redesigns against original on:
1. Purpose Alignment (1-10)
2. Visual Hierarchy (1-10)
3. Trust & Credibility (1-10)
4. Implementation Feasibility (1-10)

### Step 4.4: Human Decision

Ask user which direction they prefer.

---

## STAGE 5: DELIVERY

### Step 5.1: Generate Audit Report

Write to `$OUTPUT_DIR/AUDIT_REPORT.md` with:
- Executive summary and score
- Critical issues with fixes
- Strengths to preserve
- Detailed analysis per screenshot
- Recommended redesign with rationale
- Implementation roadmap
- Cost summary

### Step 5.2: Generate Implementation CSS

Extract design system from winning mockup:
```css
:root {
  --color-primary: #...;
  --color-secondary: #...;
  --color-background: #...;
  --font-heading: '...', sans-serif;
  /* ... */
}
```

### Step 5.3: Final Summary

Report output files and next steps.

---

# ERROR HANDLING

## API Key Missing
```
ERROR: GOOGLE_API_KEY not found.
Please set it: export GOOGLE_API_KEY=your_key
```

## Image Generation Failed
Report error, offer retry or skip.

## Low Resolution Screenshots
```
WARNING: Screenshot resolution is low.
For best results, use screenshots at least 1920px wide.
```

---

# PRICING GUIDANCE

| Component | Est. Cost |
|-----------|-----------|
| Analysis (per screenshot) | ~$0.02 |
| Annotated critique | ~$0.00 (local Python) |
| Mockup generation (per image) | ~$0.14 |
| AI evaluation | ~$0.06 |
| **Typical audit (3 screenshots, 2 mockups)** | **~$0.40** |
