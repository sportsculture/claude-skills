---
description: Send tournament invitation emails via SendGrid with click tracking
---

# Send Tournament Invitations via SendGrid

Send tournament invitation emails to team managers using SendGrid with click tracking.

## Usage

```
/send-invitations
```

## What This Skill Does

This skill guides you through sending tournament invitation emails:

1. **Generate CSV** - Creates recipient list from Google Sheets using `/generate-campaign-csv`
2. **Preview** - Shows recipient list and multi-team managers (dry run)
3. **Send** - Sends individual emails via SendGrid with click tracking
4. **Report** - Shows who clicked their join links

## Interactive Workflow

### Step 1: Generate or Select CSV
Either:
- Run `/generate-campaign-csv` to create a new CSV from Google Sheets
- Or use an existing CSV file from `output/` directory

### Step 2: Preview Recipients (Dry Run)
```bash
npx tsx scripts/send-tournament-invitations.ts \
  --csv output/<tournament>-invitations.csv \
  --dry-run
```

This shows:
- Total recipient count
- Sample recipients (first 5)
- Multi-team managers who will receive multiple emails

### Step 3: Send Emails
```bash
npx tsx scripts/send-tournament-invitations.ts \
  --csv output/<tournament>-invitations.csv
```

Sends individual emails with:
- Subject: `Register [TeamName] for the [Tournament] – [Division]`
- Preview text: `Click to register, then share the player join link with your team.`
- Tracked join links for click reporting
- Rate limited at 100ms between sends

### Step 4: View Tracking Report
```bash
npx tsx scripts/send-tournament-invitations.ts \
  --report output/send-report-<timestamp>.json
```

Shows:
- Total sent/failed counts
- Click rate percentage
- Per-recipient click status

## Email Content

**Subject:** `Register [TeamName] for the [Tournament] – [Division]`

**Preview:** `Click to register, then share the player join link with your team.`

**Body includes:**
- Tournament hero image
- Personalized greeting with manager's first name
- Team name and division
- CTA button with tracked join link
- Instructions for sharing player invite links

## Click Tracking

Each join link is wrapped with a tracking token:
```
/api/tracking/click/{token}?redirect={originalJoinLink}
```

When a manager clicks:
1. Click is recorded in the send report JSON
2. Manager is redirected to the actual join link
3. Report shows who clicked and when

## CLI Options

```bash
npx tsx scripts/send-tournament-invitations.ts [options]

Options:
  --csv <file>           CSV file with recipients
  --dry-run              Preview without sending
  --tournament-name <n>  Override tournament name in emails
  --hero-image <url>     Override hero image URL
  --report <file>        View tracking report
```

## CSV Format

```csv
EmailAddress,firstname,TeamName,Division,JoinLink
john@example.com,John,Warriors,70s,https://gridplay.sportsculture.io/join/ABC123
```

## Multi-Team Managers

Managers with multiple teams receive separate emails for each team. The script identifies and reports these duplicates in the dry run output.

## Pre-Send Checklist

Before sending, always verify:

1. **Join links are correct and fresh** - Links may expire
2. **Query recent registrations first** - Don't send duplicate invites
3. **Dry run to preview** - Review recipient count and divisions

## Requirements

- `SENDGRID_API_KEY` configured in `.env`
- CSV file with recipients (generate with `/generate-campaign-csv`)
