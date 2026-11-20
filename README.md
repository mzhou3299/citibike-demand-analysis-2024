# Citi Bike 2024: Demand Patterns and a Wait-or-Ride Decision

This project explores demand across 44.2 million NYC Citi Bike trips in 2024, then tests a simple model for deciding whether to wait at a full station or ride to another one.

- Part 1, demand patterns: [`Demand_analysis.ipynb`](Demand_analysis.ipynb)
- Part 2, the wait-or-ride decision: [`Decision_model.ipynb`](Decision_model.ipynb)

## Part 1: Demand patterns

### Method
Arrivals and departures were counted at each station in 30-minute windows and joined with hourly Central Park weather (converted from UTC to New York time) and US holidays. Stations were matched to NYC neighborhoods.

### Findings
- Monthly rides rose from about 2 million in January and February to a plateau from June through October, peaking at 5.1 million in October.
- 49 of the 50 busiest stations are in Manhattan. Traffic at individual stations varies a lot: at the busiest station, the standard deviation across half-hours is about as large as the mean.
- Rides increase with temperature. Comparisons involving heavier rain in the heatmap are based on small samples. In the Part 2 regression, which controls for station, time of day and season, any rain is associated with at least 22% fewer departures, and temperatures below 35°F with about 45% fewer than 65–75°F.
- Arrivals near Madison Square Garden before the Nov 8 Knicks game were within the range of other Friday evenings.

![Monthly rides](figures/monthly_trend.png)

![Station demand](figures/station_demand.png)

![Rides by temperature and precipitation](figures/temp_precip_heatmap.png)

The station map counts rides starting or ending at each station; the citywide charts count each ride once. An interactive map is in [`station_demand_map.html`](station_demand_map.html) (open it in a browser).

## Part 2: Wait or ride to another station?

When a rider arrives to return a bike and every dock is taken, they can wait for a dock to open or ride to a nearby station and walk back.

### Method
- Each station's departure rate was estimated from trips on days 1–21 of each month: its usual pattern for that half-hour and day type, times a citywide weather and season multiplier from a Poisson regression. Each departure frees a dock, so the rate gives the probability that a dock opens within a given time.
- These probabilities were checked on days 22–end against 2-minute station status snapshots taken while stations were full. A three-parameter correction was fitted on days 15–21.
- 47,894 full-station moments from days 22–end were replayed. At each one, a rule chose between waiting and riding to one of the 5 nearest stations within 500 m, and the choice was scored using what actually happened.
- With live dock counts, a station showing an open dock can still fill before the rider arrives. A fill-aware version of the model estimates that chance from days 15–21, based on how many docks are open and how busy the station usually is at that time.

### Results
Total minutes per rider: ride to the chosen station, wait there (capped at 60 minutes), and walk back. The last column is the share of riders still waiting after 60 minutes at the chosen station.

| Rule | Live data | Mean | 90th percentile | Still waiting after 60 min |
|---|---|---|---|---|
| Always wait | no | 23.2 | 60.0 | 19.0% |
| Always ride to the nearest station (if one is within 500 m) | no | 12.9 | 35.9 | 6.4% |
| Model, trip-based | no | 11.4 | 24.0 | 4.0% |
| Model, recalibrated | no | 10.8 | 20.2 | 3.5% |
| Nearest station showing open docks | yes | 7.4 | 9.7 | 1.0% |
| Nearest station showing 2+ open docks | yes | 7.2 | 9.8 | 0.9% |
| Model, trip-based + live | yes | 7.9 | 11.8 | 1.8% |
| Model, recalibrated + live | yes | 7.3 | 9.8 | 1.0% |
| Model, recalibrated + live, fill-aware | yes | 7.1 | 9.6 | 0.9% |
| Best choice in hindsight | | 5.4 | 6.9 | 0.3% |

![Decision rules compared](figures/decision_model/decision_rules_comparison.png)

Without live dock counts, the recalibrated model averaged 10.8 minutes, compared with 12.9 for always riding to the nearest station (2.0 minutes faster; 95% interval 1.7 to 2.4, from resampling test days).

With live dock counts, the recalibrated model and the nearest-open-dock rule had similar average times: 7.3 and 7.4 minutes. Both treat a station showing an open dock as a sure thing, but a quarter of riders were sent to a station showing just one, and 22% of those found it full on arrival. The fill-aware model averaged 7.1 minutes, 0.31 minutes faster than the nearest-open-dock rule (95% interval 0.21 to 0.41). Simply preferring stations with 2+ open docks got about half of that improvement. The trip-based model without the correction overestimated how quickly docks open at full stations and averaged 7.9 minutes with live data.

### Limitations
- Only moments when the full station and its candidate neighbors were all in service and reporting for the following hour were replayed (47,894 of 60,000 sampled).
- Waits are capped at 60 minutes and measured in 2-minute steps. The replay assumes riders don't affect each other, estimates travel time from straight-line distance, and uses observed weather in place of a forecast.
- Intervals come from resampling the 113 test days. They don't include uncertainty in the fitted models, and all results come from one year of data.

## How to run
1. Create an environment (Python 3.10+, tested on 3.13):
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```
2. Download the 2024 monthly trip files from [Citi Bike System Data](https://citibikenyc.com/system-data) and unzip them into `data/trips/`.
3. Run the notebooks top to bottom. `Decision_model.ipynb` downloads the station status file (4.2 GB, public) into `data/status/` on its first run. Both notebooks load all 2024 trips into memory; they were run on a laptop with 16 GB of RAM.

## Data
- Trips: [Citi Bike System Data](https://citibikenyc.com/system-data), used under the Citi Bike data license agreement.
- Station status snapshots: [rosenthal/citi-bike-stations](https://www.kaggle.com/datasets/rosenthal/citi-bike-stations) on Kaggle (CC BY 4.0).
- Weather: [Meteostat](https://meteostat.net) (CC BY-NC 4.0).
- Neighborhood boundaries: NYC Neighborhood Tabulation Areas (`Neighborhood_NTA.geojson`).
