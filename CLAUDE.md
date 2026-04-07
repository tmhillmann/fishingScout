# CLAUDE.md

Guide for Claude when working on **MN Fishing Scout** — a Flask app for analyzing MN DNR CPUE (Catch Per Unit Effort) fishing survey data.

## What this app does

Helps the user find good fishing lakes in Minnesota by combining three data sources:
1. **Excel uploads** of CPUE data (pre-existing MN DNR format with multiple lakes per sheet)
2. **MN DNR LakeFinder metadata API** — for lake geometry, county, coordinates, species list
3. **MN DNR Lake Survey JSON API** — for actual survey results (CPUE, avg weight, counts per species/gear)

The user filters and searches lakes by species, CPUE, gear type, survey recency, lake size, depth, county, and distance from a point. The goal is identifying lakes with above-normal catch rates for target species.

## Project structure

```
fishing_app/
├── app.py                  # Flask backend (routes, parsers, DNR API clients)
├── templates/
│   └── index.html          # Single-page UI (all HTML/CSS/JS in one file)
├── requirements.txt        # flask, openpyxl, requests
├── species_map.json        # User-extensible species abbreviation → common name map
├── fishing.db              # SQLite database (auto-created; safe to delete to reset)
├── uploads/                # Saved Excel files from upload endpoint
└── README.md               # User-facing setup/usage docs
```

## Tech stack conventions

- **Python 3.12+** — user runs on Windows with Python 3.14
- **Flask** with server-rendered single HTML template. No React, no build step.
- **SQLite** via stdlib `sqlite3`. Schema migrations use `ALTER TABLE` with try/except for backwards compatibility.
- **Vanilla JS** in the template. No bundlers, no frameworks. All CSS in a single `<style>` block using CSS variables for theming.
- **Dark theme** with blue accent. CSS variables defined at the top of the template — stick to those when adding UI.
- The entire frontend is in `templates/index.html`. Do not split into separate JS/CSS files.

## Data sources & important URLs

1. **Lake metadata API** (returns JSON for a single lake):
   `http://services.dnr.state.mn.us/api/lakefinder/by_id/v1/?id={LAKE_ID}`
   Returns name, county, morphology (area/depth), coordinates, species list.

2. **Lake survey API** (returns all surveys for a lake):
   `https://maps.dnr.state.mn.us/cgi-bin/lakefinder/detail.cgi?type=lake_survey&id={LAKE_ID}`
   Returns `result.surveys[]` — each survey has `surveyType`, `surveyDate`, `fishCatchSummaries[]` with species abbreviations, gear, CPUE, weights, counts. **This is the primary source for survey data, not HTML scraping.**

3. **Lake report page** (human-readable, not scraped anymore):
   `https://www.dnr.state.mn.us/lakefind/showreport.html?downum={LAKE_ID}`

**Important:** Earlier versions of this app tried to scrape the HTML showreport page. That approach has been **removed**. Do not reintroduce BeautifulSoup or HTML scraping. The JSON API at `maps.dnr.state.mn.us` is the source of truth.

## Species abbreviations

The DNR's survey JSON returns species as 2-5 letter abbreviations (e.g. `WAE` = walleye, `SMB` = smallmouth bass, `RBS` = rainbow smelt). These are translated to common names via `SPECIES_MAP` in `app.py`.

**Three ways to extend the map**:
1. Add entries to the `SPECIES_MAP` dict constant in `app.py`
2. Create/edit `species_map.json` in the app directory (loaded on startup AND on each add-lake operation)
3. `POST /api/species_map` with `[{"abbreviation": "XYZ", "species": "Name"}]` — this both persists to `species_map.json` AND retranslates existing database records in-place

**Gotcha**: historically the species map was only loaded once at startup. It now reloads from the file on every `fetch_survey_data()` call, so edits to `species_map.json` are picked up without restarting. But **existing DB records with raw abbreviations don't auto-update** — use the POST endpoint to retranslate them, or delete & re-add the lake.

## Database schema

Two tables:

**`lakes`** — one row per lake, keyed by DOW number (8-digit string).
Columns: `id, name, county, nearest_town, area, max_depth, mean_depth, littoral_area, shore_length, latitude, longitude, fish_species, metadata_json, last_updated`

**`survey_data`** — one row per species/gear/survey combination.
Columns: `id, lake_id, lake_name, survey_year, survey_date, survey_type, species, gear, cpue, normal_range_cpue, avg_weight, normal_range_weight, count, source_sheet, uploaded_at`

The `survey_type` column is key: values are `"Standard Survey"`, `"Targeted Survey"`, `"Population Assessment"`, etc. The default search filter only shows Standard Surveys, and the default display shows only the most recent survey per lake.

**Schema migrations**: `init_db()` runs `CREATE TABLE IF NOT EXISTS` plus defensive `ALTER TABLE ADD COLUMN` calls wrapped in try/except. When adding new columns, follow this pattern so existing user databases don't break.

## Key architectural decisions

