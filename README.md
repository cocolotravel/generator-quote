# Cocolo Travel — Quote Generator

Web-based tour pricing calculator for Cocolo Travel staff. Built as a single HTML file with no build step required.

**Live URL:** <https://quote.cocolotravel.com>
**Last updated:** June 2026

## Files

| File | Description |
| --- | --- |
| `index.html` | Main application (UI + logic) |
| `cities_data.csv` | List of available destination cities (93 cities) |

Services and transport catalogs are loaded at runtime from the Xano API via the Drafts API proxy (see below). Only `cities_data.csv` needs to be in the Linode bucket alongside `index.html`.

## Usage

Open `index.html` in a browser. The tool fetches all catalog data on load — no local server needed.

---

<!-- markdownlint-disable MD024 -->

## Quote Generator

**Company:** Cocolo Travel (ここロトラベル合同会社)
**Location:** Ebisu, Tokyo

### Architecture

- Single HTML file + `cities_data.csv` in Linode Object Storage
- Cities loaded via `fetch()` from the bucket
- Services and transports loaded via the Drafts API Xano proxy (authenticated, credentials never reach the browser)
- Draft persistence via Drafts API
- Excel and Markdown pro-forma export built-in
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
- City: dropdown from `cities_data.csv`
- Hotel Name: free text
- Meals, Room Type: dropdowns
- Twin ¥ / Single ¥ / Guide ¥: per-night prices (tax-inclusive)
- Single Supplement: auto-calculated = `(Single × 2) − Twin`
- Drag-to-reorder rows

##### Services section

- Day number → City auto-filled from matching hotel night (read-only, not a dropdown)
- Service name: searchable autocomplete from Xano catalog, with custom entry fallback
- Per Person ¥, Per Group ¥, Guide ¥, Qty
- Drag-to-reorder rows

##### Transports section

- Same pattern as Services, catalog loaded from Xano
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

### Exports

- **Excel** — 6 sheets: Dashboard · Hotels · Services · Transports · Guide Days · Totals
- **Pro-forma** — Markdown file with itinerary, hotels, services, pricing table. Filename = `[Tour Name] - pro-forma.md`

### Drafts

Saved to/loaded from the Drafts API, not localStorage. The **📁 Drafts** button opens a modal with search and save. **💾 Save Draft** saves the current quote by name.

### Known Gotchas

- `loadCatalogs()` must be the last call in the script — removing it breaks the loading screen
- `cities_data.csv` must be in the same Linode bucket directory as `index.html`
- Autocomplete dropdowns use `position: fixed` anchored to viewport coordinates (not `absolute`) to avoid clipping inside `overflow: hidden` table containers

---

## Drafts API

**Live URL:** <https://drafts.cocolotravel.com>
**Repo:** github.com/axelder/internal-services (subfolder `/drafts`)
**Server path:** `/opt/main-stack/drafts`
**Storage:** Linode Object Storage bucket `drafts` (jp-osa-1), prefix `quotes/`

### Purpose

- Stores and retrieves quote drafts (JSON) in Linode Object Storage
- Proxies Xano API requests for services and transports — Xano credentials never reach the browser

### Architecture

```text
Browser (quote tool)              VPS Docker container
quote.cocolotravel.com            drafts.cocolotravel.com
        │                                   │
        │  x-api-key header  ─────────────► │  ◄──► Linode Object Storage (drafts)
        │                                   │  ◄──► Xano API (services, transports)
```

### Stack

- Node.js 20 + Express + `@aws-sdk/client-s3`
- Docker container on VPS, managed via `docker compose` + `.env` file
- Nginx reverse proxy

### API Endpoints

| Method | Path | Description |
| --- | --- | --- |
| GET | `/health` | Health check (no auth) |
| GET | `/drafts` | List all drafts (sorted by date desc) |
| POST | `/drafts/:name` | Save a draft |
| GET | `/drafts/:name` | Load a draft |
| DELETE | `/drafts/:name` | Delete a draft |
| GET | `/xano/services` | Fetch services catalog from Xano (1 hr cache) |
| GET | `/xano/transports` | Fetch transports catalog from Xano (1 hr cache) |

All endpoints except `/health` require `x-api-key` header.

As of October 2026, the quote tool no longer calls this API directly from the browser — it goes through a `netlify/functions/drafts.js` proxy on the Netlify site, which keeps the key server-side (set as the `DRAFTS_API_KEY` Netlify environment variable). See [generator-namelist](https://github.com/cocolotravel/generator-namelist) for the same pattern, applied first there after its repo went public.

### Environment Variables (`.env` file)

| Variable | Description |
| --- | --- |
| `API_KEY` | Shared secret — must match the `DRAFTS_API_KEY` Netlify environment variable (previously embedded directly as `DRAFTS_KEY` in the HTML tool; no longer is) |
| `LINODE_ACCESS_KEY` | Object Storage access key |
| `LINODE_SECRET_KEY` | Object Storage secret key |
| `BUCKET_NAME` | `drafts` |
| `BUCKET_REGION` | `jp-osa-1` |
| `XANO_EMAIL` | Xano account email used to authenticate the proxy |
| `XANO_PASSWORD` | Xano account password |

### Deployment

```bash
# On VPS — one-time setup
cd /opt/main-stack/drafts
git pull
docker build -t cocolo-drafts:latest .
docker compose up -d
```

To update after code changes:

```bash
cd /opt/main-stack/drafts && git pull
docker build -t cocolo-drafts:latest .
docker compose up -d --force-recreate
```

### Nginx Config

```text
Domain:           drafts.cocolotravel.com
Forward:          http://<VPS host IP>:3099
SSL:              Let's Encrypt, Force SSL enabled
```

> ⚠️ If Nginx runs in Docker, use the host IP — not `localhost`.

---

## Infrastructure

| Service | URL | Hosting | Deploy method |
| --- | --- | --- | --- |
| Quote Generator | quote.cocolotravel.com | Linode Object Storage | Cyberduck upload |
| Drafts API | drafts.cocolotravel.com | VPS Docker port 3099 | git pull + docker build |
| Draft Storage | drafts.jp-osa-1.linodeobjects.com | Linode Object Storage | Written by Drafts API |

---

## Staff

| Name | Role |
| --- | --- |
| デルベ 真紀 | Registered travel business manager |
| 水迫伶菜 (Reina Mizusako) | Operations |
| Merouane | Operations |
| Shanshan | Operations |
