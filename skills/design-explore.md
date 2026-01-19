# Design Explore

AI-powered design exploration skill. Generates visual directions, facilitates human decisions, produces implementation code.

## Invocation

```
/design-explore [product] [lens]
```

Examples:
- `/design-explore` — Interactive mode, I'll ask for details
- `/design-explore FamilyMeetup "military ops console"`
- `/design-explore MyApp "retro gaming"`

## Prerequisites

**Required environment variable:**
```bash
GOOGLE_API_KEY  # For Nano Banana Pro (gemini-3-pro-image-preview)
```

Source from: `../gridplay-v1/.env` or set directly.

---

# EXECUTION INSTRUCTIONS

When this skill is invoked, I follow these stages exactly. Each stage completes before the next begins.

---

## STAGE 1: SETUP

### Step 1.1: Parse Arguments

Extract from the user's command:
- `PROJECT_NAME` — The product being designed
- `LENS` — The aesthetic filter (e.g., "military ops", "retro gaming", "luxury")

If not provided, use `AskUserQuestion` to gather:

```
Questions:
1. "What product are we designing?" (header: "Product")
2. "What aesthetic lens should filter all designs?" (header: "Lens")
```

### Step 1.2: Select Themes

Default themes (unless user specifies):

| Theme ID | Name | Inspiration |
|----------|------|-------------|
| `cyberpunk` | Cyberpunk | Blade Runner, neon noir |
| `industrial` | Industrial | Alien, utilitarian, warning stripes |
| `minimal` | Minimal | Apple, Linear, clean |
| `brutalist` | Brutalist | Bloomberg Terminal, dense |

### Step 1.3: Cost Confirmation

Calculate cost:
- Phase 1: 1 hero image per theme × $0.14 = ~$0.56 for 4 themes
- Phase 2 (optional): Additional variants at $0.14 each

**MANDATORY: Use AskUserQuestion before proceeding:**

```
Question: "Ready to generate design exploration?"
Header: "Cost"
Options:
- "Yes, generate 4 hero images (~$0.56)"
- "Customize themes first"
- "Cancel"
```

If user selects "Cancel", stop execution.

### Step 1.4: Setup Output Directory

Execute:
```bash
PROJECT="[PROJECT_NAME in lowercase]"
DATE=$(date +%Y%m%d)
OUTPUT_DIR="design-exploration/outputs/${PROJECT}-${DATE}"
mkdir -p "$OUTPUT_DIR"/{cyberpunk,industrial,minimal,brutalist}
echo "Output directory: $OUTPUT_DIR"
```

---

## STAGE 2: IMAGE GENERATION

### Step 2.1: Load API Key

```bash
# Source API key
export GOOGLE_API_KEY=$(grep "^GOOGLE_API_KEY=" ../gridplay-v1/.env 2>/dev/null | cut -d= -f2)
if [ -z "$GOOGLE_API_KEY" ]; then
  echo "ERROR: GOOGLE_API_KEY not found"
  exit 1
fi
echo "API key loaded (${#GOOGLE_API_KEY} chars)"
```

### Step 2.2: Generate Hero Images (Phase 1)

For each theme, generate ONE hero image using this exact API call:

