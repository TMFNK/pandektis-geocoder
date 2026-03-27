# Greece-GEO: Settlement Renamings Geocoding Pipeline

## Specification v1.0

---

## 1. Business Need

### Problem

The Greek National Documentation Centre (EKT) maintains **PANDEKTIS**, a digital archive of primary sources for Greek history and culture. One of its collections — *"Name Changes of Settlements in Greece"* — contains **4,413 records** documenting official settlement renamings published in the Government Gazette (ΦΕΚ) between the early 1900s and the present.

Each record includes the old name (often Turkish, Slavic, or archaic Greek), the new name, the renaming date, the prefecture, province, and municipality. However, the dataset has **no geographic coordinates**, making it unusable for GIS analysis, spatial visualization, or mapping applications.

### Goal

Build an automated pipeline that:

1. **Scrapes** all 4,413 settlement renaming records from PANDEKTIS
2. **Geocodes** each record using the new settlement name and prefecture
3. **Exports** the results as GeoJSON (RFC 7946) and Excel (.xlsx)

### Use Cases

- Historical GIS analysis of Greek toponymic changes
- Overlaying settlement renamings on political/administrative boundary maps
- Academic research into Hellenization of place names
- Integration with open-data platforms (QGIS, Kepler.gl, Mapbox)
- Manual auditing via spreadsheet format

---

## 2. Data Source

### Source URL

```
https://pandektis.ekt.gr/pandektis/handle/10442/4968/browse?type=title&sort_by=1&order=ASC&rpp=100
```

### Entry Page Example

```
https://pandektis.ekt.gr/pandektis/handle/10442/34731
```

### HTML Structure

The site uses a handle-based URL system with server-rendered HTML (no JavaScript frameworks). Each entry page contains paired Greek/English metadata fields rendered in `<div>` elements with CSS classes:

```html
<div class="ekt_met_curved">Νέα ονομασία&nbsp:</div>
<div class="ekt_met_curved_metadata"> Καλλιθέα</div>

<div class="ekt_met_curved">New name&nbsp:</div>
<div class="ekt_met_curved_metadata"> Kallithea</div>
```

### Available Fields Per Entry

| Greek Label              | English Label                        | Example Value      |
|--------------------------|--------------------------------------|--------------------|
| Νομός                   | Prefecture                           | THESSALONIKI       |
| Επαρχία                 | Province                             | THESSALONIKI       |
| Αυτοδιοικητική μονάδα   | Community or Municipality            | Community          |
| Κοινότητα ή Δήμος       | Name of Community or Municipality    | Thermi             |
| Κωδικός οικισμού        | Code of settlement                   | 54112331           |
| Παλαιά ονομασία         | Old name                             | 21 Apriliou        |
| Ημερομηνία μετονομασίας | Date of renaming                     | 21/5/1975          |
| ΦΕΚ                     | Official Journal                     | 96/1975            |
| Νέα ονομασία            | New name                             | Kallithea          |

### Browse Pagination

The browse endpoint returns up to 100 entries per page (`rpp=100`) with `offset`-based pagination. The server wraps around after the last entry instead of returning empty results, so the scraper must detect duplicate URLs to know when to stop.

---

## 3. Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    scrape_pandektis.py                       │
│                                                             │
│  Phase 1: URL Collection                                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Browse pages (rpp=100) → collect 4,413 entry URLs     │  │
│  │ Duplicate detection for wrap-around handling          │  │
│  └───────────────────────────────┬───────────────────────┘  │
│                                  │                          │
│  Phase 2: Scraping              ▼                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ For each URL: fetch page, parse metadata              │  │
│  │ CSS class extraction: ekt_met_curved_metadata         │  │
│  │ Retry logic: 3 attempts with exponential backoff      │  │
│  │ Follow modern settlement links when present           │  │
│  └───────────────────────────────┬───────────────────────┘  │
│                                  │                          │
│  Phase 3: Geocoding             ▼                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Multi-strategy Nominatim queries:                     │  │
│  │   1. English name + prefecture                        │  │
│  │   2. Greek name + prefecture                          │  │
│  │   3. Municipality (EN) + prefecture                   │  │
│  │   4. Municipality (GR) standalone                     │  │
│  │   5. Greek municipality nominative form               │  │
│  │   6. Greek name + Greek prefecture                    │  │
│  │   7. English/Greek name alone                         │  │
│  │ Prefecture bounding box validation (52 prefectures)   │  │
│  │ Rate limit: 1.1s between requests                     │  │
│  └───────────────────────────────┬───────────────────────┘  │
│                                  │                          │
│  Phase 4: Export                ▼                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ GeoJSON (RFC 7946) — [lon, lat] order                │  │
│  │ Excel (.xlsx) — formatted headers, frozen panes       │  │
│  │ Sample JSON — first 5 geocoded records                │  │
│  │ Failed geocoding CSV — for manual review              │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  Resume Support: progress.json checkpoint                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Components

### 4.1 `scrape_pandektis.py` — Main Pipeline Script

