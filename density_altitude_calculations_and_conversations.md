# Density Altitude Calculations and Recent Conversation Notes

Date: 2026-07-05
Project: 3B3-DA
Primary implementation: density_altitude.html

## 1) Purpose

This document describes how density altitude is computed in the current tool, what data sources are used, and how missing data is handled. It also includes notes from the most recent conversation context available in this environment.

## 2) Data Sources Used by the Page

1. NWS hourly forecast
- Endpoint: https://api.weather.gov/gridpoints/BOX/48,100/forecast/hourly
- Inputs used: hourly temperature and dewpoint.

2. Open-Meteo hourly pressure
- Endpoint: https://api.open-meteo.com/v1/forecast?latitude=42.4259025&longitude=-71.7928508&hourly=surface_pressure&timezone=America%2FNew_York&forecast_days=7
- Input used: hourly surface pressure (hPa).

3. Latest KFIT observation
- Endpoint: https://api.weather.gov/stations/KFIT/observations/latest
- Inputs used: current temperature, dewpoint, and pressure fields.

## 3) Site and Elevation Constants

- KFIT station elevation: 345 ft
- Sterling field elevation used for display/performance context: 460 ft
- Elevation delta applied to current observed DA display: +115 ft (460 - 345)

## 4) Core Calculations

## 4.1 Unit conversions

- Fahrenheit to Celsius:
  $$
  C = (F - 32) \cdot \frac{5}{9}
  $$

- Celsius to Fahrenheit:
  $$
  F = C \cdot \frac{9}{5} + 32
  $$

## 4.2 Pressure altitude from pressure

Pressure altitude is computed from station pressure (hPa):

$$
PA_{ft} = 145366.45 \left(1 - \left(\frac{P_{hPa}}{1013.25}\right)^{0.190284}\right)
$$

## 4.3 Station pressure derivation when only barometric pressure is available

If station pressure is missing in current observations, sea-level-adjusted barometric pressure is converted to station pressure using the barometric formula:

$$
P_{station} = P_{sea\_level} \left(1 - \frac{Lh}{T_0}\right)^{exp}
$$

Where:
- h is elevation in meters
- T0 = 288.15 K
- L = 0.0065 K/m
- exp = 5.2558797

This conversion uses KFIT elevation for the current observation path.

## 4.4 Saturation vapor pressure from dewpoint (Tetens)

At dewpoint, air is saturated, so actual vapor pressure e is:

$$
e_{hPa} = 6.1078 \cdot 10^{\left(\frac{7.5\,T_{dew,C}}{237.3 + T_{dew,C}}\right)}
$$

If dewpoint is unavailable, the code falls back to dry-air temperature behavior in the virtual-temperature step.

## 4.5 Virtual temperature

Virtual temperature accounts for moisture effects on density:

$$
T_{v,K} = \frac{T_K}{1 - 0.378\,\frac{e}{P}}
$$

Where:
- T_K is ambient temperature in Kelvin
- e is vapor pressure from dewpoint
- P is station pressure in hPa

If dewpoint is missing, $T_{v,K} = T_K$.

## 4.6 Moist-air density and density altitude

Air density is computed from pressure and virtual temperature:

$$
\rho = \frac{P_{Pa}}{R_d\,T_v}
$$

With:
- $P_{Pa} = P_{hPa} \cdot 100$
- R_d = 287.05 J/(kg*K)

Then density altitude is solved by inverting ISA density relationship:

$$
h_m = \frac{T_0}{L} \left(1 - \left(\frac{\rho}{\rho_0}\right)^{\frac{1}{4.2559}}\right)
$$

$$
DA_{ft} = h_m \cdot 3.28084
$$

With ISA constants:
- rho0 = 1.225 kg/m^3
- T0 = 288.15 K
- L = 0.0065 K/m

## 4.7 Origins of selected constants (why these numbers appear)

This subsection explains the less-obvious numerical constants used above.

1. The exponent $5.2558797$ in the pressure formula

In the ISA troposphere, combining hydrostatic balance with a linear lapse rate gives:

