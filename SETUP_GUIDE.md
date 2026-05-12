# MMR Dashboard — Google Sheets Setup Guide

## Overview

The MMR Scrap Price Tracker dashboard now fetches live data from a **Google Sheet** instead of using hardcoded data. This means you can update prices in Google Sheets and the dashboard automatically reflects the changes.

---

## Step 1: Upload Your Data to Google Sheets

1. Go to [Google Sheets](https://sheets.google.com) and create a **New Spreadsheet**
2. Name it something like `MMR Weekly Data`
3. **Import the Excel file**:
   - Go to **File → Import → Upload**
   - Select your `Weekly MMR Sheets (1).xlsx` file
   - Choose **Replace spreadsheet** and click **Import data**
4. Make sure the first row has these column headers (exactly):

| Column | Header Name |
|--------|-------------|
| A | Week |
| B | Month |
| C | Issue Date From |
| D | Issue Date To |
| E | Type |
| F | Description |
| G | State |
| H | Weekly Avg Date |
| I | Spot Rate |
| J | Year |

> **Important**: The dashboard reads columns **E (Type)**, **G (State)**, **H (Weekly Avg Date)**, and **I (Spot Rate)**. These must be present.

---

## Step 2: Publish the Google Sheet to the Web

This is the critical step that allows the dashboard to read the data.

1. Open your Google Sheet
2. Go to **File → Share → Publish to web**
3. In the dialog:
   - **Link**: Select `Sheet1` (or whichever tab has your data)
   - **Format**: Select **Comma-separated values (.csv)**
4. Click **Publish**
5. Click **OK** on the confirmation dialog

> **Note**: The sheet does NOT need to be shared publicly. "Publish to web" is different from sharing — it creates a read-only CSV feed.

---

## Step 3: Get Your Google Sheet ID

Your Google Sheet URL looks like this:

```
https://docs.google.com/spreadsheets/d/1aBcDeFgHiJkLmNoPqRsTuVwXyZ/edit
```

The Sheet ID is the long string between `/d/` and `/edit`:

```
1aBcDeFgHiJkLmNoPqRsTuVwXyZ
```

---

## Step 4: Configure the Dashboard

### Option A: Edit `config.js` directly (Local / Simple)

Open `config.js` and replace the placeholder:

```javascript
const MMR_CONFIG = {
  GOOGLE_SHEET_ID: '1aBcDeFgHiJkLmNoPqRsTuVwXyZ',  // ← Your Sheet ID
  SHEET_NAME: 'Sheet1',
  CACHE_MINUTES: 30,
  USE_FALLBACK: true
};
```

### Option B: Use GitHub Environment Variables (GitHub Pages)

If you're deploying via GitHub Pages with GitHub Actions:

1. Go to your GitHub repo → **Settings → Secrets and variables → Actions**
2. Click **New repository variable** (under the **Variables** tab)
3. Add:
   - **Name**: `GOOGLE_SHEET_ID`
   - **Value**: Your Google Sheet ID (e.g., `1aBcDeFgHiJkLmNoPqRsTuVwXyZ`)

4. Create/update your GitHub Actions workflow (`.github/workflows/deploy.yml`):

```yaml
name: Deploy MMR Dashboard

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 6 * * 5'  # Every Friday 6 AM UTC (refresh weekly)
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Inject Google Sheet ID into config.js
        run: |
          sed -i "s|YOUR_GOOGLE_SHEET_ID_HERE|${{ vars.GOOGLE_SHEET_ID }}|g" config.js

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./
```

> This workflow replaces the placeholder in `config.js` with the actual Sheet ID from your GitHub variable before deploying.

---

## Step 5: Test the Dashboard

1. Open `MMR.html` in your browser
2. You should see the loading spinner followed by your data
3. If you see an error, check:
   - ✅ The Sheet ID in `config.js` is correct
   - ✅ The Google Sheet is **published to web** (Step 2)
   - ✅ The sheet tab name matches `SHEET_NAME` in config
   - ✅ Column headers match exactly (Type, State, Weekly Avg Date, Spot Rate)

---

## How Data Flows

```
Google Sheet (your data)
    ↓ Published as CSV
Dashboard (MMR.html)
    ↓ Fetches CSV via config.js Sheet ID
    ↓ Parses CSV rows
    ↓ Builds DB object (dates, series, etc.)
    ↓ Renders all tabs (Price Table, Monthly, Trend, etc.)
```

---

## Adding New Weekly Data

1. Open your Google Sheet
2. Add new rows at the bottom with the latest week's data
3. The dashboard will pick up the new data on next load (after cache expires, default 30 min)
4. To force refresh: clear browser cache or add `?nocache=1` to the URL

---

## File Structure

```
Dashboard/
├── MMR.html          ← Main dashboard (loads from Google Sheets)
├── config.js         ← Google Sheet ID configuration
├── SETUP_GUIDE.md    ← This guide
└── .github/
    └── workflows/
        └── deploy.yml  ← (Optional) GitHub Actions for auto-deploy
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Google Sheet Not Configured" | Set `GOOGLE_SHEET_ID` in `config.js` |
| "Failed to Load Data" | Make sure Sheet is published to web as CSV |
| CORS error in console | Use the published CSV URL (not the edit URL) |
| Data looks wrong | Check column headers match exactly |
| Stale data showing | Clear localStorage or wait for cache to expire |
| Blank dashboard | Open browser console (F12) for error details |

---

## Security Notes

- **Published to web** only creates a read-only CSV feed — nobody can edit your sheet
- The Sheet ID is not sensitive — it only gives read access to published data
- For extra security, you can restrict the published range to only the data columns
