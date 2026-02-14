---
name: nbp-refine
description: Refine an image with Nano Banana Pro using a reference image + prompt
---

# /nbp-refine — Reference-Based Image Refinement

Feed an existing image back to Nano Banana Pro with a refinement prompt. Generates N variations that stay grounded in the reference while applying requested changes.

## Invocation

```
/nbp-refine <reference_image_path> <refinement_prompt> [--n 2] [--dir design_targets]
```

Examples:
- `/nbp-refine design_targets/triage_v2.png "Make the secondary cards more muted, add a timer" --dir design_targets`
- `/nbp-refine design_targets/comparator_v2.png "Reduce cyan glow intensity" --n 3`
- `/nbp-refine` — Interactive mode, will ask for image path and prompt

## Prerequisites

- `VERTEX_API_KEY` in `.env` (preferred — Vertex AI, no RPD cap)
- Falls back to `GOOGLE_API_KEY` if no Vertex key found
- An existing reference image to refine

---

# EXECUTION INSTRUCTIONS

## Step 1: Parse Arguments

Extract from `$ARGUMENTS`:
- `REFERENCE_PATH` — Path to the reference image (required)
- `REFINEMENT_PROMPT` — What to change (required)
- `N` — Number of variations to generate (default: 2)
- `DIR` — Output directory (default: same directory as reference image)

If arguments are missing, use `AskUserQuestion` to gather them.

### Ancestry Naming

Output filenames encode the refinement lineage:
- Reference is `triage_v2.jpg` → outputs are `triage_v2_r1.{ext}`, `triage_v2_r2.{ext}`
- Reference is `triage_v2_r1.jpg` → outputs are `triage_v2_r1_r1.{ext}`, `triage_v2_r1_r2.{ext}`

**Rule:** Strip the file extension from the reference filename, append `_r{N}.{ext}` where ext is determined by the API response mime type.

## Step 2: Validate Reference Image

```bash
if [ ! -f "$REFERENCE_PATH" ]; then
  echo "ERROR: Reference image not found: $REFERENCE_PATH"
  exit 1
fi
echo "Reference: $REFERENCE_PATH ($(du -h "$REFERENCE_PATH" | cut -f1))"
```

## Step 3: Generate Refined Images

**IMPORTANT: Always use Python for API calls.** Reference images are large and will exceed shell argument limits if passed via curl.