$$
\frac{P}{P_0} = \left(1 - \frac{Lh}{T_0}\right)^{\frac{g_0}{R_d L}}
$$

So the exponent is:

$$
\frac{g_0}{R_d L} \approx \frac{9.80665}{287.05 \cdot 0.0065} \approx 5.25588
$$

This is the source of the $5.2558797$ used in Section 4.3.

2. The exponent $0.190284$ in pressure-altitude form

That value is just the reciprocal of the previous exponent:

$$
0.190284 \approx \frac{1}{5.25588}
$$

It appears because pressure-altitude equations are rearranged to solve for altitude directly from pressure ratio.

3. Why $\frac{1}{4.2559}$ appears in the density inversion

For ISA in the troposphere:

$$
\frac{\rho}{\rho_0} = \left(1 - \frac{Lh}{T_0}\right)^{\left(\frac{g_0}{R_d L} - 1\right)}
$$

The density exponent is therefore:

$$
\frac{g_0}{R_d L} - 1 \approx 5.25588 - 1 = 4.25588
$$

So when solving for $h$, the equation uses the reciprocal power:

$$
\left(\frac{\rho}{\rho_0}\right)^{\frac{1}{4.2559}}
$$

This is why the value $4.2559$ appears in Section 4.6.

4. Constant $145366.45$ in feet-based pressure-altitude equation

This is a unit-converted ISA scale factor:

$$
\frac{T_0}{L} \cdot 3.28084 \approx \frac{288.15}{0.0065} \cdot 3.28084 \approx 145{,}366.45\ \text{ft}
$$

It is the meter-based ISA height scale converted to feet.

5. Constant $0.378$ in virtual temperature expression

Using mixing ratio and gas-constant ratios, virtual temperature can be approximated as:

$$
T_v = \frac{T}{1 - (1-\varepsilon)\frac{e}{P}}, \quad \varepsilon = \frac{R_d}{R_v} \approx 0.622
$$

Thus:

$$
1-\varepsilon \approx 1-0.622 = 0.378
$$

So $0.378$ is not arbitrary; it comes from dry-air vs water-vapor gas constants.

## 5) Current vs Forecast Path

Current gauge path:
- Uses latest KFIT observation.
- Prefers stationPressure.
- If stationPressure is missing, uses barometricPressure converted to station pressure at KFIT elevation.
- Computes PA and DA from that observation.
- Applies elevation offset from KFIT to Sterling for display of current DA.

Forecast path:
- Uses NWS hourly temperature/dewpoint.
- Matches each hour to nearest Open-Meteo pressure sample.
- Match is accepted only if within 90 minutes.
- If no pressure match exists, pressure altitude and density altitude are shown as N/A for that hour.

## 6) Operational Threshold Coloring

DA gain above field elevation drives color coding:
- Gain < 1000 ft: cyan
- 1000 ft to < 2500 ft: amber
- >= 2500 ft: red

## 7) Error Handling and Data Availability

- Each API call uses a 12-second timeout.
- If any required source fails, the UI shows an error with a Retry button.
- Missing pressure for forecast hour results in N/A for pressure, pressure altitude, and density altitude for that hour.

## 8) Notes from Repository Memory

Previously captured project note indicates:
- NWS barometricPressure in latest observations is sea-level adjusted (altimeter-like), not station pressure.
- Therefore stationPressure should be used when present; otherwise derive station pressure from barometric pressure using station elevation before DA/PA calculations.

This behavior matches the current implementation.

## 9) Recent Conversation Notes (Available in This Session)

The local session history database currently returned no prior saved conversation rows, so only current-session context is available.

Current conversation highlights:
1. User requested a document describing the calculations and including notes from recent conversations.
2. I extracted formulas and assumptions from the current implementation.
3. I checked local session history storage, which returned no prior stored turns in this environment.
4. This document therefore includes full calculation documentation plus current-session conversation notes.

## 10) Suggested Future Improvement

If historical conversation summaries are required automatically, enable/populate the local session index and add a small export step that appends the latest saved turns into this document format.
