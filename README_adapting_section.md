## Adapting this for your own airport or club

The whole app is one file, `density_altitude.html`. Everything site-specific lives in
the `SITE` block near the top (inside `<head>`, under the "SITE CONFIGURATION" comment).
Fork the repo, edit that block, and host the file anywhere that serves static HTML
(GitHub Pages works fine). No other changes are needed.

**US sites only:** both the forecast and the observations come from `api.weather.gov`.

### What to set

| Field | What it is | Where to get it |
|---|---|---|
| `eyebrow`, `airportName`, `place`, `shortName` | Page text | Your choice |
| `lat`, `lon` | Airport location | e.g. airnav.com |
| `fieldElevFt` | Field elevation, ft MSL | Airport data / airnav.com |
| `nwsGrid` (`office`, `x`, `y`) | NWS forecast grid cell | `https://api.weather.gov/points/<lat>,<lon>` → `gridId`, `gridX`, `gridY` |
| `obsStation.id` | Nearby NWS observation station | Same `/points` response → `observationStations`, or pick a nearby ASOS/AWOS airport |
| `obsStation.elevFt` | That station's elevation, ft MSL | `https://api.weather.gov/stations/<ID>` → `elevation` (meters; × 3.28084) |
| `obsStation.name` | Fallback display name | Your choice |

### Choosing the observation station

- Open `https://api.weather.gov/stations/<ID>/observations/latest` and check that it has
  recent temperature, dewpoint and pressure values.
- The current-DA figure is computed at the station and then shifted by
  `fieldElevFt − obsStation.elevFt`. That shortcut is good when the station is
  close to the field and in similar terrain. It gets worse as distance or terrain
  differences grow.
- If the station reports `stationPressure`, it is used directly. Otherwise the page
  converts the sea-level-adjusted barometric pressure using `obsStation.elevFt`, so
  that elevation must be right.
- If the station has no usable data, the page falls back to the forecast for the
  current value.

### Notes

- Times are handled in UTC internally and displayed in each viewer's local time zone, so
  there is no time zone setting.
- The forecast pressure comes from Open-Meteo for the lat/lon you configure. It is a
  model value, not an observation.