```python
python3 << 'PYEOF'
import base64, json, urllib.request, sys, os

# Load API key from .env — prefer VERTEX_API_KEY (no RPD cap), fall back to GOOGLE_API_KEY
api_key = None
api_base = None
for env_path in [".env", "../.env"]:
    if os.path.exists(env_path):
        for line in open(env_path):
            if line.startswith("VERTEX_API_KEY=") and not api_key:
                api_key = line.strip().split("=", 1)[1]
                api_base = "https://aiplatform.googleapis.com/v1/publishers/google/models"
            elif line.startswith("GOOGLE_API_KEY=") and not api_base:
                api_key = line.strip().split("=", 1)[1]
                api_base = "https://generativelanguage.googleapis.com/v1beta/models"
    if api_key:
        break

if not api_key:
    print("ERROR: VERTEX_API_KEY or GOOGLE_API_KEY not found in .env")
    sys.exit(1)

ref_path = "$REFERENCE_PATH"
refinement_prompt = """$REFINEMENT_PROMPT"""
n = $N
output_dir = "$DIR"

# Read reference image and detect mime type
ref_ext = os.path.splitext(ref_path)[1].lower()
ref_mime = "image/jpeg" if ref_ext in [".jpg", ".jpeg"] else "image/png"
with open(ref_path, "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

# Compute ancestry-based output names
ref_basename = os.path.splitext(os.path.basename(ref_path))[0]  # e.g., "triage_v2"

url = f"{api_base}/gemini-3-pro-image-preview:generateContent?key={api_key}"

payload = json.dumps({
    "contents": [{"parts": [
        {"inlineData": {"mimeType": ref_mime, "data": img_b64}},
        {"text": refinement_prompt}
    ]}],
    "generationConfig": {"responseModalities": ["TEXT", "IMAGE"]}
}).encode()

generated = []
for i in range(1, n + 1):
    try:
        req = urllib.request.Request(url, data=payload, headers={"Content-Type": "application/json"})
        resp = urllib.request.urlopen(req, timeout=120)
        data = json.loads(resp.read())

        for part in data["candidates"][0]["content"]["parts"]:
            if "inlineData" in part:
                mime = part["inlineData"].get("mimeType", "image/jpeg")
                ext = "jpg" if "jpeg" in mime else "png"
                name = f"{ref_basename}_r{i}.{ext}"
                path = os.path.join(output_dir, name)
                img_bytes = base64.b64decode(part["inlineData"]["data"])
                with open(path, "wb") as f:
                    f.write(img_bytes)
                print(f"OK: {name} ({len(img_bytes)//1024}K)")
                generated.append({"name": name, "label": f"r{i}"})
                break
        else:
            print(f"FAILED: r{i} — no image data in response")
    except Exception as e:
        print(f"ERROR: r{i} — {e}")

# Generate HTML gallery with reference + refinements + select/copy
if generated:
    ref_filename = os.path.basename(ref_path)
    cards = f'<div class="card ref"><img src="{ref_filename}"><div class="label">REFERENCE</div></div>\n'
    for g in generated:
        cards += f'<div class="card" data-name="{g["name"]}" data-label="{g["label"]}" onclick="selectCard(this)"><img src="{g["name"]}"><div class="label">{g["label"]}<button class="pick-btn" onclick="event.stopPropagation();pickCard(this.closest(\'.card\'))">Select</button></div></div>\n'
    html = f"""<!DOCTYPE html>
<html><head><title>{ref_basename} — NBP Refinement</title>
<style>
body {{ background:#111; color:#fbbf24; font-family:monospace; padding:20px; margin:0; }}
h1 {{ font-size:18px; letter-spacing:4px; text-transform:uppercase; border-bottom:1px solid #333; padding-bottom:8px; }}
.prompt {{ color:#888; font-size:12px; margin-top:4px; }}
.toolbar {{ display:flex; gap:12px; align-items:center; margin-top:12px; }}
.toolbar button {{ background:#1a1a2e; color:#fbbf24; border:1px solid #333; border-radius:6px; padding:6px 16px; font-family:monospace; font-size:13px; cursor:pointer; transition:all 0.15s; }}
.toolbar button:hover {{ border-color:#fbbf24; }}
.toolbar button:disabled {{ opacity:0.3; cursor:default; }}
.toolbar .status {{ font-size:12px; color:#666; }}
.grid {{ display:grid; grid-template-columns:repeat(auto-fit,minmax(400px,1fr)); gap:16px; margin-top:16px; }}
.card {{ background:#1a1a2e; border:2px solid #333; border-radius:8px; overflow:hidden; cursor:pointer; transition:border-color 0.15s, box-shadow 0.15s; }}
.card:hover {{ border-color:#555; }}
.card.ref {{ border-color:#555; cursor:default; }}
.card.selected {{ border-color:#fbbf24; box-shadow:0 0 20px rgba(251,191,36,0.15); }}
.card img {{ width:100%; display:block; }}
.card .label {{ padding:8px 12px; font-size:14px; color:#fbbf24; display:flex; justify-content:space-between; align-items:center; }}
.pick-btn {{ background:transparent; color:#666; border:1px solid #444; border-radius:4px; padding:3px 10px; font-family:monospace; font-size:11px; cursor:pointer; transition:all 0.15s; }}
.pick-btn:hover {{ color:#fbbf24; border-color:#fbbf24; }}
.card.selected .pick-btn {{ background:#fbbf24; color:#111; border-color:#fbbf24; }}
.copied {{ color:#4ade80 !important; }}
</style></head><body>
<h1>{ref_basename} — Refinement</h1>
<div class="prompt">{refinement_prompt[:200]}</div>
<div class="toolbar">
  <button id="copyBtn" onclick="copySelection()" disabled>Copy selection</button>
  <span id="status" class="status"></span>
</div>
<div class="grid">{cards}</div>
<script>
let selected = null;
function selectCard(card) {{
  if (card.classList.contains('ref')) return;
  document.querySelectorAll('.card').forEach(c => c.classList.remove('selected'));
  card.classList.add('selected');
  selected = card;
  document.getElementById('copyBtn').disabled = false;
  document.getElementById('status').textContent = card.dataset.label + ' selected';
}}
function pickCard(card) {{
  selectCard(card);
  copySelection();
}}
function copySelection() {{
  if (!selected) return;
  const text = selected.dataset.label;
  navigator.clipboard.writeText(text).then(() => {{
    const btn = document.getElementById('copyBtn');
    const status = document.getElementById('status');
    btn.textContent = 'Copied!';
    btn.classList.add('copied');
    status.textContent = selected.dataset.label + ' — ' + selected.dataset.name;
    setTimeout(() => {{ btn.textContent = 'Copy selection'; btn.classList.remove('copied'); }}, 2000);
  }});
}}
</script>
</body></html>"""
    gallery_path = os.path.join(output_dir, f"{ref_basename}_refine_gallery.html")
    with open(gallery_path, "w") as f:
        f.write(html)
    print(f"GALLERY: {gallery_path}")
PYEOF
```

## Step 4: Display Results

### Open HTML gallery in browser

```bash
xdg-open "$DIR/{ref_basename}_refine_gallery.html"
```

### Show inline for Claude context

Also use the `Read` tool to show EACH refined image to the user inline, labeled with ancestry:
```
**{ref_basename} > r1:**
Read: $DIR/{ref_basename}_r1.{ext}

**{ref_basename} > r2:**
Read: $DIR/{ref_basename}_r2.{ext}
```

Report: reference image used, refinement prompt summary, output filenames, and gallery HTML path.

---

# ERROR HANDLING

## Reference Image Not Found
Ask user for the correct path.

## API Key Missing
```
VERTEX_API_KEY or GOOGLE_API_KEY not found. Add one to .env.
```

## Generation Failed
If an individual refinement fails:
1. Report the error
2. Continue generating remaining variations
3. Suggest simplifying the refinement prompt if all fail

## Rate Limited (429)
Wait 10 seconds and retry once.
