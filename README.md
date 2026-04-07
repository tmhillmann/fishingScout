# MN Fishing Scout

A Flask application for analyzing MN DNR CPUE (Catch Per Unit Effort) survey data to find great fishing lakes.

## Features

- **Upload CPUE Data**: Import Excel files with MN DNR survey data (species, gear, CPUE, weight, counts)
- **Lake Metadata**: Fetch and view lake details (area, depth, county, species) from the MN DNR LakeFinder API
- **Advanced Filtering**: Search lakes by species, CPUE, average weight, gear type, survey recency, lake size, and depth
- **Above-Normal Detection**: Identify lakes where species CPUE exceeds the normal range
- **20-Year Default**: Surveys older than 20 years are excluded by default (configurable)
- **Sortable Results**: Click column headers to sort search results

## Setup

```bash
# Install Python 3.12+ if needed
python3 --version

# Install dependencies
pip install -r requirements.txt

# Run the app
python app.py
```

Then open http://localhost:7002 in your browser.

## Usage

### 1. Upload CPUE Data
Go to the **Upload** tab and drag/drop your Excel file. The parser expects the MN DNR format:

```
Lake Name    Year    ID    DNR_ID
Species      Gear    CPUE  Normal Range  Avg Weight  Normal Range  Count
walleye      Standard gill nets  13.44  2.3-18.1  2.16  1.0-2.3  121
...
(blank row separates lakes)
```

### 2. Refresh Lake Metadata
Go to the **Lakes** tab and click "Refresh All Metadata from DNR" to pull lake details (area, depth, county, coordinates, fish species list) from the MN DNR LakeFinder API.

The API endpoint used: `http://services.dnr.state.mn.us/api/lakefinder/by_id/v1/?id=LAKE_ID`

### 3. Search & Filter
Go to the **Search & Filter** tab to query your data:

- **Species**: Filter by specific fish species
- **Gear**: Filter by survey method (gill nets, trap nets, etc.)
- **CPUE Range**: Set minimum/maximum catch per unit effort
- **Avg Weight**: Filter by average fish weight
- **Survey Year**: Override the 20-year default to narrow or expand the date range
- **Lake Area / Depth**: Filter by physical lake characteristics (requires metadata refresh)
- **Above Normal Only**: Show only records where CPUE exceeds the DNR normal range

Click any row to view full lake details and all survey data.

## Data Storage

All data is stored in a local SQLite database (`fishing.db`) in the app directory. Delete this file to reset.

## Excel Format Notes

The parser handles multiple lakes per sheet, separated by blank rows. Each lake block starts with a header row containing the lake name, survey year, "ID" label, and the DNR lake ID. The sample file included has sheets like "CPUE Data 1" and "CPUE Data 2" with this format.

## Mapping Species
MN DNR provides species information in survey data in codes. These are then translated via the JS when the survey data is viewed, but not available in the JSON. I created a species map JSON, but if you find new species need to be mapped, they can be added via the browser console and the following command:
```
fetch('/api/species_map', {method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify([{"abbreviation":"RBS","species":"Rainbow Smelt"}])})
```
