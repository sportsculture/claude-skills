---
name: design-target
description: Full design target workflow — generate, review, refine, pick winner
---

# /design-target — Design Target Generation Workflow

Orchestrates the full design target loop: generate initial variations, get human feedback, iteratively refine with reference images, declare a winner.

## Invocation

```
/design-target <morph_name> [--prompt "custom prompt"] [--dir design_targets]
```

Examples:
- `/design-target triage`
- `/design-target comparator --prompt "Bloomberg Terminal style weighted matrix"`
- `/design-target simulator --dir design_targets`

## Prerequisites

- `VERTEX_API_KEY` in `.env` (preferred — Vertex AI, no RPD cap)
- Falls back to `GOOGLE_API_KEY` if no Vertex key found
- Build prompt document at `docs/build_prompts.md` (for default prompts)

---

# EXECUTION INSTRUCTIONS

## Stage 1: Setup

### Step 1.1: Parse Arguments

Extract from `$ARGUMENTS`:
- `MORPH_NAME` — The morph to generate a design target for (required)
- `PROMPT` — Custom generation prompt (optional — if not provided, extract from build prompts doc)
- `DIR` — Output directory (default: `design_targets`)

### Step 1.2: Load Prompt

If no custom `--prompt` provided:
1. Read `docs/build_prompts.md`
2. Find the Phase 0 prompt for the given morph name (P0.1 for triage, P0.2 for comparator, etc.)
3. Extract the text inside the code block as the generation prompt

### Step 1.3: Confirm with User

Use `AskUserQuestion`:
```
Question: "Generate 3 variations for {MORPH_NAME} design target? (~$0.42)"
Header: "Generate"
Options:
- "Yes, generate 3 variations"
- "Customize prompt first"
- "Cancel"
```

If "Customize prompt first": show the prompt text, let user edit, then proceed.

---

## Stage 2: Initial Variations

### Step 2.1: Generate 3 Variations

Invoke `/nbp` logic (do NOT invoke as a slash command — execute the same Python generation pattern inline):
- Prompt: the loaded/custom prompt
- N: 3
- Dir: `$DIR`
- Prefix: `$MORPH_NAME`

This produces: `{morph}_v1.png`, `{morph}_v2.png`, `{morph}_v3.png`

### Step 2.2: Present Variations

The Python generation script (nbp pattern) produces an HTML gallery automatically. Open it in the browser:

```bash
xdg-open "$DIR/{MORPH_NAME}_gallery.html"
```

Also use the `Read` tool to display all 3 images inline for Claude context:

```
**{MORPH_NAME} v1:**
Read: {DIR}/{MORPH_NAME}_v1.{ext}

**{MORPH_NAME} v2:**
Read: {DIR}/{MORPH_NAME}_v2.{ext}

**{MORPH_NAME} v3:**
Read: {DIR}/{MORPH_NAME}_v3.{ext}
```

### Step 2.3: Ask User to Pick Base

Use `AskUserQuestion`:
```
Question: "Which variation is the strongest base for refinement?"
Header: "{MORPH_NAME}"
Options:
- "v1" with description of its key characteristics
- "v2" with description
- "v3" with description
MultiSelect: false
```

The user may respond with:
- **A pick** — proceed to refinement
- **A pick + feedback** — proceed to refinement with their notes as the refinement prompt
- **Detailed feedback only** — infer which they liked most from context, confirm, then proceed

---

## Stage 3: Refinement Loop

### Step 3.1: Collect Refinement Feedback

If the user provided feedback with their pick, use it as the refinement prompt.

If not, use `AskUserQuestion`:
```
Question: "What should be refined in {chosen_version}?"
Header: "Refine"
```

### Step 3.2: Generate Refined Variations

Invoke `/nbp-refine` logic (execute the Python reference-image pattern inline):
- Reference: `{DIR}/{chosen_version}.png`
- Prompt: the user's refinement feedback
- N: 2

This produces ancestry-named files: `{morph}_{parent}_r1.png`, `{morph}_{parent}_r2.png`

### Step 3.3: Present Refined Variations

The Python refinement script (nbp-refine pattern) produces an HTML gallery with reference + refinements. Open it:

```bash
xdg-open "$DIR/{parent}_refine_gallery.html"
```

Also display inline for Claude context:
```
**{parent} > r1:**
Read: {DIR}/{parent}_r1.{ext}

**{parent} > r2:**
Read: {DIR}/{parent}_r2.{ext}
```

### Step 3.4: Decision Point

Use `AskUserQuestion`:
```
Question: "Pick the winner or refine further?"
Header: "Decision"
Options:
- "r1 is the winner"
- "r2 is the winner"
- "One more round" — feed the better one back with more tweaks
MultiSelect: false
```

**If "One more round":** Go back to Step 3.1, using the chosen refinement as the new reference. The ancestry chain grows (e.g., `triage_v2_r1_r1.png`).

**If winner declared:** Proceed to Stage 4.

---

## Stage 4: Record Winner

### Step 4.1: Log the Result

Report to the user:
```
DESIGN TARGET COMPLETE: {MORPH_NAME}

Winner: {winning_filename}
Lineage: {ancestry chain, e.g., "v2 → v6 → r1"}
Location: {DIR}/{winning_filename}

Design notes for implementation:
- {any notes the user provided during feedback rounds}
```

### Step 4.2: Update Task (if applicable)

If there's an active task for this morph, mark it as completed with the winner info in the description.

---

# DESIGN NOTES COLLECTION

Throughout the loop, collect and accumulate any implementation-relevant notes the user provides. These are NOT visual changes for the image — they're notes for when the morph is built in code. Examples:
- "Timer should shift to past tense when it hits zero"
- "Scores need hover drill-down showing weighted contributions"
- "Slider should snap to 5% increments"

Include all collected design notes in the Stage 4 report.

---

# ERROR HANDLING

## Build Prompts Not Found
If `docs/build_prompts.md` doesn't exist:
1. Search for files matching `*build_prompts*` in the docs directory
2. If found, use that file instead
3. If still not found and no custom prompt provided, ask user for the generation prompt directly

## Generation Failures
If image generation fails mid-loop:
1. Report which images failed
2. Ask user if they want to retry or continue with what succeeded
3. Never silently skip failures

## User Cancels
At any AskUserQuestion, if user wants to stop:
- Save all generated images
- Report what was completed and where files are
- Note which morph still needs a design target
