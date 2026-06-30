# DSA Prep Tracker — n8n Setup Guide

A 180-day / 300-question DSA accountability system. Three n8n workflows, one Google Sheet, one Telegram bot.

---

## Quick Start

### 1. Create the Google Sheet

Create a new Google Spreadsheet. You'll need the **Spreadsheet ID** (the long string in the URL between `/d/` and `/edit`).

Create **4 tabs** with these exact names and headers (Row 1 = headers):

**Tab: `Questions`**
```
ID | Problem | Link | Topic | Pattern | Difficulty | DateAdded | Status | Stage | NextReviewDate | LastConfidence
```

**Tab: `Log`**
```
Timestamp | DayNumber | Question | Type | TimeTakenMin | Confidence | Summary | Mistake | RawInput
```

**Tab: `Days`**
```
Date | DayNumber | Logged | NewCount | ReviewCount
```

**Tab: `Config`** (reference only — the start date is hardcoded in Code nodes)
```
Key       | Value
StartDate | 2026-06-30
```

> **Tip:** Leave all tabs empty after the header row. Data is created automatically by the workflows.

---

### 2. Create a Telegram Bot

1. Open Telegram, search for **@BotFather**
2. Send `/newbot`, follow the prompts, pick a name
3. Copy the **API token** (looks like `123456789:ABCdefGHIjklMNOpqr...`)
4. **Start a conversation** with your new bot (send it any message — this is required before the bot can message you back)
5. **Get your Chat ID:** Send any message to [@userinfobot](https://t.me/userinfobot) — it will reply with your numeric chat ID

---

### 3. Get a Gemini API Key

1. Go to [Google AI Studio](https://aistudio.google.com/apikey)
2. Create an API key (free tier is fine — a few calls/day is well within limits)
3. Copy the key

---

### 4. Import Workflows into n8n

For each of the 3 JSON files:

1. Open n8n
2. Click **+ Add Workflow** → **Import from File** (or paste the JSON via Ctrl+V on the canvas)
3. Import in this order:
   - `workflow2_telegram_logger.json` (most important — test first)
   - `workflow1_daily_reminder.json`
   - `workflow3_weekly_report.json`

---

### 5. Replace Placeholders

After importing each workflow, you need to replace placeholder values:

#### In EVERY workflow:

| Placeholder | Replace With | Where |
|---|---|---|
| `YOUR_CREDENTIAL_ID` | Your n8n credential IDs | Credential fields on every Telegram and Google Sheets node — click each node and re-select your credential from the dropdown |
| `YOUR_SPREADSHEET_ID` | Your Google Sheet ID | Every Google Sheets node's Document ID field |

#### In Workflow 2 (Telegram Logger):

| Placeholder | Replace With | Where |
|---|---|---|
| `YOUR_GEMINI_API_KEY` | Your Gemini API key | "Gemini Parse" HTTP Request node URL |

#### In Workflow 1 (Daily Reminder) & Workflow 3 (Weekly Report):

| Placeholder | Replace With | Where |
|---|---|---|
| `YOUR_CHAT_ID` | Your Telegram chat ID (numeric) | "Send Reminder" / "Send Report" Telegram node |
| `YOUR_GEMINI_API_KEY` | Your Gemini API key | "Gemini Report" HTTP Request node URL (Workflow 3 only) |

#### Start Date:

The start date `2026-06-30` is hardcoded in the following Code nodes. If you want a different start date, update the `START_DATE` constant:
- **Workflow 2:** "Spaced Rep Logic", "Prepare Skip Days"
- **Workflow 1:** "Build Reminder", "Check Missing Days"
- **Workflow 3:** "Compute Stats"

---

### 6. Set Up Credentials in n8n

If you haven't already:

1. **Telegram:** Settings → Credentials → Add → "Telegram API" → paste your bot token
2. **Google Sheets:** Settings → Credentials → Add → "Google Sheets OAuth2 API" → follow the OAuth flow

Then go back to each imported workflow and select the correct credential on every Telegram/Google Sheets node.

---

### 7. Activate

1. **Workflow 2 first** — Toggle Active. Send `/done Two Sum, new, 15 min, confidence 4, used hash map` to your bot. Check that rows appear in Questions, Log, and Days tabs.
2. **Workflow 1** — Toggle Active. Manually trigger once to verify the reminder message.
3. **Workflow 3** — Toggle Active. Manually trigger once to verify (will show empty stats until you have data).

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│ Workflow 2: Telegram Logger (webhook)                    │
│                                                          │
│ Telegram Trigger → Route Command → Is Done?              │
│   ├─ YES → Extract Message → Gemini Parse → Validate     │
│   │   ├─ Valid → Read Questions → Spaced Rep Logic       │
│   │   │   → Upsert Question → Log → Read Days            │
│   │   │   → Upsert Days → Confirmation                   │
│   │   └─ Invalid → Parse Error reply                     │
│   ├─ /skip → Upsert Days (0 counts) → Skip Confirm      │
│   └─ other → Help message                                │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ Workflow 1: Daily Reminder & Midnight Backfill           │
│                                                          │
│ Chain A (Reminder, cron 8 PM IST):                       │
│ Schedule → Read Questions → Read Days → Build Reminder   │
│   → Send Telegram (pace + due reviews)                   │
│                                                          │
│ Chain B (Backfill, cron 12:05 AM IST):                   │
│ Schedule → Read Days → Check Missing Days (Code)         │
│   → Write Missing Days (Logged=N)                        │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ Workflow 3: Weekly Report (cron Sunday 9 PM IST)         │
│                                                          │
│ Schedule → Read Log/Questions/Days → Compute Stats       │
│   → Gemini narrative → Send Telegram report              │
└──────────────────────────────────────────────────────────┘
```

---

## Spaced Repetition Logic

Stage intervals (days): `[1, 3, 7, 14, 30, 60]`

| Event | Confidence | Stage Change | Notes |
|---|---|---|---|
| New problem | — | Stage = 0, review in 1 day | |
| Review | 1–2 | Reset to Stage 0 | Full reset |
| Review | 3 | Stage + 1 | Normal progression |
| Review | 4–5 | Stage + 2 (skip ahead) | Capped at last index |
| Review at Stage 5 | ≥ 4 | Status → Mastered | No more reviews |

---

## Commands

| Command | What it does |
|---|---|
| `/done [description]` | Log a problem. Gemini parses your message — be as messy as you want. |
| `/skip` | Mark today as acknowledged with 0 problems. Due reviews carry over. |

**One `/done` per problem.** Don't combine multiple problems in one message.

---

## Troubleshooting

- **IF node conditions don't import correctly:** Open the IF node, delete the condition, and re-add it manually (check `$json._route` equals `done` or `skip`; check `$json.valid` is true).
- **Google Sheets "matching columns" not set:** Open the Upsert Question node → Column Mapping → ensure "Matching Columns" is set to `Problem`. For Upsert Days nodes, set it to `Date`.
- **Sheet Name dynamic loading issues ("Could not get parameter 'sheetName'"):** The workflows are configured to pass raw strings for sheet names (e.g., `"Questions"`, `"Days"`, `"Log"`) instead of using Resource Locator dropdowns. This prevents n8n from throwing dynamic list loading errors in the editor when credentials are not yet authorized or Google Drive API permissions are missing.
- **Gemini returns errors:** Verify your API key in the HTTP Request URL. Check that `gemini-2.0-flash` model is available in your region.
- **Bot doesn't respond:** Make sure you've started a conversation with the bot first (send it any message). Check that Workflow 2 is Active (toggle in top-right).
- **Timezone issues:** All three workflows have `timezone: Asia/Kolkata` in settings. Change in n8n workflow settings if needed.
- **Extra columns appearing in sheet:** The `autoMapInputData` mode maps by header name. Underscore-prefixed metadata fields (`_type`, `_dayNumber`, etc.) should be ignored, but if extra columns appear, delete them and switch the Sheets node to "Map Each Column Manually" mode.
