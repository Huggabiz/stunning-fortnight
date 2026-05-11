# CLAUDE.md

Operational guidance for Claude when working on this repo.

## Project

**Retailer Data Importer** — a single-file, browser-based tool that ingests sales/stock files from wholesale retailers (Argos, John Lewis, Dunelm, Sainsbury's, Selfridges, Fenwick, …), parses their incompatible formats, and outputs a harmonised CSV. Built for ~10 non-technical sales team members at Joseph Joseph.

**Pipeline:** HTML importer → harmonised CSV → SharePoint "Parsed Files" folder → Excel + Power Query → (future) Power BI.

## Architecture

- **Single self-contained `index.html`** at repo root. No build step, no server. SheetJS loaded from CDN.
- **All processing is client-side.** Must work in modern browsers including Safari/iPad.
- **Hosted on GitHub Pages** via the workflow in `.github/workflows/deploy.yml`. Live URL: `https://huggabiz.github.io/stunning-fortnight/`.
- **No server-side code, no backend, no dependencies to install.** If a change would require either, stop and check first.

## Files

| File | Purpose |
|---|---|
| `index.html` | The tool. Single file, hand-edited. |
| `.github/workflows/deploy.yml` | Pages deploy on push to default branch |
| `CLAUDE.md` | This file |

There is no `package.json`, no build system, no test runner. Don't add one unless explicitly asked.

## Canonical output schema — DO NOT change without confirmation

Every CSV produced has exactly these columns, in this order:

```
import_id, import_timestamp, source_filename, retailer,
period_start, period_end, period_type,
sku_code, sku_source, sku_retailer_code, product_description, ean,
location_type, location_name,
sales_units, returns_units, demand_units,
stock_units, stock_available, stock_in_transit, stock_on_order,
weeks_cover, availability_pct, serviceability_pct,
data_types
```

- `period_type`: `DAILY` | `WEEKLY` | `4_WEEK` | `SNAPSHOT`
- `sku_source`: `JJ_SKU` | `RETAILER_CODE` | `EAN` | `BLANKED`
- `location_type`: `TOTAL` | `STORE` | `WAREHOUSE` | `DEPOT` | `ONLINE`
- `data_types`: pipe-delimited subset of `SALES|STOCK|RETURNS|AVAILABILITY`

Downstream Power Query depends on this exactly. Adding columns is acceptable (appended); renaming or removing breaks the pipeline.

## Supported retailers

Each parser is hard-coded and lives inline in `index.html`. Detection uses filename patterns + header signatures.

| ID | Format | Grain | Data types |
|---|---|---|---|
| `argos_sales` | CSV, pivoted daily triplets | DAILY / TOTAL | SALES |
| `argos_stock` | CSV, per-warehouse | SNAPSHOT / WAREHOUSE | STOCK |
| `john_lewis` | CSV, SKU × branch | WEEKLY / STORE | SALES, STOCK, RETURNS |
| `dunelm` | XLSX, one sheet per week | WEEKLY / TOTAL | SALES, STOCK, AVAILABILITY |
| `sainsburys` | XLSX, multi-level header | WEEKLY / TOTAL | SALES |
| `selfridges` | XLSX, week-number filename | WEEKLY / TOTAL | SALES, STOCK |
| `fenwick` | XLSX, ~140 cols, store blocks | SNAPSHOT / STORE+WAREHOUSE+ONLINE | SALES, STOCK |

Adding a new retailer = a new entry in the `RETAILERS` detection array + a new parser function. Follow the existing pattern; return `{rows: [...]}` or `{error: "..."}`.

## Design decisions — already settled, do not relitigate without asking

1. **Hard-coded parsers, no generic column-mapper.** Formats are structurally too varied (Argos pivots, Fenwick 140-col store blocks, Dunelm multi-sheet). A generic mapper was considered and rejected.
2. **No "custom format" mode.** It was in v0.2.0 and removed in v0.3.0 — it couldn't handle non-tabular files and created false confidence. New retailers get a proper parser, not a self-service mapping UI.
3. **Per-file dates, not global dates.** Each uploaded file has its own From/To inputs, pre-populated where the parser can infer them, user-editable. Don't reintroduce a single global date pair.
4. **SKU blanking toggle, not forced master lookup.** When a file only contains retailer codes, the user toggles "Blank out SKU code"; the retailer code is preserved in `sku_retailer_code` and `sku_source` is set to `BLANKED`. SKU master lookup is deferred future work, not a blocker.
5. **Deduplication happens in Power Query, not the importer.** Every row gets `import_id` + `import_timestamp`; the natural key is `retailer + sku_code + period_start + period_end + location_type + location_name`. The importer does not try to detect or merge overlaps.
6. **No import log CSV.** Was in v0.2.0, removed in v0.3.1. Every harmonised row already carries `import_id` and `import_timestamp`; the separate log added no value.
7. **One harmonised CSV per source file.** Multi-file upload produces one CSV for each input file, named with a sequence number (`_001`, `_002`, …) to disambiguate when retailer + dates collide. Format consistency is still enforced — adding a different format to an existing batch is blocked. (Earlier versions merged into a single combined CSV; changed in v0.3.3.)
8. **Save dialog uses File System Access API where available, falls back to auto-download.** Chrome/Edge desktop get a folder picker (`showDirectoryPicker`) so all CSVs land in one chosen location. Safari/iPad/Firefox fall back to sequential browser downloads. Don't drop the fallback — iPad support is a hard requirement.

## Working conventions

- **Direct, practical outputs.** No scaffolding the user didn't ask for. No "future-proofing" abstractions. Three similar lines beats a premature helper.
- **Iterative, build-and-test.** Hugo prefers seeing a working change quickly over a long plan.
- **Version visible in the page.** The header in `index.html` shows the current version (`v0.X.Y`). Bump it whenever functional behaviour changes; keep `APP_VERSION` in the script in sync.
- **CSV export filenames carry context, not a tool version.** Format: `harmonised_{retailer}_{YYYYMMDD}_{YYYYMMDD}_{NNN}.csv` where `NNN` increments per output file in a batch (`001`, `002`, …). Built in `buildExportFiles()`.
- **Commit messages: imperative, brief, explain the why if non-obvious.** One-liner is fine for small changes.
- **Raw M code for Power Query.** No `.xlsx` generation; queries are pasted into Excel's Advanced Editor.

## Editing `index.html`

- Keep it a **single self-contained file**. No splitting into separate `.js` / `.css` unless explicitly asked.
- Inline styles in the `<style>` block use the CSS variables at the top — reuse them rather than introducing new colours.
- State lives in the `S` object. The four steps are `upload | detect | preview | export`; the `render()` function dispatches.
- Parsers all take `(data, meta)` (or `(wb, meta, sheets)` for Dunelm) and return `{rows: [...]}` or `{error: "..."}`. Each row must contain every column in the canonical schema (use `null` for missing numeric fields).

## Deploy / release flow

1. Commit and push to the current default branch.
2. The Pages workflow (`.github/workflows/deploy.yml`) deploys automatically.
3. Live URL: `https://huggabiz.github.io/stunning-fortnight/`.
4. For meaningful releases, tag the commit: `git tag v0.3.2 && git push --tags`.

When asked to push or release, do not push to `main` (or any non-feature branch) without explicit permission.

## Things to avoid

- Adding a build step, bundler, framework, or npm dependency.
- Server-side code or any backend calls.
- Fuzzy product matching, ML, or any heuristic SKU resolution. SKUs are exact-match only.
- Reintroducing the custom-format mapper, the import log CSV, or global date inputs.
- Renaming or reordering output schema columns.
- Silent error swallowing in parsers — surface a clear `{error: "..."}` instead.

## Deferred work (don't start without asking)

- SKU master file / lookup with fuzzy name matching fallback
- Direct SharePoint upload from the browser
- New retailer parsers (these are added on demand, one at a time)
- Power BI dashboard layer
- Sainsbury's channel split (M2M Store / M2M Web / Store Sales as separate location types)
