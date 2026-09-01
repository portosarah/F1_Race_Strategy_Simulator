# F1 Race Strategy Simulator

## Overview

This project was developed as a practical way to learn how data science can be applied to Formula 1 race strategy, starting with a simple question: how much can tyre strategy influence race time?

The simulator explores how tyre degradation and race progression affect lap-time performance, which race strategies are more likely to produce faster outcomes, and how uncertainty may change the comparison between alternative strategies.

Historical lap and tyre data are first cleaned to remove observations that are not representative of normal race pace. An OLS regression model is then used to estimate race progression, tyre-age effects, compound-specific degradation, and driver-related pace differences. These estimated parameters are subsequently used in a race strategy simulator, while Monte Carlo simulation is applied to represent lap-time uncertainty and compare alternative strategies probabilistically.

The 2025 Bahrain Grand Prix was selected as the initial case study because tyre management and multi-stint race strategy play an important role in the event, making it a useful starting point for studying tyre degradation and strategy simulation.

## Research Questions

1. How do tyre degradation and race progression influence lap-time performance?
2. How can alternative tyre strategies affect simulated total race time?
3. How does lap-time uncertainty influence the probability that one strategy outperforms another?

## Dataset

Historical Formula 1 data for this project were obtained from the public
[`tapanBabbar9/f1`](https://github.com/tapanBabbar9/f1/tree/main/dataset)
dataset repository.

The current analysis uses four source files:

| File | Purpose |
|---|---|
| `races.csv` | Identifies race events, seasons, rounds, and race IDs |
| `lap_times.csv` | Provides driver lap times, race position, and lap-level timing information |
| `tyre_laps.csv` | Provides tyre compound, stint number, tyre life, and driver code for each lap |
| `drivers.csv` | Provides additional driver metadata that can be linked through `driverId` |

For the initial case study, the data were filtered to the **2025 Bahrain Grand Prix (`raceId = 1148`)**. The lap timing and tyre datasets each contained 1,128 Bahrain lap records before cleaning. :contentReference[oaicite:1]{index=1}

After the lap timing and tyre datasets were merged, each observation represents one driver's car performance on one race lap together with the tyre and stint conditions associated with that lap.

The raw datasets are not duplicated in this repository because they are already publicly available from the original data source. This repository instead focuses on the analysis workflow, modelling, simulation, and generated results.

### Data Cleaning

The raw Bahrain dataset was cleaned to focus the modelling process on laps that were more representative of normal on-track race pace.

The cleaning process excluded:

- **Lap 1**, because standing-start effects, first-lap traffic, and field bunching make it substantially different from normal racing laps.
- **Pit-adjacent laps**, because pit entry and pit exit effects can substantially increase lap time without representing underlying race pace or tyre degradation.
- **Abnormally slow laps**, identified using a robust median and median absolute deviation (MAD)-based approach within each driver stint, to reduce the influence of non-pace events and unusually slow running.
- **Very short stints**, with fewer than four clean laps, because they provide limited information for estimating within-stint pace behavior.

The objective of this cleaning process was not simply to remove slow laps, but to construct a modelling dataset that better represents normal race-pace behavior for estimating race progression and tyre degradation.

The cleaning pipeline reduced the Bahrain GP dataset from **1,128 raw lap observations to 966 modelling laps**, retaining approximately **85.6%** of the original observations.

### Pace and Tyre Degradation Modelling

An OLS regression model was used to estimate changes in lap time associated with tyre age while accounting for other observable sources of pace variation.

The model includes:

- **Race progression (`raceLap`)** to account for broad lap-time changes throughout the race, such as those associated with fuel burn and evolving race conditions.
- **Tyre age (`tyreLife`)** to estimate how lap time changes as a tyre accumulates laps.
- **Tyre compound** to account for baseline pace differences between Soft, Medium, and Hard tyres.
- **Tyre age × compound interactions** to allow the estimated degradation rate to differ between compounds.
- **Driver fixed effects (`driverCode`)** to account for persistent driver/car-specific pace differences within the race.

The objective is therefore not to measure a purely causal tyre-degradation effect, but to estimate tyre-age-related pace changes while controlling for several major observable sources of lap-time variation.

### Monte Carlo Strategy Simulation

The fitted pace model provides an expected lap time for a given race lap, tyre age, and compound. However, actual lap performance is not deterministic and can vary around the model prediction.

To incorporate this uncertainty, stochastic variation based on the regression residuals is added to simulated lap times. Thousands of simulated races are then generated for each strategy.

Rather than identifying a single deterministic "best" strategy, the Monte Carlo analysis estimates how frequently one strategy produces a lower total race time than another under the modelled lap-time uncertainty.

In the baseline experiment, 10,000 simulations were used to compare two alternatives 57-lap Bahrain strategies.

## Results

### Pace Model

The compound-specific OLS model was fitted using **966 cleaned lap observations** from the 2025 Bahrain Grand Prix.

The model achieved an **R² of 0.744**, indicating that approximately 74.4% of the observed lap-time variation in this dataset was explained by the variables included in the model.

The estimated race-progression coefficient was:

- **−0.0626 s/lap**

This indicates that, after accounting for tyre age, compound, and driver/car-specific baseline differences, later race laps tended to be faster in the Bahrain dataset.

This coefficient should be interpreted as a broad race-progression effect rather than a direct fuel-load estimate.

### Estimated Tyre Degradation

The compound-specific model produced the following degradation point estimates:

| Compound | Estimated degradation |
|---|---:|
| Soft | +0.1216 s/lap |
| Medium | +0.1043 s/lap |
| Hard | +0.1156 s/lap |

Positive values indicate increasing lap time as tyre age increases.

These estimates suggest that Medium tyres had the lowest degradation rate in this model, while Soft tyres had the highest estimated degradation rate.

However, differences between the compound degradation slopes should be interpreted cautiously. The Medium–Hard interaction was not conventionally statistically significant (`p = 0.080`), while the Soft–Hard interaction was also not significant (`p = 0.434`). Therefore, the values are treated as single-race point estimates rather than evidence of definitive compound performance differences.

### Monte Carlo Strategy Comparison

Two alternative 57-lap strategies were evaluated using **10,000 Monte Carlo simulations**.

| Result | Value |
|---|---:|
| Strategy A produced the faster race time | 31.6% |
| Strategy B produced the faster race time | 68.4% |
| Median A − B difference | +2.73 s |
| Mean A − B difference | +2.66 s |
| 90% simulation interval | −6.77 to +11.95 s |

The race-time difference is defined as:

`Strategy A total time − Strategy B total time`

Therefore, a positive value indicates an advantage for **Strategy B**, while a negative value indicates an advantage for **Strategy A**.

The median difference of **+2.73 seconds** suggests that Strategy B was typically around 2.73 seconds faster under the current simulation assumptions.

However, the 90% simulation interval crosses zero. This means that while Strategy B was favored overall, Strategy A still produced the faster race time under some simulated conditions. The result therefore represents a probabilistic strategy advantage rather than a guaranteed superior strategy.

## Limitations

The current simulator is an initial single-race model and therefore represents a simplified version of Formula 1 race strategy.

Several factors that can influence real race performance are not yet explicitly modelled:

- **Pit-stop loss:** the current simulator still uses a placeholder pit-loss value rather than a value calibrated from historical Bahrain pit-stop data.
- **Race-control events:** Safety Cars, Virtual Safety Cars, yellow flags, and red flags can substantially alter lap times and the strategic value of a pit stop.
- **Weather and track conditions:** changes in temperature, rainfall, wind, and track evolution are not explicitly represented.
- **Traffic and track position:** the current model does not simulate overtaking, dirty air, time lost behind other cars, or undercut/overcut interactions.
- **Detailed telemetry:** variables such as throttle application, braking behavior, speed, and other car-performance signals are not currently included.
- **Mechanical and incident-related effects:** failures, damage, spins, and other race incidents are not explicitly simulated.
- **Single-race calibration:** the pace and tyre-degradation estimates are currently derived only from the 2025 Bahrain Grand Prix and should not be assumed to generalize to other circuits or seasons.

The Monte Carlo component currently represents uncertainty primarily through stochastic lap-time variation based on the fitted regression residuals. It therefore captures only part of the uncertainty present in an actual Formula 1 race.

## Next Steps

The current Bahrain GP model represents the first working version of the simulator. Future development will focus on improving both model calibration and race realism.

Planned improvements include:

1. **Calibrate pit-stop loss using real Bahrain 2025 race data** rather than relying on the current placeholder value.
2. **Validate the simulator against historical race strategies** to evaluate how closely simulated outcomes reflect actual race performance.
3. **Extend the model to additional 2025 races** to examine whether tyre degradation and pace parameters generalize across circuits.
4. **Model additional sources of race uncertainty**, including Safety Cars, Virtual Safety Cars, yellow/red flag conditions, traffic, and pit-stop variability.
5. **Incorporate weather and track-condition information** where suitable data are available.
6. **Explore telemetry-based features**, such as throttle, braking, and speed, for more detailed pace modelling.
7. **Evaluate 2026 race data separately**, rather than assuming that parameters estimated from the 2025 season remain applicable.

## Repository Structure

```text
F1-Race-Strategy-Simulator/
│
├── 01_F1_Race_Strategy_Simulator_Bahrain_2025.ipynb
├── .gitignore
├── LICENSE
├── README.md
└── requirements.txt

## How to Run

The current version of this project is designed to run in **Google Colab**.

### 1. Open the notebook

Open `Bahrain_2025.ipynb` in Google Colab.

### 2. Obtain the dataset

Download the following CSV files from the public F1 dataset:

- `races.csv`
- `drivers.csv`
- `lap_times.csv`
- `tyre_laps.csv`

Dataset source:  
https://github.com/tapanBabbar9/f1/tree/main/dataset

### 3. Prepare the data directory

Create the following directory in Google Drive:

```text
MyDrive/
└── F1-Race-Strategy-Simulator/
    └── data/
        └── source/
            ├── races.csv
            ├── drivers.csv
            ├── lap_times.csv
            └── tyre_laps.csv
The notebook currently reads the source data from:
/content/drive/MyDrive/F1-Race-Strategy-Simulator/data/source/

### 4. Install dependencies

Required Python packages are listed in requirements.txt.

In Google Colab, the required packages can also be installed directly if needed:
!pip install numpy pandas matplotlib statsmodels fastf1

### 5. Run the notebook
Run the notebook cells sequentially from top to bottom.

The workflow performs:
Data loading and preparation
Bahrain GP 2025 filtering
Lap-data cleaning
Pace and tyre-degradation modelling
Strategy simulation
Monte Carlo strategy comparison

Running outside Google Colab
The current notebook uses a Google Drive-specific file path. If running locally or in another environment, update PROJECT_DIR and SOURCE_DIR to match the location of the dataset on your system.