```bash
THEME="[theme_id]"
THEME_NAME="[theme_name]"
PROMPT="Create a UI design mockup for ${PROJECT_NAME} - a [brief product description].

Design Direction: ${THEME_NAME}
Aesthetic Lens: ${LENS}

Screen: Main dashboard / hero view (1920x1080)
Show: Primary interface with key functionality visible

Style guidelines for ${THEME_NAME}:
[Include theme-specific style notes - see THEME PROMPTS section below]

Requirements:
- Dark mode interface
- High contrast, readable text
- Show realistic UI elements (cards, buttons, navigation)
- No device frames, just the UI

Make it feel like a professional, production-ready interface."

# API Call
response=$(curl -s "https://generativelanguage.googleapis.com/v1beta/models/gemini-3-pro-image-preview:generateContent?key=$GOOGLE_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"contents\": [{
      \"parts\": [{
        \"text\": \"$PROMPT\"
      }]
    }],
    \"generationConfig\": {
      \"responseModalities\": [\"TEXT\", \"IMAGE\"]
    }
  }")

# Save image
echo "$response" | jq -r '.candidates[0].content.parts[] | select(.inlineData) | .inlineData.data' | base64 -d > "$OUTPUT_DIR/$THEME/hero.png"

if [ -s "$OUTPUT_DIR/$THEME/hero.png" ]; then
  echo "OK: $THEME/hero.png ($(du -h "$OUTPUT_DIR/$THEME/hero.png" | cut -f1))"
else
  echo "FAILED: $THEME"
fi
```

### Step 2.3: Display Results

After generating all hero images, use the `Read` tool to display each image to the user:

```
Read: $OUTPUT_DIR/cyberpunk/hero.png
Read: $OUTPUT_DIR/industrial/hero.png
Read: $OUTPUT_DIR/minimal/hero.png
Read: $OUTPUT_DIR/brutalist/hero.png
```

Present them with labels:
- "**CYBERPUNK** — Neon noir, atmospheric"
- "**INDUSTRIAL** — Utilitarian, warning stripes"
- "**MINIMAL** — Clean, spacious"
- "**BRUTALIST** — Dense, functional"

---

## STAGE 3: AI EVALUATION

### Step 3.1: Run Multi-Model Evaluation

Use `mcp__pal__consensus` to get evaluations from multiple AI models:

```
Models: ["google/gemini-2.5-pro", "anthropic/claude-sonnet-4"]
Prompt: "Evaluate these 4 design directions for [PROJECT_NAME] with lens '[LENS]'.

Rate each on:
1. Visual Impact (1-10)
2. Functional Clarity (1-10)
3. Brand Alignment (1-10)
4. Implementation Feasibility (1-10)
5. Accessibility (1-10)

Then recommend:
- Your top pick and why
- A synthesis combining the best elements from multiple directions
- What to avoid from rejected directions"
```

### Step 3.2: Synthesize Results

Compile the evaluation into a summary:
- Scores table
- Each evaluator's pick
- Proposed synthesis (what to take from each theme)
- Rejected elements

---

## STAGE 4: HUMAN DECISION

### Step 4.1: Present Decision

Use `AskUserQuestion` with these options:

```
Question: "Which direction should we pursue?"
Header: "Direction"
Options:
- "Accept AI Synthesis (recommended)" — Combine best elements
- "Pure Cyberpunk" — Use cyberpunk direction as-is
- "Pure Industrial" — Use industrial direction as-is
- "Pure Minimal" — Use minimal direction as-is
- "Pure Brutalist" — Use brutalist direction as-is
- "Generate more variants" — Expand exploration ($0.14/image)
- "Start over with different themes"
```

### Step 4.2: Handle "Generate More"

If user wants more variants, use `AskUserQuestion`:

```
Question: "Which themes need more variants?"
Header: "Expand"
MultiSelect: true
Options:
- "Cyberpunk (+3 variants, ~$0.42)"
- "Industrial (+3 variants, ~$0.42)"
- "Minimal (+3 variants, ~$0.42)"
- "Brutalist (+3 variants, ~$0.42)"
```

Then generate additional variants:
- `detail.png` — Expanded/detail view
- `mobile.png` — Mobile responsive view
- `component.png` — Key component (card, button)

Return to Stage 4.1 after generating.

---

## STAGE 5: IMPLEMENTATION

Based on the user's choice, generate implementation artifacts.

### Step 5.1: Create Design System CSS

Write to `$OUTPUT_DIR/design-system.css`:

