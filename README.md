# Automotive Paint Color Codes Scraper

> A production-grade web scraper that extracts automotive touch-up paint color codes from **[colorndrive.com](https://colorndrive.com/)** across **248 car manufacturers** and **145,000+ color entries**, built to bypass bot-protection layers and feed a structured MySQL database.

---

## The Business Problem

Automotive e-commerce and parts platforms need comprehensive, structured paint color data to help customers find the exact touch-up paint for their vehicle. This data is not available via any public API — it lives in paginated, JavaScript-rendered product pages on [colorndrive.com](https://colorndrive.com/) behind bot-detection (Cloudflare). Manually collecting it across hundreds of manufacturers would take months.

This tool automates the entire pipeline: from navigating protected pages, to structured extraction, to bulk-loading a normalized database — cutting that effort down to a single command.

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Browser automation | **Playwright (Chromium)** | Renders JS, supports persistent sessions for Cloudflare bypass |
| HTML parsing | **BeautifulSoup 4 + lxml** | Fast, reliable CSS-selector parsing |
| Data storage | **JSON** (intermediate) + **MySQL** (final) | Decouples scrape from import; safe to resume |
| DB import | **PHP 8 + PDO** | Matches the target platform's backend stack |
| Bot protection | **Persistent Chromium profile** | Survives challenge cookies across runs without re-solving |

---

## Architecture & Data Flow

```
source.txt (248 maker URLs)
    │
    ▼
[1] Maker Page  /en/{maker}-touch-up-paint
    │  parse_models() — dual-strategy link extraction
    ▼
[2] Model Page  /en/touch-up-paint/{maker}-{model}
    │  paginated: ?p=1, ?p=2 … until empty
    ▼
[3] parse_colors() — extract color code, name, alternative names
    │
    ▼
output.json  (~145,596 raw entries)
    │
    ▼
import_to_db.php — consolidate by (maker, color_code) → MySQL
```

Progress is checkpointed after every model page, so the scrape is fully resumable.

---

## Key Features

### Cloudflare Bypass — Persistent Browser Profile
Playwright launches a real Chromium instance with a persistent user data directory. On the first run, Cloudflare's JS challenge executes inside the real browser and the solved cookies are saved to disk. Every subsequent run reuses those cookies — no challenge, no re-solve, no external proxy needed.

```python
context = pw.chromium.launch_persistent_context(
    user_data_dir=BROWSER_DATA_DIR,
    args=["--disable-blink-features=AutomationControlled"],
)
```

### Dual-Strategy Model Link Detection
Many manufacturers structure their pages differently. `parse_models()` tries the primary DOM structure (`div.model-list`) and falls back to a full-page link scan if that container is absent or empty — catching edge cases that would silently produce zero results.

```python
def parse_models(html):
    # Primary: structured model list
    model_list_div = soup.find("div", class_="model-list")
    anchors = model_list_div.find_all("a", href=True) if model_list_div else []

    # Fallback: scan all /en/touch-up-paint/ links
    if not anchors:
        anchors = [a for a in soup.find_all("a", href=True)
                   if a["href"].startswith("/en/touch-up-paint/")
                   and a["href"] not in SKIP_MODEL_PATHS]
```

### Resumable Progress Tracking
`progress.json` caches model link lists per manufacturer and tracks fully-completed model URLs. Interrupt anytime — re-running resumes exactly where it stopped.

### Rate Limiting & Human-Like Delays
Randomized delays between requests (configurable min/max) prevent rate-limiting and reduce detection risk.

### Data Consolidation on Import
The PHP import script consolidates raw entries by `(maker, color_code)` — the same code can appear across multiple models. Models and alternative names are deduplicated and stored as JSON arrays, keeping the schema clean while preserving all relationships.

---

## Database Schema

**Table: `car_color_codes`**

| Column | Type | Description |
|---|---|---|
| `id` | INT (PK) | Auto-increment |
| `car_maker_id` | INT | FK to makers table (0 = unmapped) |
| `maker` | VARCHAR | Manufacturer name (e.g. `Toyota`) |
| `model` | JSON | Array of model names sharing this color code |
| `year` | INT / NULL | Model year — NULL (source data is year-agnostic) |
| `color_code` | VARCHAR | OEM paint code (e.g. `218`) |
| `color_description` | TEXT | Primary color name(s) joined with ` / ` |
| `alternative_names` | JSON | Regional name variants (US, Japan markets) |

**Consolidation logic:** Multiple raw rows with the same `(maker, color_code)` are merged into one DB record, with models and alternative names deduplicated into JSON arrays. This prevents duplicate codes while preserving cross-model associations.

---

## Output Sample

```json
[
  {
    "maker": "Toyota",
    "model": "4Runner",
    "color_code": "218",
    "color_name": "MIDNIGHT BLACK METALLIC",
    "alternative_names": ["MIDNIGHT BLACK METALLIC", "ATTITUDE BLACK MICA"]
  }
]
```

---

## Running the Scraper

```bash
# Install dependencies
pip install -r requirements.txt
playwright install chromium

# Full scrape (browser visible — required on first run for Cloudflare)
python scraper.py

# Resume after interruption
python scraper.py

# After first run succeeds — headless is safe
python scraper.py --headless

# Test with first 2 makers only
python scraper.py --test

# Target specific manufacturers
python scraper.py --makers toyota ford-america

# Custom request delay (seconds)
python scraper.py --delay-min 2 --delay-max 5

# Debug a manufacturer showing 0 models
python scraper.py --debug-maker saipa

# Wipe progress and start fresh
python scraper.py --reset
```

## Importing to MySQL

Configure database credentials via environment variables (see Configuration below), then:

```bash
php import_to_db.php
```

Output:
```
Connected to database.
Loaded 145596 raw entries from output.json.
Consolidated to 89412 unique (maker, color_code) pairs.
  Inserted 500 / 89412...
  ...
Done. Inserted: 89412  Skipped: 0  Total unique pairs: 89412
```

---

## Configuration

`import_to_db.php` reads connection details. Replace hardcoded values with environment variables before deploying:

```php
define('DB_HOST', getenv('DB_HOST'));
define('DB_NAME', getenv('DB_NAME'));
define('DB_USER', getenv('DB_USER'));
define('DB_PASS', getenv('DB_PASS'));
define('DB_PORT', (int) getenv('DB_PORT') ?: 3306);
```

---

## Challenges Solved

**1. Cloudflare bot protection**
Standard `requests`/`httpx` approaches return 403. Solution: a real Chromium browser with a persistent profile that stores solved challenge cookies — no paid proxy service needed.

**2. Inconsistent page structure across manufacturers**
~30% of manufacturers don't use the standard `div.model-list` container. The dual-strategy parser silently falls back to full-page link scanning, with a `--debug-maker` flag to inspect any outlier.

**3. Session poisoning from challenge pages**
Early runs cached empty model lists because Cloudflare returned a challenge page instead of content. Fixed with a cache-validation pattern: entries with 0 models are re-fetched rather than trusted.

**4. Scale and reliability**
145,596 entries across 248 manufacturers with paginated model pages. Checkpointing after each model page means any crash or network hiccup loses at most one page of work.

**5. Data normalization**
Raw scraped entries are 1-to-1 with model pages. The import step consolidates these into canonical `(maker, color_code)` records with deduplicated JSON arrays, avoiding duplicate rows while preserving all model associations.

---

## Project Structure

```
colorndrive2/
├── scraper.py          # Main scraper — Playwright + BeautifulSoup
├── import_to_db.php    # Consolidation + MySQL bulk import
├── source.txt          # 248 manufacturer URLs (manually curated)
├── requirements.txt
├── browser_data/       # Persistent Chromium profile (gitignored)
├── output.json         # Raw scraped data (~145K entries, gitignored)
└── progress.json       # Resume state (gitignored)
```

---

## Author

**Kashif Umar** — Backend Developer
