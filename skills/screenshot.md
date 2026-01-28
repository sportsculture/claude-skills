---
description: Take a screenshot of a localhost page to verify visual changes
---

# Screenshot Skill

Take a screenshot of a localhost page to verify visual changes.

## System Requirements

- Desktop: COSMIC (System76), GNOME, or other Wayland/X11 compositor
- Tools: `grim` + `slurp` (Wayland) or `scrot`/`gnome-screenshot` (X11)

## Usage

```
/screenshot [path] [port]
/screenshot --calibrate
```

## Setup

Create the screenshot script at `~/bin/screenshot-page`:

```bash
#!/bin/bash
# Screenshot a browser window region

CONFIG_FILE="$HOME/.config/screenshot-page.conf"
OUTPUT_DIR="/tmp"

# Check for calibration mode
if [ "$1" = "--calibrate" ]; then
    echo "Draw a rectangle around the browser window..."
    GEOMETRY=$(slurp 2>/dev/null)
    if [ -n "$GEOMETRY" ]; then
        echo "$GEOMETRY" > "$CONFIG_FILE"
        echo "Saved geometry: $GEOMETRY"
    else
        echo "Calibration cancelled"
        exit 1
    fi
    exit 0
fi

# Parse arguments
PATH_ARG="${1:-/}"
PORT="${2:-4001}"

# Load saved geometry
if [ -f "$CONFIG_FILE" ]; then
    GEOMETRY=$(cat "$CONFIG_FILE")
else
    echo "No calibration found. Run: $0 --calibrate"
    exit 1
fi

# Open page in Firefox
firefox "http://localhost:$PORT$PATH_ARG" &
sleep 2

# Take screenshot
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
OUTPUT="$OUTPUT_DIR/page_$TIMESTAMP.png"

grim -g "$GEOMETRY" "$OUTPUT" 2>/dev/null

if [ -f "$OUTPUT" ]; then
    echo "$OUTPUT"
else
    echo "Screenshot failed. Try recalibrating: $0 --calibrate"
    exit 1
fi
```

Make it executable:
```bash
chmod +x ~/bin/screenshot-page
```

## Execution

### Step 1: Run the script

```bash
~/bin/screenshot-page "{PATH}" {PORT}
```

Default: path `/`, port `4001`

### Step 2: Check output

If the script outputs a path like `/tmp/page_TIMESTAMP.png`, use Read to display it.

If it fails with "geometry did not intersect" or captures wrong area, **recalibrate**.

### Step 3: Recalibrate (when needed)

```bash
~/bin/screenshot-page --calibrate
```

Tell the user: "Please draw a rectangle around the browser window with your mouse."

Wait for user confirmation, then retry the screenshot.

## When to Recalibrate

- First time using the skill
- Screenshot captures wrong area
- Browser window moved to different monitor
- Error: "geometry did not intersect with any outputs"

## Config

Coordinates saved to: `~/.config/screenshot-page.conf`

## Troubleshooting

### "slurp: command not found"
Install on Fedora: `sudo dnf install slurp grim`
Install on Ubuntu: `sudo apt install slurp grim`

### "geometry did not intersect"
The saved region is no longer valid. Recalibrate.

### Black screenshot
Window may not be visible. Ensure the browser is on the correct monitor and not minimized.
