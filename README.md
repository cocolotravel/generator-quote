# Cocolo Travel — Quote Generator

A browser-based tool for generating travel quotes. Built as a single HTML file with no build step required.

## Files

| File | Description |
| --- | --- |
| `index.html` | Main application (UI + logic) |
| `cities_data.csv` | List of available destination cities |
| `services_data.csv` | Available services and pricing |
| `transports_data.csv` | Available transport options and pricing |

## Usage

Open `index.html` directly in a browser. No server or installation needed.

## Data

The CSV files are loaded at runtime to populate the quote form. To update destinations, services, or transport options, edit the corresponding CSV file.

---

<!-- markdownlint-disable MD024 -->

## Internal Tools Technical Reference

**Company:** Cocolo Travel (ここロトラベル合同会社)
**Location:** Ebisu, Tokyo
**Last updated:** May 2026

---

## Overview

Three internal tools have been built to replace manual workflows. All tools follow the same architecture pattern: standalone HTML files with no backend dependency, deployed as static sites on Linode Object Storage. The exception is the Drafts API, which is a lightweight Express server running in Docker.

---

## Tool 1 — Fax Generator

**Live URL:** <https://fax.cocolotravel.com>
**File:** `fax-generator-v2.html` + `hotels_data.csv`

### Purpose

Replaces the manual workflow of composing Japanese-language faxes to hotels. Staff send 2–3 faxes daily to a database of 450+ Japanese hotels.

### Architecture

- Single HTML file + CSV (hotel database)
- CSV loaded via `fetch()` on page load
- PDF generated via `html2pdf.js` with `html2canvas` for rendering
- Deployed to Linode Object Storage with CNAME `fax.cocolotravel.com`

### Key Features

- Hotel search dropdown (453 hotels from CSV)
- Sender selection with kanji mapping (English UI → Japanese output)
- Multiple reservation blocks per fax (`予約内容1`, `予約内容2`, etc.)
- Accurate per-page PDF rendering (X枚目/Y枚) via two-pass html2canvas
- `break-inside: avoid` prevents reservation blocks from splitting across pages

### Known Issues & Fixes

- **Blank pages**: caused by `visibility: hidden` on clones — fixed via `html2canvas` `onclone` callback to remove overflow constraints before rendering
- **Bucket naming**: Linode Object Storage requires the bucket name to exactly match the custom domain for TLS certificate validation (bucket must be named `fax.cocolotravel.com`)

### Deployment

Upload `fax-generator-v2.html` and `hotels_data.csv` to the bucket via Cyberduck. Both files must be in the same directory.

---

## Tool 2 — Hotel Name List Generator

**Live URL:** Not deployed (used locally)
**File:** `Cocolo_NameList_Generator.html`
**Wiki:** wiki.cocolotravel.com (Operations collection)

### Purpose

Replaces a manual Excel workflow where staff assign tour guests to hotel rooms (Twin, Single, Double, Triple) for group tours.

### Architecture

- Single HTML file, no backend
- Draft persistence via `localStorage`
- Export to Excel (xlsx.js) and PDF (jsPDF + jsPDF-autotable)

### Key Features

- Auto-create rooms based on guest count
- Drag-and-drop guest assignment between rooms
- Drag-and-drop room reordering
- Multiple hotels per tour
- Keyboard shortcuts
- Validation warnings
- Undo/redo
- Draft save/load via localStorage
- Room numbers always sequential (calculated dynamically at render time, not stored statically)

### Export

- **Excel**: one sheet per hotel
- **PDF**: one file per hotel, filename includes hotel name and tour reference

---

## Tool 3 — JR Seat Reservation Order Form

**Live URL:** Not deployed (used locally)
**File:** `JR_Reservation_Tool.html`

### Purpose

Staff must physically visit JR station counters to purchase train tickets. The booking system exports data in French format; the JR counter requires a specific printed form. This tool auto-converts French-format train data into a formatted JR order form.

### Architecture

- Single HTML file, no backend
- localStorage draft persistence
- PDF export via jsPDF + html2canvas

### Key Features

- Paste French-format booking data → auto-populates JR form
- 80+ Japanese station name dictionary (organized by region, handles variants and typos)
- Unrecognized station warnings (⚠ badges)
- Multi-customer tab navigation
- PDF filename = customer name + date range
- One PDF per customer

---

## Tool 4 — Quote Generator

**Live URL:** <https://quote.cocolotravel.com>
**Files:** `Cocolo_Quote_Generator_v3.html` + `services_data.csv` + `transports_data.csv` + `cities_data.csv`

### Purpose

Web-based tour pricing calculator replacing a manual Excel template. Staff build quotes for group tours and generate pricing for B2B partners.

### Architecture

- Single HTML file + 3 CSV catalog files
- CSVs loaded via `fetch()` on page load — **all 4 files must be in the same directory**
- Draft persistence via Drafts API (see below)
- Excel export via xlsx.js
- Deployed to Linode Object Storage with CNAME `quote.cocolotravel.com`

