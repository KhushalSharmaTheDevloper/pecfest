# AIRPOCALYPSE: Chandigarh Edition

A Flask air-quality dashboard and illustrative scenario simulator for Chandigarh. The app keeps measured readings, third-party station data, model estimates, and what-if calculations distinguishable; simulated values are not monitoring observations or forecasts.

## What the app displays

- **Overview:** latest database reading, optional Open-Meteo model estimate or WAQI station median, and a historical AQI trend.
- **Air map:** available WAQI station readings and catalogued stations. CPCB and CPCC are listed as possible sources, but are not treated as connected feeds unless configured.
- **Simulator:** user-editable particulate baselines, hypothetical intervention controls, an estimated AQI range, control advice, a GRAP reference meter, and an assumption-based five-year projection.
- **Compare scenarios:** an explicitly inactive comparison page.

## AQI standards and labels

The app does not use a single AQI standard for every source:

- Open-Meteo and WAQI AQI values are presented as **US EPA AQI**.
- Database readings identified as CPCB use **Indian National AQI**.
- The simulator's particulate conversion uses the **US EPA particulate AQI breakpoints** implemented in `app.py` and `static/js/simulator.js`.
- The simulator's descriptive labels are local display bands, not an additional AQI calculation:

| Simulator label | US EPA AQI |
|---|---:|
| Balanced | 0–50 |
| Moderate | 51–150 |
| Poor | 151+ |

### Particulate concentration to US EPA AQI

For a concentration `C`, the simulator first applies pollutant-specific truncation:

- PM₂.₅: `Cₜ = floor(max(0, C) × 10) / 10` (truncate to one decimal place).
- PM₁₀: `Cₜ = floor(max(0, C))` (truncate to an integer).

It locates the breakpoint interval `[C_low, C_high]` containing `Cₜ`, then linearly interpolates:

```text
AQI = round(
  (I_high - I_low) / (C_high - C_low) × (Cₜ - C_low) + I_low
)
```

The implemented breakpoints are:

| Pollutant | Concentration interval (µg/m³) | AQI interval |
|---|---:|---:|
| PM₂.₅ | 0.0–9.0 | 0–50 |
| PM₂.₅ | 9.1–35.4 | 51–100 |
| PM₂.₅ | 35.5–55.4 | 101–150 |
| PM₂.₅ | 55.5–125.4 | 151–200 |
| PM₂.₅ | 125.5–225.4 | 201–300 |
| PM₂.₅ | 225.5–325.4 | 301–500 |
| PM₁₀ | 0–54 | 0–50 |
| PM₁₀ | 55–154 | 51–100 |
| PM₁₀ | 155–254 | 101–150 |
| PM₁₀ | 255–354 | 151–200 |
| PM₁₀ | 355–424 | 201–300 |
| PM₁₀ | 425–604 | 301–500 |

Concentrations above the last listed interval return AQI 500 in this implementation. Concentration gaps between breakpoint intervals also return 500; the current code does not interpolate or clamp those gaps to the neighboring interval.

### Combining PM₂.₅ and PM₁₀

When calculating AQI from particulate concentrations, the app takes the larger sub-index:

```text
particulate_AQI = max(PM₂.₅_AQI, PM₁₀_AQI)
```

When a provider supplies its own AQI (for example Open-Meteo), the simulator uses that provider value as the displayed baseline AQI while using PM₂.₅ and PM₁₀ to estimate scenario changes. WAQI station AQI values are provider-reported; station medians are calculated from the available station AQIs.

## Simulator formulas

All intervention percentages are represented as fractions between `0` and `1`:

```text
control_fraction = slider_percent / 100
winter_intensity = winter_slider_percent / 100
```

For each pollutant, every control has an assumed source-share range `[share_low, share_high]`. These are broad illustrative assumptions in the browser model, not Chandigarh source-apportionment measurements:

| Control | PM₂.₅ share range | PM₁₀ share range |
|---|---:|---:|
| Road motor vehicles | 8–25% | 5–20% |
| Trucks and heavy vehicles | 4–15% | 5–20% |
| Construction dust | 2–12% | 15–40% |
| Firecrackers | 1–12% | 1–8% |
| Green infrastructure | 0–3% | 0–5% |

For a selected set of controls `i`, with control fraction `cᵢ`, the browser model combines their independent residual fractions multiplicatively:

