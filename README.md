# heart_repro_paper324

This repository is for estimating and comparing regression linear models, with the e-scooter speed as the dependent variable.

## Summary

| File / Folder | Description |
|---|---|
| `escooter_linear_speed_R_for_hEART.ipynb` | R notebook: loading, reduction, models, diagnostics and LRT |
| `functions_linear.R` | Utility functions called by the notebook (`source("functions_linear.R")`) |
| `data/` | Datasets |
| `model_results_linear/` | Model outputs (parameters, statistics, plots) |

## Data

Two CSV files are read by the notebook, joined on `source` + `frame`:

- `data/clean_dataset.csv.gz` — one row per video frame (30 fps) of an e-scooter trip.
- `data/clean_dataset_vru_detections.csv.gz` — one row per VRU (vulnerable road user) detected in a frame.

Both files are anonymised: the original clip/trip filename has been replaced by an
integer trip identifier in the `source` column. Only the columns actually used by the
notebook are kept in `clean_dataset.csv`.

Both files are stored **gzip-compressed** (148 MB → 5.2 MB and 31 MB → 8.1 MB) to stay
under GitHub's 100 MB file limit. No extra step is needed: `read.csv()` decompresses
`.csv.gz` transparently. To get plain CSVs back, run `gunzip -k data/*.csv.gz`.

### `clean_dataset.csv.gz` — one row per frame

**Identifiers and time**

| Column | Type | Description |
|---|---|---|
| `source` | integer | Anonymised trip identifier (1…N). One value per video clip / trip. Used as the panel/trip grouping variable. |
| `frame` | integer | Frame index within the trip (30 fps). The notebook derives `second = frame %/% 30` and aggregates to 1 Hz. |
| `rider_id` | string | Anonymised rider identifier. Upper level of the nested random-effect structure (`rider_id/source`). |

**Speed (dependent variables)**

| Column | Type | Description |
|---|---|---|
| `speed_kmh_kalman` | numeric | Kalman-filtered e-scooter speed at the current frame (km/h). Explanatory variable (current speed). Frames with speed ≤ 1 km/h are dropped. |
| `speed_kmh_kalman_t1` | numeric | Kalman-filtered speed at the next time step (km/h). **Dependent variable** of the models. Set to `NA` on the last second of each trip. |

**Riding dynamics and geometry**

| Column | Type | Description |
|---|---|---|
| `delta_yaw_deg` | numeric | Change in heading (yaw) over the frame, in degrees. Proxy for steering / curvature. |
| `road_width_perp_m` | numeric | Width of the road/path perpendicular to the direction of travel (m). |
| `slope_signed_pct` | numeric | Signed longitudinal slope (%); positive uphill, negative downhill. |
| `at_intersection` | 0/1 | Whether the current position is inside an intersection. |
| `turn_label` | string | Position relative to a manoeuvre: `none`, `before_left_turn`, `before_right_turn`, `after_left_turn`, `after_right_turn`. Aggregated into the binary `is_before_turn` / `is_after_turn` indicators. |
| `environment_type` | string | Type of surrounding environment (e.g. `street`, `park`, `square`). Used to build the `is_park` / `is_square` dummies. |

**VRU counts in the frame**

| Column | Type | Description |
|---|---|---|
| `n_pedestrians` | integer | Number of pedestrians visible in the frame. |
| `n_cyclists` | integer | Number of cyclists visible in the frame. |
| `n_elderly` | integer | Number of VRUs annotated as elderly. |
| `n_children` | integer | Number of VRUs annotated as children. |
| `n_running` | integer | Number of VRUs whose gait is annotated as running. |
| `n_groups` | integer | Number of VRUs belonging to a group (rather than travelling alone). |
| `n_crossing` | integer | Number of VRUs crossing the rider's path. |
| `n_pedestrians_crossing` | integer | Pedestrians crossing the rider's path. |
| `n_pedestrians_opposite` | integer | Pedestrians moving in the opposite direction. |
| `n_pedestrians_same_direction` | integer | Pedestrians moving in the same direction as the rider. |
| `n_pedestrians_stationary` | integer | Stationary pedestrians. |
| `n_cyclists_crossing` | integer | Cyclists crossing the rider's path. |
| `start_crossing` | 0/1 | Flag marking the frame where a VRU starts crossing. |

**VRU composition and interaction shares** (proportions in `[0, 1]`, computed over the VRUs present in the frame)

| Column | Type | Description |
|---|---|---|
| `prop_vru_pedestrian` | numeric | Share of visible VRUs that are pedestrians. |
| `prop_vru_cyclist` | numeric | Share of visible VRUs that are cyclists. |
| `prop_interaction_same_direction` | numeric | Share of VRUs moving in the same direction as the rider. |
| `prop_interaction_opposite_direction` | numeric | Share of VRUs moving in the opposite direction. |
| `prop_interaction_crossing` | numeric | Share of VRUs crossing the rider's path. |
| `prop_interaction_stationary` | numeric | Share of VRUs that are stationary. |