### Tabs

#### 1. Builder

Staff input for the tour:

##### Tour Setup (global fields)

| Field | Notes |
| --- | --- |
| Tour Name, Ref, Season | Free text |
| Guides | Number of guides |
| Nights | Auto-calculated from hotel row count (read-only) |
| Tax Rate % | Default 10% — used in margin calculations only, not applied to input prices |
| 1 EUR = ¥ | Exchange rate |
| Guide Daily Rate / Allowance | Pre-filled defaults for Guide Days section |
| Partner Markup % | Used to calculate partner commission |
| Partner Subject to Tax | Checkbox — when checked, effective commission cost = commission × 0.9 (input tax recovery) |
| Target Markup % | Default 30% — used for Suggested Price calculation |

##### Hotels section

- Night number: auto-numbered, read-only, renumbers on drag-reorder
- City: dropdown from `cities_data.csv` (93 cities, ranked by foreign tourist popularity)
- Hotel Name: free text
- Meals, Room Type: dropdowns
- Twin ¥ / Single ¥ / Guide ¥: per-night prices (tax-inclusive)
- Single Supplement: auto-calculated = `(Single × 2) − Twin`
- Drag-to-reorder rows

##### Services section

- Day number → City auto-filled from matching hotel night (read-only text, not a dropdown)
- Service name: searchable autocomplete from 329 catalog items (`services_data.csv`), with custom entry fallback
- Per Person ¥, Per Group ¥, Guide ¥, Qty
- Drag-to-reorder rows

##### Transports section

- Same pattern as Services, 501 catalog routes (`transports_data.csv`)
- Drag-to-reorder rows

##### Guide Days section

- Auto-synced to hotel nights + 1 (always one more day than hotel nights)
- Day number: auto-numbered, read-only
- City: auto-filled from matching hotel night; last day always shows "Return"
- Rate, # Guides, Allowance, Notes, Extras per day

#### 2. Pricing Summary

Full 1–30 PAX table. All input prices are tax-inclusive — no multiplier applied to costs.

Columns: PAX · Twins · Singles · Hotels ¥ · Services ¥ · Transports ¥ · Guides ¥ · Total Cost ¥ · Cost/PAX ¥ · Suggested ¥ · Your Price ¥ (editable) · EUR/PAX · Partner Comm ¥ · Gross Margin ¥/PAX · Gross Margin % · Tax on Price ¥/PAX · Tax on Cost ¥/PAX · Net Margin ¥/PAX · Total Net Margin ¥

Formulas:

```text
Twin/Single split:  twins = floor(PAX/2), singles = PAX % 2
Suggested Price:    Cost/PAX ÷ (1 − target markup%)
Partner Comm cost:  not taxable → commission × 1.0
                    taxable     → commission × 0.9  (input tax recovery)
Gross Margin/PAX:   Your Price − Cost/PAX − Partner Comm/PAX
Gross Margin %:     Gross Margin / Your Price
Tax on Price/PAX:   Your Price × taxRate%
Tax on Cost/PAX:    Cost/PAX × taxRate%
Net Margin/PAX:     Gross Margin/PAX − (Tax on Price − Tax on Cost)/PAX
Total Net Margin:   Net Margin/PAX × PAX
```

Margin color coding: green ≥ 20%, yellow ≥ 0%, red = loss

#### 3. Budgets

##### Hotels Budget

- Staff enters Twin Rooms, Single Rooms, Guide Rooms manually
- Hotels auto-populated from Builder, consecutive same city+type stays grouped into one row
- PAX ≈ indicator (twins × 2 + singles)
- Per-hotel cost breakdown + grand total

##### Services & Transports Budget

- Single PAX input drives both tables
- Total per line = `(Per Person × PAX × Qty) + (Per Group × Qty) + (Guide ¥ × Qty × numGuides)`
- Subtotals per section + combined total

### Catalog Files

| File | Contents | Columns |
| --- | --- | --- |
| `services_data.csv` | 329 services | name, group_price, adult_price, child_price |
| `transports_data.csv` | 501 transport routes | name, price |
| `cities_data.csv` | 93 cities ranked by foreign tourist popularity | city |

**Important:** CSV headers must be lowercase. The parser normalises headers to lowercase on load — `City`, `city`, `CITY` all work.

### Excel Export

6 sheets: Dashboard · Hotels · Services · Transports · Guide Days · Totals

### Drafts

Drafts are saved to/loaded from the Drafts API (see below), not localStorage. The **📁 Drafts** button opens a modal showing all saved drafts. **💾 Save Draft** saves current quote by name.

### Known Gotchas

- `loadCatalogs()` must be the last call in the script — removing it breaks the loading screen
- CSV parser must normalise line endings (`\r\n` → `\n`) and lowercase headers
- Autocomplete dropdowns use `position: fixed` anchored to input viewport coordinates (not `absolute`) to avoid clipping inside `overflow: hidden` table containers
- The Linode bucket serves files; `fetch()` uses relative URLs so all 4 files must be in the same bucket directory

