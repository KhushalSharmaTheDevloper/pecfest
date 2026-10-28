# AIRPOCALYPSE: Chandigarh Edition

A Flask and SQLite application for exploring Chandigarh air-quality observations. The app distinguishes source-reported readings, locally stored measured records, and illustrative scenario estimates.

## What the app does

| Page | Current behavior |
|---|---|
| **Overview** (`/`) | Shows the latest locally stored measured record, if any; a measured-only AQI trend; and an optional live WAQI view. WAQI overview AQI is calculated from PM₂.₅/PM₁₀, not copied from the provider's overall AQI. |
| **Air map** (`/map`) | Plots station markers and source-reported values on Leaflet/OpenStreetMap. Shows source and AQI standard; does not create a heatmap or interpolate citywide readings. WAQI station details can include PM₂.₅, PM₁₀, O₃, NO₂, SO₂, and CO. |
| **Simulator** (`/simulator`) | Loads median current PM₂.₅ and PM₁₀ from Chandigarh WAQI stations when available, or lets the user enter a baseline. Sliders update hypothetical particulate/AQI ranges, an illustrative recommendation, and a five-year compounded scenario. |
| **Compare scenarios** (`/scenarios`) | Prepared interface only; comparison results are not implemented. |
| **Data sources** (`/sources`) | Lists WAQI and source candidates. CPCB and CPCC are not connected. |

The map has a CPCB selector, but there is no CPCB station API implementation in the current Flask app. WAQI uses the **US EPA AQI** standard; CPCB's Indian National AQI is not interchangeable. There is no Meteo or weather integration: winter influence is a user-adjustable calendar-month assumption, not a weather observation or forecast. No demonstration readings or stations are inserted into the database.

## AQI calculation

For each pollutant, the app first truncates PM₂.₅ concentration to one decimal place and PM₁₀ to an integer, then linearly interpolates within the matching US EPA breakpoint band:

`AQI = round((Ihi - Ilo) / (Chi - Clo) × (C - Clo) + Ilo)`

Here `C` is the rounded concentration, `Clo`/`Chi` are the concentration-band limits, and `Ilo`/`Ihi` are that band's AQI limits. For a combined particulate AQI, it takes the **maximum** of available PM₂.₅ and PM₁₀ AQIs. Values above the highest listed concentration band return AQI 500.

| Pollutant | Concentration (µg/m³) | AQI |
|---|---:|---:|
| PM₂.₅ | 0.0–9.0 | 0–50 |
|  | 9.1–35.4 | 51–100 |
|  | 35.5–55.4 | 101–150 |
|  | 55.5–125.4 | 151–200 |
|  | 125.5–225.4 | 201–300 |
|  | 225.5–325.4 | 301–500 |
| PM₁₀ | 0–54 | 0–50 |
|  | 55–154 | 51–100 |
|  | 155–254 | 101–150 |
|  | 255–354 | 151–200 |
|  | 355–424 | 201–300 |
|  | 425–604 | 301–500 |

When the overview uses WAQI, it calculates each station's particulate AQI from its PM readings and displays the median across stations with a calculable AQI. PM₂.₅ and PM₁₀ cards separately show their respective station medians. A missing pollutant is excluded from that pollutant's median; if no numeric readings are available, it remains unavailable.

## Trend chart

The Chart.js line graph uses **only locally stored rows marked `measured` with a non-null AQI**; live WAQI values are not written into the history. It groups readings by hour for 24 hours or by day for 7, 30, and 90 days, and plots each bucket's median. Times and labels use India Standard Time (IST). Empty buckets remain gaps, not zeros. The latest stored reading and station summaries also come from measured database records.

```text
Stored measured AQI readings ──> hourly/daily buckets ──> median per bucket ──> trend chart
WAQI station PM readings ──> US EPA PM AQIs ──> live overview / simulator baseline
                                                     └──> illustrative scenario range
```

## Simulator: formulas and assumptions

The browser simulator uses these fixed, illustrative source-share ranges. A control's coverage is its slider percentage divided by 100. The share values are assumptions, not measured Chandigarh source apportionment.

| Control | PM₂.₅ assumed share | PM₁₀ assumed share |
|---|---:|---:|
| Road motor vehicles | 8–25% | 5–20% |
| Heavy vehicles | 4–15% | 5–20% |
| Construction dust | 2–12% | 15–40% |
| Firecrackers | 1–12% | 1–8% |
| Green infrastructure | 0–3% | 0–5% |

For pollutant `p`, control coverage `uⱼ`, and assumed share `sⱼ`:

```text
Rlow  = min(85%, 1 - ∏ⱼ(1 - uⱼ × slow,ⱼ))
Rhigh = min(85%, 1 - ∏ⱼ(1 - uⱼ × shigh,ⱼ))
Winter uplift = winter slider / 100 × assumed winter share
Scenario concentration range =
  [baseline × (1 + winter × winter-low) × (1 - Rhigh),
   baseline × (1 + winter × winter-high) × (1 - Rlow)]
```