```text
remaining_low  = product over i of (1 - cᵢ × share_lowᵢ)
remaining_high = product over i of (1 - cᵢ × share_highᵢ)

reduction_low  = min(0.85, 1 - remaining_low)
reduction_high = min(0.85, 1 - remaining_high)
```

The 85% cap is a model guard: the assumptions cannot reduce particulate concentrations by more than 85%. Since the high source shares produce the stronger reduction, the estimated concentrations are:

```text
winter_lift_low  = winter_intensity × winter_share_low
winter_lift_high = winter_intensity × winter_share_high

scenario_concentration_low =
  baseline_concentration × (1 + winter_lift_low) × (1 - reduction_high)

scenario_concentration_high =
  baseline_concentration × (1 + winter_lift_high) × (1 - reduction_low)
```

Winter-smog uplift assumptions:

| Pollutant | Low uplift at 100% winter intensity | High uplift at 100% winter intensity |
|---|---:|---:|
| PM₂.₅ | 8% | 25% |
| PM₁₀ | 5% | 18% |

The winter slider is automatically initialized from the browser's current calendar month. The implemented starting intensities are:

| Month | Starting winter intensity |
|---|---:|
| January | 100% |
| February | 75% |
| March | 35% |
| October | 25% |
| November | 60% |
| December | 100% |
| April–September | 0% |

These are illustrative seasonality settings, not measured Chandigarh meteorology. The winter factor increases estimated concentration; it is not an emissions-control slider.

### AQI range for a scenario

The browser converts each pollutant's low and high scenario concentrations to particulate AQI, then combines pollutants using the larger sub-index:

```text
particulate_scenario_low  = max(PM₂.₅_AQI(low),  PM₁₀_AQI(low))
particulate_scenario_high = max(PM₂.₅_AQI(high), PM₁₀_AQI(high))
```

To keep a provider-supplied baseline AQI aligned with modeled particulate changes, the browser calculates a scale factor:

```text
particulate_baseline_AQI = max(PM₂.₅_AQI(baseline), PM₁₀_AQI(baseline))

AQI_scale =
  provider_baseline_AQI / particulate_baseline_AQI, if particulate_baseline_AQI > 0
  1, otherwise
```

For manually edited PM readings, the baseline AQI is calculated from the particulate readings, so the scale factor is normally `1`.

The final scenario range is:

```text
scenario_AQI_low  = min(round(particulate_scenario_low  × AQI_scale),
                        round(particulate_scenario_high × AQI_scale))

scenario_AQI_high = min(500, max(round(particulate_scenario_low  × AQI_scale),
                                 round(particulate_scenario_high × AQI_scale)))
```

The high endpoint is capped at 500. The current browser implementation does not separately cap the low endpoint after applying `AQI_scale`.

The displayed particulate impact is calculated from the low/high estimated concentrations relative to their input baselines:

```text
reduction_at_low_estimate  = 1 - scenario_concentration_high / baseline_concentration
reduction_at_high_estimate = 1 - scenario_concentration_low  / baseline_concentration
```

The UI reports the minimum and maximum across PM₂.₅ and PM₁₀ as a percentage. With winter uplift, this value can be negative, meaning the combined assumptions estimate a net particulate increase rather than a reduction.

### Control advice

