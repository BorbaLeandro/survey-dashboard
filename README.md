# survey-dashboard
Manual entry dashboard for course survey results with weighted averages.
[README.md](https://github.com/user-attachments/files/21712771/README.md)
# Survey Dashboard — Google Sheets Edition

This is a React + Vite app that **reads survey entries from a public Google Sheet (CSV)** and renders a dashboard with **weighted monthly trends** and **per-question charts**.

## 1) Prepare your Google Sheet

Create a sheet with the following headers (order can vary, but these keys must exist):

```
course, year, month, participants, Q1, Q2, Q3, Q4, Q5
```

- `course`: string (e.g., "Course X")
- `year`: number (e.g., 2025)
- `month`: number 1–12
- `participants`: number (e.g., 16, 20, ...)
- `Q*`: question scores (1–10 by default). You can use your own labels instead of Q1..Q5 — the app will read labels from the CSV header.

Example rows:

```
Course X, 2025, 7, 16, 9.2, 9.0, 8.8, 9.6, 9.4
Course X, 2025, 8, 20, 9.1, 9.3, 9.0, 9.5, 9.2
Course Y, 2025, 8, 18, 8.7, 8.9, 8.5, 9.1, 8.8
```

### Publish the sheet as CSV
- In Google Sheets: **File → Share → Publish to web → Entire document → CSV**.
- Copy the CSV link. It will look like:
  `https://docs.google.com/spreadsheets/d/{SHEET_ID}/pub?output=csv`

Alternatively you can use the direct export format:
`https://docs.google.com/spreadsheets/d/{SHEET_ID}/export?format=csv&id={SHEET_ID}&gid={GID}`

## 2) Configure the app

Edit `.env` or set environment variables at deploy time:

```
VITE_SHEET_CSV_URL=<your_public_csv_url>
VITE_SCALE_MIN=1
VITE_SCALE_MAX=10
```

For local dev, create a file named `.env` in the project root with those values.

## 3) Run locally

```
npm install
npm run dev
```

## 4) Deploy

- **Vercel**: Import this repo and set `VITE_SHEET_CSV_URL` in Project → Settings → Environment Variables.
- **CodeSandbox/StackBlitz**: Import the repo/ZIP, set the env var in their UI, or hardcode a fallback in `src/config.js`.

---

If you don't have a sheet ready yet, the app lets you paste a CSV link on the page for quick testing.