Winter shares are 8–25% for PM₂.₅ and 5–18% for PM₁₀. The seasonal slider's starting value by browser calendar month is: January 100%, February 75%, March 35%, October 25%, November 60%, December 100%, and 0% in other months. The user can adjust it.

The simulator converts both ends of each particulate range to AQI using the breakpoint method above, takes the worse pollutant AQI at each end, and shows the rounded midpoint as the headline scenario AQI. Its display labels are **Balanced** (≤50), **Moderate** (51–150), and **Poor** (>150); these are simulator labels, not a full official health-category scale. Scenario results are not forecasts: the model is not locally calibrated and does not model weather, atmospheric chemistry, imported pollution, actual seasonal inversions, or local emission inventories. It assumes interventions reduce the selected source contributions, and caps modeled reduction at 85%; AQI is not forced to zero.

The recommendation uses the conservative (higher) scenario AQI and targets ≤50. If maximum controls cannot reach the target, it says so; otherwise it searches for a common proportional increase to the five emission-control sliders (20 binary-search iterations). This is only a model-based suggestion, not a guarantee.

The optional five-year outlook compounds the current **scenario** AQI using a user-entered annual increase (0–25%, default 5%):

`AQIyear n = round(scenario AQI × (1 + annual increase / 100)ⁿ)`, for years 1–5.

It holds current control settings constant and is explicitly illustrative, not a forecast. Its display labels are Good (≤50), Moderate (51–100), Moderate-Poor (101–150), Unhealthy (151–200), Very Unhealthy (201–300), and Hazardous (>300). A GRAP meter is also shown as a hypothetical reference only (Level 1: 201–300; Level 2: 301–400; Level 3: 401–450; Level 4: >450); it does not indicate that GRAP measures are active in Chandigarh.

The simulator presets are starting slider values, not measured or recommended policies:

| Preset | Traffic | Heavy | Dust | Crackers | Green | Winter |
|---|---:|---:|---:|---:|---:|---:|
| Cleaner commute | 75% | 80% | 25% | 0% | 35% | Current month's default |
| Dust-free build | 10% | 25% | 100% | 0% | 20% | Current month's default |
| Winter haze | 0% | 0% | 0% | 0% | 0% | 100% |
| Reset | 0% | 0% | 0% | 0% | 0% | Current month's default |

### Separate server simulation endpoint

`POST /api/simulate` is a server-side screening calculation with only traffic, dust, and greenery controls. It uses PM₂.₅ shares of 8–25%, 2–12%, and 0–3%, and PM₁₀ shares of 5–20%, 15–40%, and 0–5%, respectively. It applies the same multiplicative reduction and 85% cap, but **does not** include heavy vehicles, firecrackers, or winter uplift. The current browser simulator calculates its fuller model locally and does not call this endpoint.

## Data, endpoints, and boundaries

- SQLite is created at `data/airpocalypse.sqlite3` on startup from `data/schema.sql`. Tables store data-source records, monitoring stations, and readings classified as `measured`, `estimated`, or `simulation`.
- `GET /map-data` returns locally stored station summaries; `GET /api/map/waqi-stations` fetches WAQI stations for validated Chandigarh-area bounds; `GET /api/map/station/<id>` fetches WAQI station pollutant detail.
- `GET /healthz` returns application health. `POST /api/simulate` accepts `pm25`, `pm10`, `traffic`, `dust`, and `green`; concentrations must be 0–1000 µg/m³ and controls 0–100%.
- Live map and simulator WAQI polling is about every 5 seconds while the page is visible. The server proxy requires a WAQI token; browser fallback is also implemented for environments where the server cannot reach WAQI.
- Missing data is displayed as unavailable—not zero, safe air, or proof that a location has no pollution. The map centre is geographic context, not a station.

## Run locally

Requirements: Python 3.13+ and Flask 3.1+. Internet access is needed for live WAQI and CDN-hosted Leaflet, Chart.js, and fonts.

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python app.py
```

Open [http://localhost:8000](http://localhost:8000). Set `PORT` to change the default port. To enable WAQI, obtain a token through [WAQI API registration](https://aqicn.org/api/) and put it in a root `.env` file:

```dotenv
WAQI_API_TOKEN=your-token
```

The app loads `.env` for local development; existing environment variables take precedence. Without a token, the app still runs but live WAQI data is unavailable. CPCB/CPCC are candidates only; no feed is configured.

**Token note:** The current direct-browser WAQI fallback receives the configured token in rendered pages. Treat that token as visible to visitors; it is not server-only when this fallback is enabled.

Run tests with:

```bash
python -m unittest discover -s tests
```

## Project layout

```text
app.py                 Flask routes, SQLite queries, WAQI proxy, AQI and server simulation
data/schema.sql        SQLite tables and indexes
templates/             Jinja pages
static/css/style.css   Application styling
static/js/              Dashboard, map, and client-side simulator
tests/test_app.py       Automated application tests
```
