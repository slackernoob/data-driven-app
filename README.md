# Run Window SG

A single static page that answers one question: **when in the next 12 hours should I run?**

Scored on estimated WBGT (heat stress), with rain as a hard veto. No build step, no
dependencies, no API keys, no backend — one `index.html` served from GitHub Pages.

## The metric

```
WBGT ≈ 0.7·Twb + 0.2·Tg + 0.1·Ta
Tg   = Ta + 0.0155·solar − 0.15·wind      (transparent heuristic, not a validated model)
```

Fed by Open-Meteo's hourly `wet_bulb_temperature_2m`, `temperature_2m`,
`shortwave_radiation` and `wind_speed_10m`.

**This is an estimate, not NEA's official WBGT.** Open-Meteo gives *thermodynamic*
wet-bulb, while the WBGT formula wants *natural* wet-bulb, which runs warmer under
solar load — so this number reads low. It is labelled as an estimate in the UI.

Bands (`CONFIG.BANDS`) are tuned for heat-acclimatised tropical runners. ACSM's published
bands are far stricter and would black-flag most Singapore afternoons.

## Tunables

Everything worth arguing with is in `CONFIG` at the top of the script:

| Key | Default | Note |
|---|---|---|
| `RAIN_VETO_POP` | `50` | Blunt. On a thundery day this vetoes most of the afternoon. |
| `BANDS` | 28 / 30 / 32 °C | Verify against a source you trust. |
| `GLOBE_SOLAR_K`, `GLOBE_WIND_K` | `0.0155`, `0.15` | Globe-temp heuristic. |
| `WINDOW_TOLERANCE_C` | `0.7` | How far from the best hour still counts as the window. |
| `FALLBACK` | central SG | Used when geolocation is denied or times out. |

## API notes (verified 2026-08-26)

Findings from probing the Singapore open-data landscape, kept because they cost an
afternoon to establish:

- **NEA / data.gov.sg cannot drive this app.** `twenty-four-hr-forecast` returns three
  coarse periods plus one island-wide temperature *range* (e.g. 24–33 °C) and text
  categories. `uv` returns **past hours only**. There is no hourly numeric forecast
  anywhere in the keyless Singapore data.
- `wbgt` and `lightning` return **403** — they require an API key.
- Every other `api-open.data.gov.sg/v2/real-time/api/*` endpoint is keyless, and all send
  `access-control-allow-origin: *`, so a static page can call them directly.
- data.gov.sg **403s on non-browser User-Agents** (Python `urllib` fails, `curl` succeeds).
  Irrelevant in a browser; a trap for any build step or GitHub Action.
- data.gov.sg rate-limits hard: 12 rapid sequential requests produced `429`s.
- Live NEA station counts: air-temperature 18, relative-humidity 18, wind-speed 17,
  **rainfall 89**.
- LTA DataMall needs an `AccountKey`, but `carpark-availability`, `taxi-availability` and
  `traffic-images` are mirrored **keyless** at `https://api.data.gov.sg/v1/transport/*`.

## Not built (deliberately)

The NEA bias-correction layer — using the nearest of 18 live temperature stations to
correct Open-Meteo's ~11 km grid over a 50 km island. It's the interesting part, and it
belongs in v2, after this version has proven it gets opened.

## Run locally

```sh
python3 -m http.server 8000    # then open http://localhost:8000
```

Geolocation needs a secure context: `localhost` and `https://` both qualify.
