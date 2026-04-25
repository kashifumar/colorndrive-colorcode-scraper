# Automotive Color Codes Scraper & Data Pipeline

A production-grade Python data pipeline that crawls **[automotivetouchup.com](https://www.automotivetouchup.com/)**, a major automotive paint-code catalog, extracts **562,000+ raw color code records** across **50+ car manufacturers (1985–2025)**, and loads them into a MySQL database in two distinct schemas — raw and consolidated.

Built to feed a downstream vehicle customization SaaS that needed a structured, queryable paint-code dataset without a commercial API.

---

## The Business Problem

Paint touch-up retailers and automotive services often need a reliable, structured database of OEM color codes per make, model, and year. No public API exists for this data. The only option is to extract it from web sources, normalize it, and persist it in a form that supports fast lookups.

This pipeline was built to:
- Produce a complete dataset of OEM color codes for 50+ brands spanning 4 decades
- Support incremental re-scrapes without re-downloading already-processed pages
- Deliver two storage schemas: a raw audit trail and a deduplicated analytical view
- Run reliably on a single machine without triggering rate-limits or bans

---

## Architecture Overview

```
[automotivetouchup.com]
      │
      ▼
 scraper.py          ← Multi-level crawler with anti-blocking & resume support
      │
      ▼
 output.json         ← Raw flat records  (~562,561 entries)
      │
      ▼
 consolidate.py      ← Aggregation by (maker, color_code)
      │
      ▼
 consolidated.json   ← Deduplicated records (~199,111 entries)
      │
      ├──────────────────────────────────────┐
      ▼                                      ▼
import_to_db.py                 import_consolidated_to_db.py
(raw schema)                    (consolidated schema)
      │                                      │
      └──────────────┬───────────────────────┘
                     ▼
              MySQL / AWS RDS
           table: car_color_codes
```

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Language | Python 3.10+ | Mature scraping ecosystem, fast iteration |
| HTTP | `requests` + `Session` | Connection reuse, header control |
| Parsing | `BeautifulSoup4` + `lxml` | Robust HTML parsing; lxml is 3–5× faster than html.parser |
| Storage | MySQL / AWS RDS | Structured queries, team-accessible, hosted |
| DB Driver | `mysql-connector-python` | Official Oracle driver, `executemany` bulk inserts |
| Progress | JSON file | Portable, human-readable, no extra infrastructure |

---

## Key Features

### Resilient HTTP Layer
- **Rotating User-Agents** — cycles through 6 realistic browser fingerprints per request
- **Random delays** (configurable, default 3–7s) — mimics human browsing cadence
- **Exponential back-off on 429s** — waits 60–90s before retrying rate-limited requests
- **Per-attempt retry loop** — up to 3 attempts with 8–20s back-off on network errors
- **Session reuse** — persistent `requests.Session` with realistic browser headers

### Resumable Scraping
Progress is serialized to `progress.json` after every year-page scrape. If the process is interrupted (network failure, manual stop, crash), restarting it picks up exactly where it left off — no duplicate requests, no data loss.

```json
{
  "completed_year_urls": ["https://..."],
  "makers_list": [...],
  "touchup_urls": {...},
  "year_lists": {...}
}
```

### Multi-Level Navigation
The scraper handles a 4-level URL hierarchy automatically:

```
/paint-codes/                        → maker list
/paint-codes/{slug}.aspx             → resolves to touch-up-paint URL
/touch-up-paint/{slug}/              → year index
/touch-up-paint/{slug}/{year}/all-models/  → color table (with fallback)
```

Slug resolution includes a fallback for makers whose touch-up URL can't be found via anchor parsing.

### Cartesian Product Expansion
Some table rows contain multiple comma-delimited descriptions or multiple codes in a single cell. The parser performs a full Cartesian product expansion so every `code × description` combination becomes its own row — essential for downstream exact-match lookup.

### Flexible CLI
```bash
python scraper.py                              # Full run (or resume)
python scraper.py --test                       # 2 makers only — fast smoke test
python scraper.py --makers ford toyota honda   # Target specific brands
python scraper.py --delay-min 1 --delay-max 2  # Tune request pacing
python scraper.py --reset                      # Wipe progress and start clean
```

---

## Data Pipeline Stages

### Stage 1 — `scraper.py` → `output.json`

Flat list of records, one per color code per maker per year:

```json
[
  {
    "maker": "Toyota",
    "year": "2022",
    "color_code": "1F7",
    "color_description": "Silver Metallic"
  }
]
```

**Stats:** ~562,561 entries across 50+ manufacturers, years 1985–2025.

---

### Stage 2 — `consolidate.py` → `consolidated.json`

Groups by `(maker, color_code)` and collapses years and descriptions into sorted arrays:

```json
[
  {
    "maker": "Toyota",
    "color_code": "1F7",
    "years": [2005, 2006, 2007, 2008, 2009, 2010, 2011, 2012],
    "color_descriptions": ["Silver Metallic", "Classic Silver Metallic"]
  }
]
```

**Stats:** ~199,111 unique `(maker, color_code)` pairs — a 2.8× compression ratio.
This view is ideal for lookup tables, dropdowns, and deduplication use cases.

---

### Stage 3 — DB Import

Two import modes:

| Script | Table Schema | Use Case |
|---|---|---|
| `import_to_db.py` | `(maker, year INT, color_code, color_description)` | Full audit trail, per-year filtering |
| `import_consolidated_to_db.py` | `(maker, color_code, year JSON, color_description JSON)` | Compact lookup, deduplication |

Both use `executemany` for bulk insert performance.

---

## Database Schema

### Raw Table — `car_color_codes`

```sql
CREATE TABLE car_color_codes (
    id               INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    maker            VARCHAR(100)  NOT NULL,
    year             SMALLINT      NOT NULL,
    color_code       VARCHAR(50)   NOT NULL,
    color_description VARCHAR(255) NOT NULL
);
```

### Consolidated Table — `car_color_codes` (consolidated variant)

```sql
CREATE TABLE car_color_codes (
    id                INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    maker             VARCHAR(100) NOT NULL,
    color_code        VARCHAR(50)  NOT NULL,
    year              JSON         NOT NULL,  -- e.g. [2010, 2011, 2012]
    color_description JSON         NOT NULL   -- e.g. ["Silver Metallic"]
);
```

---

## Configuration

Copy `.env.example` to `.env` and set your database credentials:

```env
DB_HOST=your-rds-host.rds.amazonaws.com
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=your_database
```

The import scripts read from environment variables — never hardcode credentials in source files.

---

## Setup & Usage

```bash
# 1. Install dependencies
pip install -r requirements.txt
pip install mysql-connector-python

# 2. Configure environment
cp .env.example .env
# Edit .env with your DB credentials

# 3. Run the scraper
python scraper.py

# 4. Consolidate output
python consolidate.py

# 5. Import to database (choose one or both)
python import_to_db.py
python import_consolidated_to_db.py
```

---

## Challenges & Engineering Decisions

**Challenge: Rate limiting and bot detection**
automotivetouchup.com applies rate limits and monitors request patterns. Solved with rotating user-agents, randomized delays between every request, and an automatic 60–90s cool-down on 429 responses — keeping the scraper below the detection threshold over multi-hour runs.

**Challenge: Unreliable URL structures**
Not every maker follows the same URL pattern. The touch-up URL resolver first parses anchor tags looking for a canonical 2-segment path. If that fails, it falls back to a derived slug-based URL and logs a warning — the scrape continues rather than failing on a single maker.

**Challenge: Multi-value table cells**
Some rows encode multiple color codes or descriptions in one cell (comma-delimited or multi-anchor). Naively extracting the first value would silently drop data. The parser iterates all `codeLinks` anchors and splits description text, then applies a Cartesian product so every combination is captured.

**Challenge: Long-running interruptions**
A full scrape takes several hours. Network interruptions or manual stops would waste all progress without a resume mechanism. The progress file is written after every single year-page, so restarts are nearly zero-cost.

**Challenge: Two conflicting downstream needs**
The SaaS needed both a full audit trail (one row per code per year) and a compact lookup table (one row per unique code with year ranges). Rather than maintaining two scrapers, the consolidation step post-processes the raw output, keeping the scraper simple and the data models independent.

---

## Data Statistics

| File | Format | Entries |
|---|---|---|
| `output.json` | Flat records | ~562,561 |
| `consolidated.json` | Grouped records | ~199,111 |

- **Manufacturers covered:** 50+
- **Year range:** 1985–2025
- **Expansion ratio:** ~2.8× (raw vs. consolidated)

---

## Project Structure

```
automotivetouchup/
├── scraper.py                   # Main crawler
├── consolidate.py               # Raw → consolidated aggregation
├── import_to_db.py              # Imports output.json → MySQL
├── import_consolidated_to_db.py # Imports consolidated.json → MySQL
├── requirements.txt             # Python dependencies
├── .env.example                 # Credential template
├── output.json                  # Raw scraped data (gitignored)
├── consolidated.json            # Consolidated data (gitignored)
└── progress.json                # Resume tracker (gitignored)
```

---

## Author

**Kashif Umar** — Backend Developer

Specializing in data pipelines, web scraping, API development, and cloud-hosted database systems.

- LinkedIn: [linkedin.com/in/kashif-umar](https://www.linkedin.com/in/kashif-umar/)
- X (Twitter): [x.com/kashif_umar](https://x.com/kashif_umar)
