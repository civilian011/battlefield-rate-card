# Battlefield Rate Card Builder

Single-file web app for Battlefield Equipment Rentals sales reps. A rep builds a customer-specific rental rate card from the 2027 Zone A rates: include or remove categories and items, reorder categories, change prices, apply % adjustments, then publish a PDF through the browser's "Save as PDF". The PDF layout matches the official 2027 rate card PDF (letter size, same fonts, colours, Cat logos, Green Leaf icons, terms and contact page).

- Live: https://civilian011.github.io/battlefield-rate-card/ (public; `noindex` and robots.txt ask search engines not to list it)
- Repo: https://github.com/civilian011/battlefield-rate-card (contains only the built app)
- Copies are also kept in `Downloads\Battlefield Rate Card Builder.html` and on SharePoint (David uploads those himself).
- Source PDF: `OneDrive - Toromont Industries Ltd\Marketing\Marketing Stuff\Rate Card\2027\2027 Battlefield Equipment Rental Rates.pdf`

## Layout

| Path | What it is |
|---|---|
| `_tools/src.html` | **Edit this.** The app source: HTML, CSS and JS in one file, with `{"__DATA__":1}` and `/*__ASSETS__*/{}` placeholders. |
| `_tools/data.json` | Rate data: `meta` (title, footer, year, version, terms, contact text, links) and `sections[]` → `rows[]`. |
| `_tools/assets/` | Cat logo, Green Leaf icons, social strip (transparent PNGs cropped from the PDF), heater and HVAC photos. |
| `_tools/logo.svg` | Battlefield / Cat Rentals logo. |
| `_tools/build.mjs` | Inlines data and images into src.html, producing `Battlefield Rate Card Builder.html`. |
| `Battlefield Rate Card Builder.html` | Built single-file app (generated; don't hand-edit). |
| `site/` | Git clone of the GitHub Pages repo: `index.html` (copy of the built app), `README.md`, `robots.txt`, `.nojekyll`. |

## Build, publish

```bash
cd _tools && node build.mjs          # writes ../Battlefield Rate Card Builder.html
```

Publish: copy the built file to `site/index.html`, commit, `git push` in `site/`. Pages rebuilds in about 30 s. Then copy the built file to `C:\Users\david\Downloads\` as well.

- In PowerShell, refresh PATH first so `git` and `gh` are found: `$env:Path = [Environment]::GetEnvironmentVariable('Path','Machine') + ';' + [Environment]::GetEnvironmentVariable('Path','User')`.
- gh is signed in as `civilian011`. The repo has `core.longpaths true` and a local git identity using the GitHub noreply address.
- PowerShell 5.1 mangles quoted `--jq` arguments to gh; pipe through `ConvertFrom-Json` instead.

## Data model

Row types: `sub` (grey bar = "product group"), `group` (bold label), `item` (`rates: [daily, weekly, fourWeek]`, `null` = blank; all three null prints PLEASE INQUIRE; `green`, `indent`), `note` (red bold), `text` (black bold), `image` (`image: heaters|hvac`). `logo: 'cat'` on a sub or group places one Cat logo beside two short adjacent lines in that group. Row IDs are stable and saved rate cards reference them, so never renumber them.

David updates rates without code through the in-app ••• → Update rates (admin): export CSV, edit in Excel, re-import, then "Download updated app". That rebuilds the file from the page's own source captured at DOMContentLoaded. If he sends a new CSV or app file, the data can be pulled back into `data.json`.

## Rep state (saved rate card)

Saved in localStorage (`ber-ratecard:v1:*`) and as `.json` files: `customer`, `rep`, dates, `notes`, `hiddenSections`, `hidden`, `overrides` (typed prices per column), `custom` items (anchored after a row id), `cols`, `order` (category order), `adjust` rules (`{scope: '*'|'sec:<id>'|'sub:<id>', pct: [d,w,m]}`; `null` = falls through to the broader rule, `0` = list price). Price precedence: typed > product group rule > category rule > all equipment rule > list. `normalise()` must keep old saved cards loading. Add defaults there for any new field.

## Testing notes

- Pagination is done in JS: blocks are measured in `#measure` and packed into 8.5×11in pages. Preview and print share the same DOM, so check the print path with headless Edge: `msedge --headless=new --user-data-dir=<short path> --no-pdf-header-footer --print-to-pdf=out.pdf <url>`. Use a short output path, because long paths fail silently. Never run `msedge --version`; it opens a browser window.
- To preview in the Claude browser pane, serve the folder over a small Node static server via `.claude/launch.json`; the pane can't open `file://`. Delete the config afterwards.
- The Claude browser pane can be hidden, which stalls requestAnimationFrame. That's why rendering is scheduled with `setTimeout`.
- Fieldsets need `min-width: 0` and grid tracks `minmax(0, 1fr)`, or the left panel overflows on smaller screens.

## Changes made to the source PDF's content (deliberate)

Footer "2026 Ontario Rental Rates" → "2027 Zone A Rental Rates"; PNUEMATIC → PNEUMATIC, CONROL → CONTROL, SCRIFIERS → SCARIFIERS, TELEHANDER → TELEHANDLER, truncated "SHARPENING EXTR" → "EXTRA"; DAILY/WEEKLY/4 WEEKS headers on every category.

## Customer number, approval and ERP CSV (added 2026-10-30)

- `customer.number` (Customer tab) is required to submit; it is never printed on the PDF.
- Items carry `klass: [codes]` in data.json (from column A of David's "2027 Battlefield Equipment Rental Rates Claude.xlsx", sheet "2027 Rental Rates - Zone A"). The admin rates CSV has a `Klass` column; an older CSV without it keeps the existing codes. Custom items get a typed `klass` string, not shown on the PDF.
- `DATA.meta.managers = [{name, email, pinHash}]`, pinHash = sha256('ber-approve:' + PIN). Testing: only Paul Tillmanns. Managers and PINs are edited in ••• → Update rates (admin), then "Download updated app" (or edit data.json and rebuild).
- No server: approval travels in the saved .json file. Rep: Submit for approval → downloads "<company> - for approval.json" + opens a mailto to the manager. Manager: Open from a file → Review & approve → PIN → "Email approved file to rep" (downloads "- approved.json" + mailto). "Ask for changes" sends a note back instead.
- `state.approval = {status, fp, to, toName, submittedAt, submittedBy, by, byEmail, at, code, note}`. `fp` = sha256 of the canonical state (minus id/savedAt/approval/v) + list version, so any edit cancels approval. `code` = sha256('ber-approval|fp|email|at|pinHash'). Tamper resistance is light (hash is in the public page); fine for testing, not for production.
- ERP CSV (enabled only when approved): items on the card where any shown column differs from list (plus custom items); all shown, non-blank columns of such an item; one row per Klass code × column. Columns: customer_identifier RP-0001…, klass_identifier, location_identifier blank, customer_identifier_fk = customer number, frequency daily|weekly|twenty_eight_days, amount, quantity 1, payer_type customer, type fixed. CRLF, no BOM.
- The spreadsheet lists five items not on the rate card (R0071–R0075, rotary hammer drills / demolition hammers / jackhammer) – not added yet.

## MOAB export without approval (2026-11-12)

- MOAB is the ERP. Top-bar button "Export to CSV for MOAB" sits next to Publish PDF and is always active.
- Approval is switched off with `const APPROVAL_ON = false` in src.html (flowbar and ••• menu items hidden, code kept). Set it to true to bring approval back.
- Export blocks until the Customer number is filled in and every custom item with a price has a Klass code.
- SharePoint's HTML viewer (SitePages/*.html) runs the page in a blob: iframe with `sandbox="allow-scripts"` only, so print, downloads, localStorage and mailto are all silently blocked. Nothing in the app can lift that. Reps use the GitHub Pages link instead (David chose this 2026-11-12; the intranet gets a button that opens it in a new tab). The app detects the sandbox (`SANDBOXED`: framed and origin "null"), shows a warning bar, and Publish PDF / MOAB export explain instead of doing nothing.
- The repo must stay public for Pages to work on GitHub's free plan.

## Ideas parked for later

- Industry presets (e.g. "Landscaping" removes Aerial Work Platforms and Welders in one click).
- French version: print labels are centralised in `LABEL_DEFAULTS` / `DATA.meta.labels`; item descriptions would need a `text_fr`.
