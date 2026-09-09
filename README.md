# CSUF Pitcher Reports

Postgame pitcher reports for Cal State Fullerton Baseball, used during the 2025 Fall and 2026 Spring seasons. The tool is a Shiny app that renders a one-page report for any CSUF pitcher and game date, with a button that exports the whole staff's outings to PDF. The report leans on pitch trajectory data, not just movement numbers, so pitchers and coaches can see how their pitches tunnel and separate from a hitter's point of view.

Part of a set of CSUF report tools:
[Hitter Reports](https://github.com/dcawthon2242/CSUFHitterReports) ·
[Catcher Reports](https://github.com/dcawthon2242/CSUFCatcherReports) ·
[Pitch Trajectory Plots](https://github.com/dcawthon2242/CSUF-Pitch-Trajectory-Plots)

<img width="1107" height="856" alt="Pitcher report example page" src="https://github.com/user-attachments/assets/dd16ca03-72ef-40c0-bef8-d2d539711018" />

## Report layout

Six visuals on top, two tables underneath.

### Plots

**Strike Zone vs. RHH and vs. LHH.** Every pitch of the outing plotted by location and colored by pitch type, split by batter side. Three regions are overlaid:

| Region | Definition |
| --- | --- |
| Meatball | Within 5 in horizontally and 8 in vertically of the center of the zone, roughly half the zone's area |
| Edge-Expand | Not a meatball, but with at least a 5% modeled chance of a swing or a called strike |
| Waste | Under a 5% modeled chance of both a swing and a called strike |

The swing and called-strike probabilities come from the `xSwing` and `xCalledStrike` models described below.

**Pitch Movement.** Induced vertical break against horizontal break for every pitch, one color per pitch type, with a large marker at each type's average and an 80% confidence ellipse around any type thrown at least three times.

**Pitch Trajectories vs. RHH and vs. LHH.** The average trajectory of each pitch type drawn in three dimensions from behind the umpire, using the TrackMan 9-parameter fit (`x0, z0, vx0, vz0, ax0, az0` and release time). Seeing all pitch types on one flight path is the clearest way to show where they separate.

**Arm Angle.** The pitcher's release point relative to a body silhouette, with the arm angle computed from the pitcher's listed height. The silhouette is chosen from `Image Files/` based on handedness and arm slot.

### Top table: outing metrics with percentiles

Each value is shown alongside its percentile against a Division I pitcher-season reference.

| Metric | Definition |
| --- | --- |
| Pitches | Total pitches in the outing |
| Strike% | Share of pitches located in the zone |
| Swing% | Share of pitches swung at |
| Whiff% | Share of swings that missed |
| Chase% | Share of pitches out of the zone that were swung at |
| FPS% | Share of first pitches of a plate appearance that were strikes, including fouls and balls in play |
| Put Away% | Share of plate appearances that reached two strikes and ended on a called or swinging strike |
| Meatball% | Share of pitches in the Meatball region |
| Edge-Expand% | Share of pitches in the Edge-Expand region |
| Waste% | Share of pitches in the Waste region |

### Bottom table: per pitch type

| Column | Definition |
| --- | --- |
| Pitch | Pitch type (abbreviated) |
| Usage | Share of the outing's pitches |
| Velo / Max Velo | Average and maximum release speed |
| IVB / HB | Average induced vertical break and horizontal break |
| Spin Dir | Average spin direction as a clock face, rounded to the nearest 15 minutes |
| Rel H | Average release height |
| Arm Ang | Average arm angle in degrees |
| Top VAA / Bot VAA | Steepest and flattest vertical approach angle of the outing |
| Extension | Average release extension toward the plate |
| Strike% | Share located in the zone |
| Whiff% | Share of swings that missed |
| CSW% | Called strikes plus whiffs, as a share of pitches |
| Chase% | Share of out-of-zone pitches swung at |
| Stuff+ | Modeled pitch quality from movement, velocity, and release only. 100 is Division I average and each 10 points is one standard deviation |

## Models

**xSwing** (LightGBM) and **xCalledStrike** (XGBoost). Predict the probability a pitch is swung at and, if taken, the probability it is called a strike. Together they define the Edge-Expand and Waste regions. Feature lists and the class map for the called-strike model are stored as `.rds` files in `Models & Reference Files/`.

**Stuff+** (LightGBM, one model per pitch type). Each model regresses the change in run expectancy on a pitch's physical characteristics:

`RelSpeed`, `SpinRate`, `SpinAxis`, `x0`, `z0`, `ax0`, `az0`, `Extension`, plus the pitch's velocity and acceleration differences from the pitcher's own fastball average (`mph_diff`, `ax_diff`, `az_diff`).

Hyperparameters were tuned with Bayesian optimization over 5-fold cross-validation. Training is in `Pitch Grade Models/D1PitchGrade.R`. At report time the raw prediction is standardized against the Division I distribution for that pitch type (`stuff_plus_reference.rds`) so that `Stuff+ = 100 - 10 * z`, where a lower predicted run value is better for the pitcher.

**Arm angle.** Pitcher height comes from a `D1PitcherHeights` table (`Pitcher`, `PitcherTeam`, `Height` in inches, `PitcherThrows`). The shoulder is placed at 70% of standing height and the arm angle is the angle between vertical and the line from the shoulder to the release point. Slots are bucketed as Over The Top, High Three-Quarters, Three-Quarters, Low Three-Quarters, Slinger, Sidearm, and Submarine.

**LIVAA / LIHAA.** Location-independent approach angles: the residual between a pitch's actual approach angle and the value a linear model predicts from plate location and pitch type. These are fit on the loaded data and available as features.

## Repository layout

```
PitchReports.R                                   Shiny app, models, plots, tables, and PDF export
Pitch Grade Models/
  D1PitchGrade.R                                 Stuff+ training script (Division I TrackMan)
  <PitchType>_PitchGradeModel.txt                One LightGBM model per pitch type (Fastball, Sinker, Slider, ...)
Models & Reference Files/
  final_model_xSwing_model.txt                   xSwing LightGBM model
  final_model_xSwing_features.rds                Feature order for xSwing
  final_model_xCalledStrike_model.xgb            xCalledStrike XGBoost model
  xgb_model_pitchcall_features.rds               Feature order for xCalledStrike
  xgb_model_pitchcall_classmap.rds               Class map for xCalledStrike
  stuff_plus_reference.rds                       Per-pitch-type mean and SD of raw Stuff+ predictions (D1 reference)
  percentiles_pitcher_season_ref.rds             Reference distributions for the top-table percentiles
Image Files/                                     Pitcher silhouettes used by the arm angle plot
```

## Requirements

R packages: `shiny`, `ggplot2`, `dplyr`, `stringr`, `ggimage`, `ggrepel`, `plotly`, `car`, `grid`, `png`, `showtext`, `sysfonts`, `lightgbm`, `xgboost`, `gt`, `tibble`. The training script also uses `rBayesianOptimization`.

Input data, both loaded into the global environment before sourcing the script:

- `CSUF25`: TrackMan pitch data. Core columns used are `Pitcher`, `PitcherTeam`, `PitcherThrows`, `BatterSide`, `Date`, `PitchCall`, `TaggedPitchType`, `Balls`, `Strikes`, `PitchofPA`, `PlateLocSide`, `PlateLocHeight`, `RelSpeed`, `RelHeight`, `RelSide`, `Extension`, `SpinRate`, `SpinAxis`, `InducedVertBreak`, `HorzBreak`, `VertApprAngle`, `HorzApprAngle`, and the 9-parameter trajectory columns `x0`, `z0`, `vx0`, `vz0`, `ax0`, `az0`.
- `D1PitcherHeights`: height and handedness lookup described above.

## Running the app

1. Open `PitchReports.R` and update the hard-coded model, reference, and image paths (they point at a local `www/` folder) to the `Models & Reference Files/`, `Pitch Grade Models/`, and `Image Files/` directories in this repo.
2. Load `CSUF25` and `D1PitcherHeights`.
3. Source the script. It processes the data once on startup (height conversion, xSwing and xCalledStrike predictions, arm angles, Stuff+), filters to `CAL_FUL` pitchers, and launches the app.
4. Pick a pitcher and a date to view the report. Click **Generate PDF** to write a landscape 11 x 8.5 PDF with one page per CSUF pitcher who appeared on that date.

The app fetches the Markazi Text font from Google Fonts on startup and falls back to the system default if the download fails.

## Notes

- Only `CAL_FUL` pitchers appear in the app. The models and reference tables were built on Division I data from multiple programs so the percentiles and Stuff+ are league-relative.
- Stuff+ ignores location by design. A pitch with good Stuff+ and a high Meatball% is a location problem, not a pitch-design problem, and the report is laid out to make that distinction visible.
- Arm angle accuracy depends on the pitcher being in `D1PitcherHeights`. Pitchers without a height entry get `NA` arm angles and the plot falls back to a default silhouette.
