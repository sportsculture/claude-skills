---
name: nbp
description: Generate images with Nano Banana Pro (text-to-image)
---

# /nbp — Nano Banana Pro Text-to-Image

Generate N image variations from a text prompt using Gemini image generation.

## Invocation

```
/nbp <prompt> [--n 3] [--dir design_targets] [--prefix my_image]
```

Examples:
- `/nbp "Dark UI mockup for a crisis decision app" --prefix triage --dir design_targets`
- `/nbp "Cyberpunk dashboard with neon cyan accents" --n 2 --prefix comparator`
- `/nbp` — Interactive mode, will ask for prompt

## Prerequisites

- `VERTEX_API_KEY` in `.env` (preferred — Vertex AI, no RPD cap)
- Falls back to `GOOGLE_API_KEY` if no Vertex key found

---

# EXECUTION INSTRUCTIONS

## Step 1: Parse Arguments

Extract from `$ARGUMENTS`:
- `PROMPT` — The image generation prompt (required)
- `N` — Number of variations to generate (default: 3)
- `DIR` — Output directory (default: current directory)
- `PREFIX` — Filename prefix (default: `image`)

If `$ARGUMENTS` is empty or has no prompt, use `AskUserQuestion`:
```
Question: "What image should I generate?"
Header: "Prompt"
```

Parse flags: `--n <count>`, `--dir <path>`, `--prefix <name>`

Output filenames follow pattern: `{PREFIX}_v{N}.{ext}` where ext is determined by the API response mime type (usually `.jpg`). Example: `triage_v1.jpg`, `triage_v2.jpg`, `triage_v3.jpg`

## Step 2: Setup

```bash
mkdir -p "$DIR"
```

## Step 3: Generate Images

**IMPORTANT: Always use Python for API calls.** Shell `curl` cannot handle large payloads.

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

prompt = """$PROMPT"""
n = $N
output_dir = "$DIR"
prefix = "$PREFIX"

url = f"{api_base}/gemini-3-pro-image-preview:generateContent?key={api_key}"

payload = json.dumps({
    "contents": [{"parts": [{"text": prompt}]}],
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
                name = f"{prefix}_v{i}.{ext}"
                path = os.path.join(output_dir, name)
                img_bytes = base64.b64decode(part["inlineData"]["data"])
                with open(path, "wb") as f:
                    f.write(img_bytes)
                print(f"OK: {name} ({len(img_bytes)//1024}K)")
                generated.append({"name": name, "label": f"v{i}"})
                break
        else:
            print(f"FAILED: v{i} — no image data in response")
            for part in data.get("candidates", [{}])[0].get("content", {}).get("parts", []):
                if "text" in part:
                    print(f"  Response: {part['text'][:200]}")
    except Exception as e:
        print(f"ERROR: v{i} — {e}")

# Generate HTML gallery with select + copy
if generated:
    cards = ""
    for g in generated:
        cards += f'<div class="card" data-name="{g["name"]}" data-label="{g["label"]}" onclick="selectCard(this)"><img src="{g["name"]}"><div class="label">{g["label"]}<button class="pick-btn" onclick="event.stopPropagation();pickCard(this.closest(\'.card\'))">Select</button></div></div>\n'
    html = f"""<!DOCTYPE html>
<html><head><title>{prefix} — NBP Gallery</title>
<style>
body {{ background:#111; color:#fbbf24; font-family:monospace; padding:20px; margin:0; }}
h1 {{ font-size:18px; letter-spacing:4px; text-transform:uppercase; border-bottom:1px solid #333; padding-bottom:8px; }}
.toolbar {{ display:flex; gap:12px; align-items:center; margin-top:12px; }}
.toolbar button {{ background:#1a1a2e; color:#fbbf24; border:1px solid #333; border-radius:6px; padding:6px 16px; font-family:monospace; font-size:13px; cursor:pointer; transition:all 0.15s; }}
.toolbar button:hover {{ border-color:#fbbf24; }}
.toolbar button:disabled {{ opacity:0.3; cursor:default; }}
.toolbar .status {{ font-size:12px; color:#666; }}
.grid {{ display:grid; grid-template-columns:repeat(auto-fit,minmax(400px,1fr)); gap:16px; margin-top:16px; }}
.card {{ background:#1a1a2e; border:2px solid #333; border-radius:8px; overflow:hidden; cursor:pointer; transition:border-color 0.15s, box-shadow 0.15s; }}
.card:hover {{ border-color:#555; }}
.card.selected {{ border-color:#fbbf24; box-shadow:0 0 20px rgba(251,191,36,0.15); }}
.card img {{ width:100%; display:block; }}
.card .label {{ padding:8px 12px; font-size:14px; color:#fbbf24; display:flex; justify-content:space-between; align-items:center; }}
.pick-btn {{ background:transparent; color:#666; border:1px solid #444; border-radius:4px; padding:3px 10px; font-family:monospace; font-size:11px; cursor:pointer; transition:all 0.15s; }}
.pick-btn:hover {{ color:#fbbf24; border-color:#fbbf24; }}
.card.selected .pick-btn {{ background:#fbbf24; color:#111; border-color:#fbbf24; }}
.copied {{ color:#4ade80 !important; }}
</style></head><body>
<h1>{prefix} — Variations</h1>
<div class="toolbar">
  <button id="copyBtn" onclick="copySelection()" disabled>Copy selection</button>
  <span id="status" class="status"></span>
</div>
<div class="grid">{cards}</div>
<script>
let selected = null;
function selectCard(card) {{
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
    gallery_path = os.path.join(output_dir, f"{prefix}_gallery.html")
    with open(gallery_path, "w") as f:
        f.write(html)
    print(f"GALLERY: {gallery_path}")
PYEOF
```

## Step 4: Display Results

### MANDATORY: Open HTML gallery in browser

**This step is NOT optional.** Always open the gallery in the browser after generation. The user reviews images in the browser, not inline.

```bash
xdg-open "$DIR/{PREFIX}_gallery.html"
```

### Show inline for Claude context

Also use the `Read` tool to show EACH generated image to the user inline, labeled. Use `Glob` to find the actual files since the extension may vary (`.jpg` or `.png`):
```
**{PREFIX} v1:**
Read: $DIR/{PREFIX}_v1.{ext}

**{PREFIX} v2:**
Read: $DIR/{PREFIX}_v2.{ext}

(etc.)
```

Report summary: number generated, output directory, filenames, and gallery HTML path.

### When generating images outside /nbp

Even when generating images manually (not via this skill), ALWAYS create an HTML gallery and open it with `xdg-open`. This is a user requirement — Mac reviews all image output in the browser.

---

# ERROR HANDLING

## API Key Missing
```
VERTEX_API_KEY or GOOGLE_API_KEY not found. Add one to .env:
echo "VERTEX_API_KEY=your_key" >> .env  # Preferred — Vertex AI, no RPD cap
echo "GOOGLE_API_KEY=your_key" >> .env  # Fallback — AI Studio, 250 RPD cap
```

## Generation Failed
If an individual image fails:
1. Report the error
2. Continue generating remaining images
3. At the end, report which succeeded and which failed

## Rate Limited (429)
Wait 10 seconds and retry the failed image once.