---

## Tool 5 — Drafts API

**Live URL:** <https://drafts.cocolotravel.com>
**Repo:** github.com/axelder/internal-services (subfolder `/drafts`)
**Storage:** Linode Object Storage bucket `drafts` (jp-osa-1), prefix `quotes/`

### Purpose

Secure backend proxy for storing quote drafts. Credentials for Object Storage never reach the browser.

### Architecture

```text
Browser (quote tool)              VPS Docker container
quote.cocolotravel.com            drafts.cocolotravel.com:3099
        │                                   │
        │  x-api-key header  ─────────────► │  ◄──► Linode Object Storage
        │                                   │       drafts bucket (jp-osa-1)
        │                                   │       quotes/*.json
```

### Stack

- Node.js 20 + Express + `@aws-sdk/client-s3`
- Docker container on existing VPS
- Nginx Proxy Manager reverse proxy

### API Endpoints

| Method | Path | Description |
| --- | --- | --- |
| GET | `/health` | Health check (no auth) |
| GET | `/drafts` | List all drafts (sorted by date desc) |
| POST | `/drafts/:name` | Save a draft |
| GET | `/drafts/:name` | Load a draft |
| DELETE | `/drafts/:name` | Delete a draft |

All endpoints except `/health` require `x-api-key` header.

### Environment Variables (set in Portainer)

| Variable | Description |
| --- | --- |
| `API_KEY` | Shared secret — must match `DRAFTS_KEY` in the HTML tool |
| `LINODE_ACCESS_KEY` | Object Storage access key |
| `LINODE_SECRET_KEY` | Object Storage secret key |
| `BUCKET_NAME` | `drafts` |
| `BUCKET_REGION` | `jp-osa-1` |

### Deployment

```bash
# On VPS — one-time setup
cd /opt
git clone git@github.com:axelder/internal-services.git cocolo-drafts
cd cocolo-drafts/drafts
docker build -t cocolo-drafts:latest .
```

In Portainer: Stacks → Add Stack → paste `docker-compose.yml` → set env vars → Deploy.

To update after code changes:

```bash
cd /opt/cocolo-drafts && git pull
cd drafts && docker build -t cocolo-drafts:latest .
# Portainer: Stacks → cocolo-drafts → Recreate
```

### Nginx Proxy Manager Config

```text
Domain:           drafts.cocolotravel.com
Scheme:           http
Forward Host/IP:  <VPS host IP>   ← must be actual IP, NOT localhost
Forward Port:     3099
SSL:              Let's Encrypt, Force SSL enabled
```

> ⚠️ NPM runs in Docker so `localhost` refers to inside the NPM container. Always use the VPS host IP.

---

## Infrastructure Overview

| Service | URL | Hosting | Deploy method |
| --- | --- | --- | --- |
| Fax Generator | fax.cocolotravel.com | Linode Object Storage | Cyberduck upload |
| Quote Generator | quote.cocolotravel.com | Linode Object Storage | Cyberduck upload |
| Drafts API | drafts.cocolotravel.com | VPS Docker/Portainer port 3099 | git pull + docker build |
| Draft Storage | drafts.jp-osa-1.linodeobjects.com | Linode Object Storage | Written by Drafts API |
| Internal Wiki | wiki.cocolotravel.com | VPS | Outline |

---

## Development Patterns

### Standard tool architecture

```text
tool.html          ← single file, all HTML/CSS/JS
catalog.csv        ← data loaded via fetch() at runtime
```

### CSV loading pattern

```javascript
async function loadCatalogs() {
  const res = await fetch('catalog.csv');
  const text = await res.text();
  DATA = parseCSV(text);
  // Hide loading overlay
  // Call init()
}
loadCatalogs(); // ← MUST be last line in script
```

### CSV parser (handles Windows line endings + any header case)

```javascript
function parseCSV(text) {
  const lines = text.replace(/\r\n/g, '\n').replace(/\r/g, '\n').trim().split('\n');
  const headers = lines[0].split(',').map(h => h.trim().replace(/^"|"$/g, '').toLowerCase());
  // ...
}
```

### Draft save/load pattern

All tools use either localStorage (Name List, JR Tool) or the Drafts API (Quote Generator). The state is serialized as JSON — all DOM state is captured, not just form values.

### PDF generation

- **Fax Generator**: html2pdf.js with html2canvas `onclone` callback to fix overflow
- **Name List**: jsPDF + jsPDF-autotable
- **JR Tool**: jsPDF + html2canvas
- **Quote Generator PDF**: removed (caused CDN loading issues)

---

## Staff

| Name | Role |
| --- | --- |
| デルベ 真紀 | Registered travel business manager |
| 水迫伶菜 (Reina Mizusako) | Operations |
| Merouane | Operations |
| Shanshan | Operations |
