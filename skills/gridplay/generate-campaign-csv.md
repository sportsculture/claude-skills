---
description: Generate a Campaign Monitor-ready CSV and email template from Google Sheets
---

# Generate Campaign Monitor CSV & Email Template

Generate a Campaign Monitor-ready CSV and email template from Google Sheets containing team manager contact data for any tournament.

## Usage

```
/generate-campaign-csv
```

## What This Skill Does

This skill guides you through creating campaign materials for team manager invitations:

1. **Tournament Selection** - Lists available tournaments, you pick one
2. **Division Verification** - Confirms divisions exist with active join links
3. **Google Sheets Input** - Reads manager data (Team, Manager Name, Email)
4. **Division Matching** - Auto-matches sheet titles to tournament divisions
5. **Output Generation** - Creates both CSV and customized email template

## Interactive Workflow

### Step 1: Gather Information
Ask you for:
- Google Sheet URL(s) containing team manager data
- Logo/hero image URL for the email template

### Step 2: Select Tournament
Run the script to display available tournaments:
```bash
npx tsx scripts/generate-campaign-csv.ts
```

### Step 3: Process & Generate
The script will:
- Read each Google Sheet
- Match sheet titles to divisions (e.g., "Timeless 70s" -> 70s division)
- Extract: Email, First Name, Team Name
- Generate CSV + HTML template

### Step 4: Review Output
Confirm the generated files:
- CSV: `output/<tournament-slug>-invitations.csv`
- Template: `templates/<tournament-slug>-invitation.html`

## Google Sheet Format

The script expects sheets with columns like:
- `Team` or `Teams` or `Team Name` - Team name
- `Manager` or `Contact` - Manager's full name (first name extracted)
- `Email` or `eMail` - Email address

### Headerless Sheets

Some sheets have NO header row - just data starting in row 1 with format:
```
[Team Name] | [Manager Name] | [Email]
```

**If the script extracts only emails with missing names/teams**, the sheet likely has no headers. Query the sheet directly to verify.

## Division Matching

### 1. Age Patterns (Checked First)
If the title contains an explicit age like "50s", "45s", etc., that takes priority:
- "SC 2026 Legends 50s" → **50s** (not 53s!)
- "Masters 45s March 12" → **45s**

### 2. Word Patterns (Fallback)
Only used if no age pattern is found:

| Sheet Title Pattern | Matched Division |
|---------------------|------------------|
| "Family Ties" | Family Ties |
| "Forever Young" | 75s |
| "Timeless" | 70s |
| "Vintage" | 65s |
| "Classics" | 60s |
| "Legends" (without age) | 53s |
| "Masters" (without age) | 45s |
| "Veterans" (without age) | 35s |

## Output Files

### CSV Format
```csv
EmailAddress,firstname,TeamName,Division,JoinLink
john@example.com,John,Atlanta A's,70s,https://gridplay.sportsculture.io/join/E4E5CED8
```

### Email Template
HTML template with Campaign Monitor merge fields:
- `[firstname,fallback=Manager]`
- `[TeamName,fallback=your team]`
- `[Division,fallback=your division]`
- `[JoinLink,fallback=#]`

## Custom Join Links

The script uses join links from the database, but **the user often has fresh/custom links** for specific campaigns.

**Best practice:**
1. Ask the user for custom join links per division before generating
2. After generating the CSV, verify the links match what the user expects
3. Use find/replace to update links if needed

## Troubleshooting

### "No divisions found"
The tournament may not have divisions set up yet. Create divisions first.

### "No join link"
Create a join link for the division via the UI or API.

### "Access denied (403)"
Share the Google Sheet with the service account.

### "Division not detected"
Rename the sheet to include a recognizable pattern or manually edit the CSV.