```css
/* [PROJECT_NAME] Design System
   Direction: [CHOSEN_DIRECTION]
   Generated: [DATE]
*/

:root {
  /* Colors - extracted from chosen direction */
  --color-bg-primary: #0A0F14;
  --color-bg-surface: #16161f;
  --color-critical: #FF4242;
  --color-urgent: #FFD03D;
  --color-info: #00D8FF;
  --color-success: #28A745;

  /* Typography */
  --font-display: 'Orbitron', sans-serif;
  --font-ui: 'IBM Plex Mono', monospace;

  /* Spacing */
  --space-xs: 4px;
  --space-sm: 8px;
  --space-md: 16px;
  --space-lg: 24px;
  --space-xl: 32px;
}
```

### Step 5.2: Create Design Tokens JSON

Write to `$OUTPUT_DIR/design-tokens.json`:

```json
{
  "project": "[PROJECT_NAME]",
  "direction": "[CHOSEN_DIRECTION]",
  "colors": { ... },
  "typography": { ... },
  "spacing": { ... }
}
```

### Step 5.3: Create Implementation Guide

Write to `$OUTPUT_DIR/IMPLEMENTATION.md`:

```markdown
# [PROJECT_NAME] Design Implementation Guide

## Chosen Direction: [DIRECTION]

## What to Implement
- [List of adopted elements]

## What to Avoid
- [List of rejected elements]

## CSS Variables
See `design-system.css`

## Component Guidelines
[Specific guidance for cards, buttons, navigation, etc.]
```

### Step 5.4: Generate Summary Report

Write to `$OUTPUT_DIR/EXPLORATION_REPORT.md` with:
- Executive summary
- All theme scores
- Evaluator feedback
- Final decision rationale
- Cost breakdown

### Step 5.5: Final Confirmation

Tell the user:
```
Design exploration complete!

Output files:
- design-system.css — CSS variables ready to use
- design-tokens.json — For design tool import
- IMPLEMENTATION.md — Guidelines for developers
- EXPLORATION_REPORT.md — Full exploration summary

Total cost: $[X.XX]

Next steps:
1. Review the generated mockups
2. Import design-system.css into your project
3. Use the implementation guide when building components
```

---

# THEME PROMPTS

## Cyberpunk (Blade Runner)
```
Neon noir aesthetic with cyan and magenta accents against deep blacks.
Rain-soaked atmosphere bleeding through the UI.
Holographic elements, scan lines as subtle texture.
Japanese typography mixed with English where appropriate.
Cards should feel like LAPD replicant tracking terminals.
```

## Industrial (Alien/Weyland-Yutani)
```
1980s CRT aesthetic with green/amber monochrome accents.
Chunky industrial hardware feel with yellow/black warning stripes on critical elements.
Colonial Marines operations terminal vibe.
Thick beveled buttons with physical click affordance.
Segmented progress indicators, loading percentages.
```

## Minimal (Interstellar/NASA)
```
Clean, scientifically precise interface.
Predominantly dark with strategic accent colors for semantics only.
Apollo-era mission control calm authority.
Mathematical precision in spacing and alignment.
Generous whitespace, clear hierarchy.
Data visualization that looks like telemetry.
```

## Brutalist (Bloomberg Terminal)
```
Maximum information density, no decoration.
Monospace typography throughout.
Dense grid layout with clear borders.
Function over form, every pixel earns its place.
No rounded corners, no shadows, no gradients.
Colors only for semantic meaning (red=bad, green=good).
```

---

# ERROR HANDLING

## API Key Missing
```
ERROR: GOOGLE_API_KEY not found.
Please set it: export GOOGLE_API_KEY=your_key
Or source from: source ../gridplay-v1/.env
```

## Image Generation Failed
If an image fails to generate:
1. Report the error to user
2. Ask if they want to retry or skip that theme
3. Continue with remaining themes

## User Cancels
At any AskUserQuestion, if user wants to stop:
- Save any generated images
- Report what was completed
- Provide path to outputs