**Context: date, time and weather**

| Column | Type | Description |
|---|---|---|
| `hour` | integer | Hour of the day (0–23). |
| `day_of_week` | integer | Day of week index. |
| `day_name` | string | Day name (`Monday`…`Sunday`). |
| `is_weekend` | boolean | Whether the trip took place on a Saturday or Sunday. |
| `time_of_day` | string | `Morning`, `Afternoon`, `Evening`, `Night`. The notebook merges `Evening` into `Night`. |
| `month` | integer | Month of the year (1–12). |
| `season` | string | Season (`Winter`, `Spring`, `Summer`, `Autumn`). |
| `WEATHER_LABEL` | string | Annotated weather condition (e.g. `No adverse`, rain…). |
| `LIGHTING_LABEL` | string | Annotated lighting condition (e.g. `Daylight`, `Night`). |
| `SURFACE_CONDITION_LABEL` | string | Annotated road-surface condition (e.g. `Dry`, `Wet`). |

**Rider characteristics** (constant within a rider)

| Column | Type | Description |
|---|---|---|
| `genre` | string | Rider gender as declared in the survey (`female`, `male`, …). |
| `age` | integer | Rider age in years. |
| `experience` | string | Self-reported e-scooter experience class (e.g. `1-2` years). |
| `distance_km` | numeric | Distance typically ridden by the rider (km), from the survey. |

### `clean_dataset_vru_detections.csv.gz` — one row per detected VRU per frame

| Column | Type | Description |
|---|---|---|
| `source`, `frame` | — | Join keys with `clean_dataset.csv.gz` (anonymised trip id, frame index). |
| `track_id` | integer | Tracking identifier of the VRU within the clip. |
| `x1`, `y1`, `x2`, `y2` | numeric | Bounding-box corners in image pixels. |
| `foot_x`, `foot_y` | numeric | Estimated foot-contact point in image pixels (used for the distance projection). |
| `bbox_height` | numeric | Bounding-box height in pixels. |
| `distance_m` | numeric | Estimated longitudinal distance between the rider and the VRU (m). |
| `lateral_m` | numeric | Estimated lateral offset of the VRU (m); negative = left, positive = right. |
| `cx_px` | numeric | Horizontal centre of the bounding box in pixels. |
| `azimuth_deg` | numeric | Bearing of the VRU relative to the rider's heading (degrees). |
| `VRU_TYPE_LABEL` | string | VRU type (`Pedestrian`, `Cyclist`, …). Only `Pedestrian` rows are used. |
| `INTERACTION_LABEL` | string | Interaction type (`Same-direction`, `Opposite-direction`, `Crossing`, `Stationary`, …). |
| `VRU_AGE_GROUP_LABEL` | string | Annotated age group (`Child`, `Adult`, `Elderly`, `Unknown`). |
| `VRU_GAIT_LABEL` | string | Annotated gait (`Standing`, `Walking`, `Running`, `Unknown`). |
| `VRU_GROUP_SIZE_LABEL` | string | Annotated group size (`Solo`, `Pair`, `Group 3+`, `Unknown`). |

### Variables derived in the notebook

These are not stored in the CSV files; the notebook computes them from the columns above.

| Variable | Description |
|---|---|
| `second` | Second index within the trip (`frame %/% 30`); the analysis unit after 1 Hz aggregation. |
| `speed_kmh_kalman_t0` | Speed at the first second of the trip. |
| `sec_from_start`, `sec_from_end`, `is_first_5s`, `is_last_5s` | Position within the trip and edge indicators (first / last 5 seconds). |
| `ped_min_distance_m`, `ped_mean_distance_m`, `ped_max_distance_m` | Min / mean / max pedestrian distance in the second, from the VRU file. |
| `ped_closest_lateral_m`, `ped_closest_longitudinal_m` | Lateral and longitudinal position of the closest pedestrian. |
| `n_ped_under_5m`, `…_10m`, `…_15m`, `…_20m` | Number of pedestrians within the given distance threshold. |
| `n_ped_lat_ltm3m`, `n_ped_lat_m3_m1m`, `n_ped_lat_m1_1m`, `n_ped_lat_1_3m`, `n_ped_lat_gt3m` | Pedestrian counts by lateral band (< −3 m, −3…−1 m, −1…1 m, 1…3 m, > 3 m). |
| `n_ped_detected` | Number of pedestrians detected in the second. |
| `has_ped` | 1 if at least one pedestrian was detected (distances are mean-imputed otherwise). |
| `is_before_turn`, `is_after_turn` | Binary recoding of `turn_label`. |
| `is_afternoon`, `is_park`, `is_square` | Dummies from `time_of_day` and `environment_type`. |
| `z_*` | Standardised (z-scored) versions of the continuous predictors. |

## Environment

IRkernel under Jupyter.

Required packages:

```r
install.packages(c("dplyr", "tidyr", "ggplot2", "patchwork",
                   "lme4", "lmerTest", "nlme", "MuMIn", "HLMdiag"))
```