**Default search behavior** (these are conventions to preserve unless the user asks to change them):
- Surveys older than 20 years are excluded by default (`min_year = current_year - 20`)
- Only "Standard Survey" records are shown (`survey_type=standard` default)
- Only the most recent survey per lake is shown (`most_recent=true` default)
- Results are sorted by `lake_id ASC, lake_name ASC` — because lakes can share names, ID disambiguates them
- Lake ID is shown as the last column in search results (muted styling)

**Auto-search on filter change**: When any filter changes in the Search tab, the search re-runs automatically. Text/number inputs are debounced 400ms; dropdowns and checkboxes fire immediately. Preserve this behavior — don't add explicit "Search" buttons that the user has to click.

**Distance filter**: Uses Haversine formula in Python (not SQL) as a post-filter. Requires lake metadata to have been refreshed so `latitude`/`longitude` are populated. When active, results sort by distance instead of the default.

**Above-normal CPUE filter**: Post-filter that parses the `normal_range_cpue` string (format: `"1.4-13.8"`) and keeps only records where CPUE exceeds the high end.

## Excel upload parser

`parse_cpue_excel()` in `app.py` handles a specific multi-lake-per-sheet format:
- Each lake block starts with a header row: `lake_name | year | "ID" | dow_number`
- Followed by a column header row: `Species | Gear | CPUE | Normal Range | Avg Weight | Normal Range | Count`
- Data rows until a blank row separates lakes
- Multiple sheets per workbook are supported

The sample file `Fishing_Data.xlsx` in `/mnt/project/` has sheets "CPUE Data - Todd" and "CPUE Data - Toby" in this format. Don't break this parser — the user has existing data.

## Frontend tab structure

Three tabs, switched with the `.tab` buttons in the header:
1. **Upload** — drag/drop Excel files
2. **Lakes** — list of lakes with refresh/delete actions + "Add by DOW number" form
3. **Search & Filter** — the main tool; filter grid + results table

Modal dialog for lake detail opens when clicking a lake card or a search result row.

## Things to avoid

- **Don't reintroduce HTML scraping.** The BeautifulSoup-based scraper was removed in favor of the JSON API. If the JSON API fails for a lake, tell the user — don't silently fall back to HTML.
- **Don't split the template.** The user works across multiple sessions and a single `index.html` keeps diffs readable.
- **Don't add build steps** (webpack, Vite, TypeScript, etc.). Vanilla JS + Flask templates is the contract.
- **Don't add authentication.** This is a personal local tool.
- **Don't use emojis gratuitously in code.** The UI uses a few (🐟, 📂, 🔄, 🗑️, 📍) but keep it restrained.
- **Don't assume the DB is fresh.** Always use `CREATE TABLE IF NOT EXISTS` and `ALTER TABLE` with try/except for new columns.
- **Don't change the default sort** (lake_id, name) without being asked. The user specifically requested it because of duplicate lake names.

## Testing approach

The app is run locally by the user with `python app.py` on Windows. There's no automated test suite. When making changes:
1. Verify `app.py` imports cleanly with `python -c "from app import app"`
2. Spin up the test client and hit affected routes
3. For parser changes, test against `/mnt/project/Fishing_Data.xlsx`
4. For DNR API changes, remember the sandbox here has no network — the user verifies live

## User's environment

- Windows with OneDrive sync (`C:\Users\hillm\OneDrive\Fishing\`)
- Python 3.14
- Runs `python app.py` directly (Flask dev server on port 5000)
- Location: Minneapolis, MN — uses the distance filter for local trip planning

## Common requests & where to make changes

| Request | File(s) to edit |
|---|---|
| New filter on search results | `api_search()` route in `app.py` + filter grid in `index.html` + `doSearch()` JS |
| New column in results table | The table header and row templates in `doSearch()` AND `sortResults()` in `index.html` (both need updates — they duplicate rendering) |
| New route | `app.py` near the other `@app.route` decorators |
| Species translation | `SPECIES_MAP` in `app.py` or `species_map.json` |
| DB schema change | `init_db()` — add to `CREATE TABLE` AND add an `ALTER TABLE` migration block |
| New lake action (refresh, delete, etc.) | Backend route in `app.py` + button in lake card template in `index.html` + handler JS function |

## Gotchas I've hit

- **Duplicated rendering**: The results table HTML is built in two places in `index.html` — once in `doSearch()` and once in `sortResults()`. If you add/change a column, update both.
- **Filter options not reloading**: `loadFilterOptions()` used to have a "loaded once" flag that prevented refresh after uploads. It's been removed — always fetch fresh.
- **Schema changes on existing DBs**: Don't rely on users deleting `fishing.db`. Use the ALTER TABLE migration pattern.
- **Species map staleness**: Editing `species_map.json` doesn't retroactively update records already in the DB. Document this when the user asks about the map.
- **Lake IDs with leading zeros**: DOW numbers are 8-digit strings; some have leading zeros. The add-by-ID endpoint zero-pads inputs. Never convert lake IDs to int.
- **The `metadata_json` column**: Stores the raw DNR API response as a JSON string. Don't try to parse it in queries — it's there for debugging/future use.