**Language:** Python 3.14+
**Dependencies:** `requests`, `beautifulsoup4`, `lxml`, `openpyxl`
**Virtual Environment:** `.venv/` (project-local, no system packages)

**CLI Interface:**

```bash
python3 scrape_pandektis.py                       # Full run (scrape + geocode)
python3 scrape_pandektis.py --max-entries 50      # Test with N entries
python3 scrape_pandektis.py --skip-scrape          # Resume from progress.json
python3 scrape_pandektis.py --skip-geocode         # Scrape only, no geocoding
python3 scrape_pandektis.py --skip-scrape --skip-geocode  # Export only
```

**Phases:**

| Phase | Function | Description |
|-------|----------|-------------|
| 1 | `collect_entry_urls()` | Paginate browse pages, collect all entry URLs with duplicate detection |
| 2 | `parse_entry_page()` | Fetch each entry page, extract 9+ fields via CSS class selectors |
| 2 | `follow_modern_link()` | Some entries link to modern settlement records for updated names |
| 3 | `geocode_settlement()` | Multi-strategy Nominatim geocoding with fallback chain |
| 3 | `_nominatim_search()` | Direct HTTP calls to Nominatim API |
| 3 | `_validate_bounds()` | Coordinate validation against Greece + prefecture bounding boxes |
| 3 | `_greek_genitive_to_nominative()` | Heuristic Greek morphology converter |
| 3 | `_prefecture_to_greek()` | English prefecture name → Greek genitive form |
| 4 | `export_geojson()` | RFC 7946 FeatureCollection with CRS84 |
| 4 | `export_excel()` | Formatted .xlsx with headers, borders, frozen panes |
| 4 | `export_sample()` | First 5 geocoded records as compact JSON |

### 4.2 Output Files

| File | Format | Description |
|------|--------|-------------|
| `greece_renamings.geojson` | GeoJSON | Primary deliverable — all geocoded records |
| `greece_renamings.xlsx` | Excel | Secondary deliverable — all records for manual audit |
| `sample_5_rows.json` | JSON | Demonstration sample with Greek characters |
| `failed_geocoding.csv` | CSV | Entries that couldn't be geocoded (for manual review) |
| `progress.json` | JSON | Resume checkpoint (entry URLs + scraped records) |

### 4.3 GeoJSON Feature Schema

```json
{
  "type": "Feature",
  "geometry": {
    "type": "Point",
    "coordinates": [22.943366, 40.646679]
  },
  "properties": {
    "old_name_gr": "21η Απριλίου",
    "old_name_en": "21 Apriliou",
    "new_name_gr": "Καλλιθέα",
    "new_name_en": "Kallithea",
    "prefecture": "THESSALONIKI",
    "province": "THESSALONIKI",
    "renaming_date": "21/5/1975",
    "settlement_code": "54112331",
    "official_journal": "96/1975",
    "municipality_en": "Thermi",
    "municipality_gr": "Θέρμης",
    "source_url": "https://pandektis.ekt.gr/pandektis/handle/10442/34731"
  }
}
```

**Coordinate order:** `[longitude, latitude]` per RFC 7946 / WGS84 CRS84
**Decimal precision:** 6 decimal places (~0.1m accuracy)
**Encoding:** UTF-8 (Greek characters preserved)

---

## 5. Geocoding Strategy

### Problem

Settlement names are historical. Old names (often Turkish or Slavic) don't exist in modern geocoding databases. Even new names can be ambiguous — "Agios Georgios" appears in dozens of Greek prefectures.

### Approach

Use the **new English name + prefecture** as the primary geocoding query, with progressive fallbacks.

### Query Chain (in priority order)

| # | Strategy | Label | Example Query |
|---|----------|-------|---------------|
| 1 | EN name + prefecture | `en+pref` | `Kallithea, THESSALONIKI, Greece` |
| 2 | EN name alone | `en` | `Kallithea, Greece` |
| 3 | GR name + prefecture | `gr+pref` | `Καλλιθέα, THESSALONIKI, Greece` |
| 4 | GR name alone | `gr` | `Καλλιθέα, Greece` |
| 5 | Municipality EN + prefecture | `mun_en+pref` | `Thermi, THESSALONIKI, Greece` |
| 6 | Municipality GR alone | `mun_gr` | `Θέρμης, Greece` |
| 7 | Municipality GR + prefecture | `mun_gr+pref` | `Θέρμης, THESSALONIKI, Greece` |
| 8 | Municipality nominative form | `mun_nom` | `Αχόμαυρος, Greece` (from Αχομαύρου) |
| 9 | GR name + GR prefecture | `gr_grpref` | `Καλλιθέα, Θεσσαλονίκης, Greece` |

### Validation

- **Greece bounds:** lat 34.0–42.0, lon 19.0–30.0
- **Prefecture bounds:** 52 bounding boxes defined in `PREFECTURE_BOUNDS` dict
- **Tolerance:** ±0.3° margin for boundary settlements
- **Fallback:** If result is in Greece but outside expected prefecture, accept with `~` suffix marker

### Performance

