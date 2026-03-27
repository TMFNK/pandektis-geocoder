# Greece-GEO

Scrape and geocode settlement renaming records from [PANDEKTIS](https://pandektis.ekt.gr/) — the Greek National Documentation Centre's digital archive.

Extracts ~4,413 settlement renaming records (old name → new name, prefecture, date, official journal), geocodes them via [OpenStreetMap Nominatim](https://nominatim.openstreetmap.org/), and exports as GeoJSON + Excel.

## What It Does

| Phase | Description |
|-------|-------------|
| **Scrape** | Collects all entry URLs from the browse pagination, then parses each entry page for metadata |
| **Geocode** | Multi-strategy Nominatim queries with 9 fallback strategies and prefecture bounding box validation |
| **Export** | RFC 7946 GeoJSON, formatted Excel (.xlsx), sample JSON, and failed geocoding CSV |

## Quick Start

```bash
# Clone and enter the repo
git clone https://github.com/youruser/Greece-GEO.git
cd Greece-GEO

# Create virtual environment and install dependencies
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Quick smoke test (5 entries, ~30 seconds)
python3 scrape_pandektis.py --max-entries 5

# Full run (~2 hours)
python3 scrape_pandektis.py
```

## CLI Options

```bash
python3 scrape_pandektis.py                       # Full run (scrape + geocode)
python3 scrape_pandektis.py --max-entries 50      # Test with N entries
python3 scrape_pandektis.py --skip-scrape          # Resume from progress.json
python3 scrape_pandektis.py --skip-geocode         # Scrape only, no geocoding
python3 scrape_pandektis.py --skip-scrape --skip-geocode  # Export only
```

## Output Files

| File | Description |
|------|-------------|
| `greece_renamings.geojson` | GeoJSON FeatureCollection (RFC 7946, CRS84) |
| `greece_renamings.xlsx` | Excel workbook with formatted headers |
| `sample_5_rows.json` | First 5 geocoded records |
| `failed_geocoding.csv` | Entries that couldn't be geocoded |
| `progress.json` | Resume checkpoint |

## GeoJSON Feature Schema

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
    "renaming_date": "21/5/1975",
    "settlement_code": "54112331",
    "official_journal": "96/1975"
  }
}
```

## Geocoding Strategy

Uses the new settlement name + prefecture as the primary query, with progressive fallbacks through Greek names, municipality names, and genitive→nominative morphological conversion. Coordinates are validated against 52 prefecture bounding boxes.

| Strategy | Example |
|----------|---------|
| `en+pref` | `Kallithea, THESSALONIKI, Greece` |
| `gr+pref` | `Καλλιθέα, THESSALONIKI, Greece` |
| `mun_en+pref` | `Thermi, THESSALONIKI, Greece` |
| `gr_grpref` | `Καλλιθέα, Θεσσαλονίκης, Greece` |

~96% geocoding success rate. No API keys required.

## Rate Limiting

- Pandektis: 0.4s between pages (~2.5 req/s)
- Nominatim: 1.1s between requests (per usage policy)
- User-Agent identifies the scraper as academic research

## Dependencies

- `requests` — HTTP client
- `beautifulsoup4` — HTML parser
- `lxml` — Fast HTML backend for BeautifulSoup
- `openpyxl` — Excel file generation

No external API keys. Python 3.14+.

## License

Data sourced from [PANDEKTIS](https://pandektis.ekt.gr/), operated by the Greek National Documentation Centre (EKT). The original dataset is public.
