# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

A static, client-side-only HTML/JS dashboard for PIH Haiti's nutrition program. There is no build step, no server, no package manager, and no tests. Open any `.html` file directly in a browser to run it.

The UI is bilingual (French primary, English secondary). French strings appear first throughout the codebase.

## How to run

Open `index.html` in a browser. No server required. For drag-and-drop file upload to work reliably, serve via a local HTTP server:

```
python -m http.server 8080
# then open http://localhost:8080
```

## Architecture

### Data flow

1. **`index.html`** — Upload page. User drops one or more CommCare `.xlsx` export files. `parser.js` parses them client-side using SheetJS (`xlsx@0.18.5` from CDN). The parsed data object is stored in `sessionStorage` as `pih_nutrition_data`.
2. **`analytics.html`** — Reads `pih_nutrition_data` from `sessionStorage` and renders all analytics charts and tables using `analytics.js`.
3. Other pages (`chw.html`, `clinical.html`, `supervisor.html`, `nextsteps.html`) also read from `sessionStorage` for their specific views.

If `sessionStorage` is full (large files), the upload page truncates to 5,000 cases as a fallback.

### JS files

- **`js/parser.js`** — Parses raw CommCare `.xlsx` workbooks into the dashboard data model. Entry point: `parseFiles(fileList)` → returns `{ meta, cases, suppMonthly, dqReport }`. Handles multi-file merging (deduplicates by `formid`). Each case may have up to three program episodes: `pns`, `pta`, `usn`.
- **`js/data.js`** — Synthetic demo/fallback data. Not used when real CommCare data is loaded. Contains `PATIENTS_RAW`, `DISCHARGED_PATIENTS`, `COHORT_DATA`, `ALERT_LOG`, risk-scoring logic (`computeRisk`), and helper functions (`getPatients`, `getSiteStats`, `getCHWPatients`). Also has a simulation slider (`SIM_DAYS`) for demo purposes.
- **`js/analytics.js`** — All rendering logic for `analytics.html`. Reads from `sessionStorage`, filters by date range, and renders charts/tables as injected HTML.
- **`js/shared.js`** — Small shared utilities: `buildNav(activeId)` (generates the nav bar HTML), `programTag`, `progressBar`, `renderBarChart`.

### CommCare data model (what `parser.js` expects)

Expected sheets in the `.xlsx` export: `Enregistrement`, `Visite PNS`, `Visite PTA`, `Visite USN`, `Exeat PNS`, `Exeat PTA`, `Exeat USN`, `Dictionnaire`.

Cases are keyed by `form.case.@case_id`. The `Dictionnaire` sheet provides code→label lookups for fields like `Oedème` and `P/T`.

### Programs

- **USN** — Inpatient SAM (Severe Acute Malnutrition). Overdue threshold: 7 days.
- **PTA** — Outpatient SAM. Overdue threshold: 14 days.
- **PNS** — Preventive/MAM (Moderate Acute Malnutrition). Overdue threshold: 14 days.

### Risk scoring (`computeRisk` in `data.js`)

Scored 0–N on: days overdue vs. program threshold, oedema, MUAC decline, low absolute MUAC, and weight gain rate (g/kg/day). Maps to `critical / high / moderate / low`.

## Key conventions

- No framework — plain DOM manipulation, `innerHTML` injection.
- CSS variables defined in `css/style.css` (e.g. `--blue`, `--gray-500`, `--radius-lg`).
- Nav is injected via `document.getElementById('nav-container').innerHTML = buildNav('<pageId>')` at the top of each page's inline `<script>`.
- Each page's inline `<script>` tag handles all page-specific logic; `analytics.js` is the exception (too large to inline).
- `exampleExcelData/` contains real-format sample `.xlsx` files for testing the parser.