The advice targets the conservative scenario endpoint `AQI ≤ 50` (the simulator's Balanced band):

```text
AQI points needed = max(0, scenario_AQI_high - 50)
```

If all five emission-control sliders (traffic, heavy vehicles, dust, crackers, and greenery) at 100% cannot bring the conservative model estimate to 50 or below, the UI says the selected model controls may not be enough. Winter intensity remains fixed during this check.

If the target appears reachable, the browser uses 20 iterations of binary search to find the smallest shared interpolation factor `t` between current and maximum control settings:

```text
candidate_controlᵢ = current_controlᵢ + t × (100 - current_controlᵢ)
```

The candidate percentages are rounded up and displayed as approximate suggestions. This is a joint model-based estimate, not a guarantee or a policy recommendation.

### No-new-controls projection

The five-year outlook does not extrapolate a measured trend. It applies the user-entered annual growth assumption `g` (default 5%, editable from 0% to 25%) to the current baseline AQI:

```text
projected_AQI(year n) = round(baseline_AQI × (1 + g / 100) ^ n)
```

No interventions are applied to this projection. Its GRAP label is a reference only:

| GRAP reference label | AQI range used by the simulator |
|---|---:|
| Below Level 1 | ≤200 |
| Level 1 | 201–300 |
| Level 2 | 301–400 |
| Level 3 | 401–450 |
| Level 4 | >450 |

The GRAP meter and thresholds are displayed as a hypothetical comparison and are **not represented as active in Chandigarh**.

## Other aggregation and display calculations

### Dashboard station median

For WAQI, the dashboard filters out missing/non-numeric values, sorts the available readings, and calculates the median separately for AQI, PM₂.₅, and PM₁₀:

```text
odd number of readings:  middle sorted value
even number of readings: mean of the two middle sorted values
```

The pollutant medians are independent summaries; they need not come from the same station as the median AQI.

### Historical trend

- Measured database readings are grouped by India-local hour for the 24-hour view or India-local date for longer views.
- Each non-empty measured bucket is the median AQI of its readings, rounded to one decimal place.
- Open-Meteo history supplies hourly `us_aqi`. The 24-hour view uses the latest 24 hourly points returned. For 7-, 30-, and 90-day views, hourly values are grouped by date and reduced to a daily median rounded to one decimal place.
- Empty buckets remain `null`; the chart does not fill missing readings with zero.

### Graphic scales

The simulator's AQI meter uses `min(100, AQI / 500 × 100)%` for marker position. Its illustrative haze opacity is `0.12 + min(AQI, 500) / 500 × 0.72`; traffic opacity is `max(0.12, 1 - average(traffic_control, heavy_control) / 100 × 0.88)`; and greenery opacity is `0.2 + green_control / 100 × 0.8`. These are visual encodings, not additional pollution calculations.

The map uses threshold-based colors for US EPA AQI and Indian National AQI; marker colors do not recalculate a reading.

The map color bands are:

| AQI standard | AQI thresholds and color |
|---|---|
| US EPA | `≤50` green; `51–100` yellow; `101–150` orange; `151–200` red; `201–300` purple; `>300` maroon |
| Indian National AQI | `≤50` green; `51–100` light green; `101–200` yellow; `201–300` orange; `301–400` red; `401–500` purple; `>500` maroon |

Missing AQI values use a neutral gray marker. The five-year projection bars use `min(100, projected_AQI / 500 × 100)%` for their fill width.

### Rounding implementation note

The browser uses JavaScript `Math.round`; the Flask API uses Python `round`. Their handling of exact half-way values can differ, so a rounded AQI can differ by one point at a tie even when the interpolation inputs are otherwise the same.

## Important distinction: browser model and legacy API

The interactive simulator runs in `static/js/simulator.js` and includes traffic, heavy vehicles, construction dust, crackers, greenery, and winter uplift.

The Flask `POST /api/simulate` endpoint in `app.py` is an older, separate implementation. It currently accepts only `pm25`, `pm10`, `traffic`, `dust`, and `green`; it does not include trucks, crackers, or winter smog. Its source-share ranges are:

| Control | PM₂.₅ share range | PM₁₀ share range |
|---|---:|---:|
| Traffic | 8–25% | 5–20% |
| Construction dust | 2–12% | 15–40% |
| Greenery | 0–3% | 0–5% |

The endpoint uses the same multiplicative residual formula and 85% reduction cap, but it does not apply the browser's seasonal uplift or provider-AQI scaling. Treat API results as this legacy model's output; they are not guaranteed to match the browser simulator.

## Limitations

- Source-share and winter-uplift ranges are assumptions, not locally calibrated source-apportionment or meteorological data.
- The model does not resolve weather, wind, atmospheric chemistry, regional transport, changing emissions, population exposure, or the time needed for interventions to affect ambient AQI.
- The no-action projection is an editable compounded-growth scenario, not a forecast and not an observed multi-year trend.
- GRAP thresholds are shown for comparison; their display does not mean GRAP is in force in Chandigarh.
- AQI estimates derived from PM₂.₅ and PM₁₀ cover those pollutants only; they do not represent all pollutants or a full regulatory AQI calculation.
- Modeled/provider readings and measured station readings have different provenance and AQI standards. Use the source labels shown in the app.

## Running locally

The app is a Python 3.13+ Flask application:

```bash
python3 -m pip install -r requirements.txt
python3 app.py
```

It serves at `http://localhost:8000` by default. Set `PORT` to use another port.

Optional WAQI station data requires a `WAQI_API_TOKEN` environment variable. `.env.example` documents the local environment variable; do not commit credentials.

Run the application tests:

```bash
python3 -m unittest discover -s tests -v
```