| Metric | Value |
|--------|-------|
| Primary strategy success rate | ~72% |
| Multi-strategy success rate | ~96% |
| Rate limit | 1.1s between requests (Nominatim policy) |
| Geocoding provider | OpenStreetMap Nominatim (free, no API key) |

### Known Limitations

- Nominatim coverage of small Greek villages is incomplete
- Some common names (e.g., "Galini") resolve to wrong prefectures
- 3–4% of entries are genuinely un-geocodable via free services
- Greek genitive→nominative conversion is heuristic (covers ~80% of patterns)

---

## 6. Encoding & Character Handling

All files are UTF-8 encoded. Greek characters are preserved throughout the pipeline:

| Field | Script | Example |
|-------|--------|---------|
| `old_name_gr` | Greek | `21η Απριλίου` |
| `old_name_en` | Latin | `21 Apriliou` |
| `new_name_gr` | Greek | `Καλλιθέα` |
| `new_name_en` | Latin | `Kallithea` |
| `municipality_gr` | Greek | `Θέρμης` |

The source website serves UTF-8 HTML. BeautifulSoup parses with `lxml` parser which handles UTF-8 natively. JSON output uses `ensure_ascii=False`. Excel output uses openpyxl's default UTF-8 handling.

---

## 7. Rate Limiting & Politeness

| Target | Delay | Rationale |
|--------|-------|-----------|
| Pandektis scraping | 0.4s between pages | ~2.5 req/s, polite for academic source |
| Nominatim geocoding | 1.1s between requests | Required by Nominatim usage policy (max 1 req/s) |
| Nominatim retry | 2s, 4s, 8s exponential | On timeout or 5xx errors |
| Pandektis retry | 2s, 4s, 8s exponential | On 500 Internal Server Error |

User-Agent string identifies the scraper as academic research:
```
Greece-GEO-Research-Scraper/1.0 (academic; settlement-geocoding)
```

---

## 8. Resume Support

The scraper saves progress to `progress.json` after scraping and after geocoding. This enables:

- Resuming interrupted runs: `python3 scrape_pandektis.py --skip-scrape`
- Re-geocoding from saved records: `python3 scrape_pandektis.py --skip-scrape`
- Incremental updates: scrape new entries, then re-run geocoding on all

Progress file structure:

```json
{
  "entry_urls": [{"url": "...", "display_title": "..."}],
  "records": [{"old_name_gr": "...", "new_name_en": "...", ...}],
  "timestamp": "2026-03-25 12:34:56"
}
```

---

## 9. Testing

### Test Modes

```bash
# Quick smoke test (5 entries, ~30 seconds)
python3 scrape_pandektis.py --max-entries 5

# Medium test (50 entries, ~5 minutes)
python3 scrape_pandektis.py --max-entries 50

# Scrape-only test (no geocoding, fast)
python3 scrape_pandektis.py --max-entries 100 --skip-geocode
```

### Verified Results (50-entry test)

| Metric | Result |
|--------|--------|
| Scraping success | 50/50 (100%) |
| Geocoding success | 48/50 (96%) |
| Greek character handling | Verified |
| GeoJSON validity | RFC 7946 compliant |
| Excel export | 50 rows, formatted |
| Coordinate accuracy | Manual spot-check passed |

---

## 10. Dependencies

```
requests>=2.32       HTTP client
beautifulsoup4>=4.14 HTML parser
lxml>=6.0            Fast XML/HTML backend for BeautifulSoup
openpyxl>=3.1        Excel file generation
```

No external API keys required. No `geopy` dependency (removed in favor of direct Nominatim HTTP calls for better control).

---

## 11. File Inventory

```
Greece-GEO/
├── scrape_pandektis.py          Main pipeline script
├── SPEC.md                      This document
├── .venv/                       Python virtual environment
├── greece_renamings.geojson     Output: GeoJSON (generated)
├── greece_renamings.xlsx        Output: Excel (generated)
├── sample_5_rows.json           Output: sample (generated)
├── failed_geocoding.csv         Output: failures log (generated)
└── progress.json                Resume checkpoint (generated)
```

---

## 12. Estimated Full Run

| Phase | Entries | Est. Time |
|-------|---------|-----------|
| URL collection | 4,413 | ~20 seconds |
| Scraping | 4,413 @ 0.4s | ~30 minutes |
| Geocoding | ~4,240 @ 1.1s (96%) | ~78 minutes |
| Export | — | ~5 seconds |
| **Total** | — | **~2 hours** |

---

## 13. Future Improvements

1. **Additional geocoding providers** — LocationIQ, HERE, or Google (require API keys) for the remaining 4% failures
2. **Parallel scraping** — concurrent requests with configurable thread pool for Phase 2
3. **GeoPackage export** — for direct QGIS integration
4. **Temporal filtering** — filter renamings by decade or date range
5. **Prefecture boundary overlay** — validate geocoded points against actual Kallikratis administrative boundaries
6. **Greek morphological analyzer** — replace heuristic genitive→nominative with proper dictionary lookup
