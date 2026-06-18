# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Mis-dashboard** is a static, no-build logistics monitoring platform for Loginter. It consists of two self-contained HTML files that fetch live data from Google Sheets and render interactive dashboards entirely in the browser.

- `dashboard_pedidos (4).html` — Orders/Backlog dashboard
- `dashboard_proximos_arribos (13).html` — Upcoming Arrivals dashboard

There is no package manager, build step, bundler, or test framework. To "run" the project, open an HTML file directly in a browser or serve it with any static file server (e.g. `python3 -m http.server`).

## Architecture

Each dashboard is a single monolithic HTML file (~530–720 lines) containing inline CSS, inline JavaScript, and a Chart.js CDN import. Both files follow the same architecture:

### Data Flow

```
Google Sheets (public CSV URL)
  → loadData()          fetch + parse
  → parseCSV()          character-by-character state machine (handles quoted fields)
  → buildColMap()       map logical column names to indices via keyword matching
  → populateFilters()   build multi-select dropdown options
  → renderTable()
       ├── getFiltered()    apply active filter selections + text search
       ├── applySort()      sort by column (auto-detects date/number/text type)
       ├── DOM update       bulk innerHTML replacement for table rows
       ├── updateKPIs()     recalculate aggregate metrics from filtered rows
       └── updateCharts()   destroy + recreate Chart.js instances
```

### Column Mapping

Columns are identified by keyword matching against header names (`findCol()`), not by fixed position. This makes the dashboards resilient to column reordering in the source sheet. Column names are in Spanish with potential accent characters.

### State

All state is held in module-level variables: `allRows` (raw data), `headers`, `colMap`, `sortCol`/`sortDir`, and a `charts`/`chartInstances` object. There is no framework — state changes trigger a full `renderTable()` re-render.

### Multi-Select Filters

Custom-built (no library). Key functions: `toggleDrop(msId)`, `getMS(msId)`, `updateBtn(msId)`, `buildDrop(dropId)`. Dropdowns are `<div>`-based, not `<select>` elements.

## Key Conventions

### Number & Date Formatting
- Numbers use `toLocaleString('es-AR')` — thousands separator is `.`, decimal is `,`
- Dates are stored as `DD/MM/YYYY`; `parseDateAR(s)` converts to `YYYY-MM-DD` for sort comparisons
- `num(v)` handles both `.` and `,` as decimal separator when parsing incoming data

### Status Badges
- `badgeEstado(v)` / `badgeStatus(v)` — map Spanish status strings to colored pill badges
- `badgeASN(v)` — specific badge for ASN presence/absence
- Color coding: green = Finalizado/Con ASN, yellow = Pendiente, red = Cancelado/Sin ASN

### Charts
- All charts are Chart.js 4.4.0, loaded from CDN
- Before creating a chart, always call `.destroy()` on the existing instance to avoid canvas reuse errors
- Dual Y-axis (left = units, right = lines/percentage) is a recurring pattern

### Data Validation
- Rows are validated by checking that the first 4 columns contain numbers before inclusion
- SAP order numbers must be 8+ digit numeric strings
- "Merged cell" propagation: blank cells in certain columns inherit the last non-blank value (`lastValue` pattern)

## Google Sheets Data Sources

- **Pedidos:** `https://docs.google.com/spreadsheets/d/e/2PACX-1vSLQWkhqF_hC4v6MjBKXBuWV8XaMtTL5F3Zn2l6efVjMf7EdMNJxxCsmuU9raeCug/pub?gid=941366860&single=true&output=csv`
- **Próximos Arribos:** `https://docs.google.com/spreadsheets/d/e/2PACX-1vQ-ElVsKzXGfIFq4yC4RshpJhQ7IGb7JlBpmjshI4kK4T0H6Gg1LyiOeLt6Raf_OE3HYum8LPOmOPYd/pub?gid=1842608063&single=true&output=csv`

The sheets must be published as CSV (File → Share → Publish to web). If data stops loading, the most likely cause is the sheet publication being revoked.

## UI / Styling Notes

- Fonts loaded from Google Fonts: **DM Sans** (body), **DM Mono** (numbers/codes), **Nunito** (headers)
- Color palette uses CSS variables defined at the top of each `<style>` block
- Responsive breakpoints at `1000px` and `900px` — table collapses to horizontal scroll below these widths
- KPI grid uses CSS Grid; charts use a flex-wrap row layout
