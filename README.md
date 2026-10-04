# Mining Behavior Patterns from Cyclist GPS Trajectories

How do cyclists actually ride through a city? This project turns raw smartphone GPS traces from Berlin into ride-level features (speed, stops, turns, braking) and looks for patterns in riding behavior and near-miss incidents.

**CSCI 4900/6000 · Team 3: Steven Gobran & Kevin Yassa · Middle Tennessee State University**

<img width="1269" height="952" alt="image" src="https://github.com/user-attachments/assets/970fb30d-9a2a-4a5e-a485-5462439a4fe4" />

*Every cleaned ride across Berlin. Brighter lines are faster riding. Red dots are near-miss incidents, and the larger ones were marked scary.*

## Problem

Cities usually count cyclists but don't know **how** they ride: which streets they pick, where they keep stopping, or which junctions feel unsafe. This project asks:

> Can raw smartphone GPS traces be turned into clear patterns of where cyclists speed up, stop, turn, and run into danger?

**Why it matters:** if planners know where riders stop and where near misses cluster, they can fix those spots first instead of guessing. Berlin already shares this data with its transport department, and the same method works for any city with bike ride data.

## Objectives

1. **Clean and map-match** noisy GPS points onto the OpenStreetMap road network
2. **Extract features** for every ride: speed, acceleration, stops and turns
3. **Find riding styles** by clustering rides
4. **Map hot spots**: popular routes and junctions with frequent stops and near misses

Objectives 2 and 3 are done for the midterm sample. Map matching and hot spots are the next phase. 

<img width="1316" height="909" alt="image" src="https://github.com/user-attachments/assets/7d0a3679-8544-4752-a193-70abd5e3f503" />


## Key results

| | Result |
|---|---|
| Ride files analyzed | 877 |
| Clean rides after filtering | **856** |
| GPS points after cleaning | 375,741 |
| Total distance ridden | 4,936 km |
| Tagged near-miss incidents | **408** |
| Typical ride | about 4 km, 15 min, 18.5 km/h while moving |

**Main findings**

- **Close passes are the biggest danger.** Half of all incidents are close passes, and 73% involve a car.
- **Incidents follow the traffic.** They cluster in the dense inner city along the busiest corridors.
- **More turns means slower riding** (correlation of -0.46).
- **Two real riding styles show up:** steady cruisers and stop-and-go riders.
- **Phone noise can look like behavior.** Acceleration variability tracks GPS error closely (0.73), and one cluster was driven by phone position, not riding style. That's why map matching and smoothing come next.

## Data

**Source:** [SimRa dataset](https://github.com/simra-project/dataset) from TU Berlin. Cyclists record rides with the SimRa phone app and tag near-miss incidents afterward.

This project uses the **June 2023** monthly batch (`Berlin_2023_06`), with 877 ride files.

Each ride file has two parts:

| Part | Columns | Notes |
|---|---|---|
| Incident block | `lat, lon, ts, bike, pLoc, incident, i1..i10, scary` | Rider-labeled near misses: type, who was involved, scary flag |
| Ride time series | `lat, lon, X, Y, Z, timeStamp, acc, a, b, c` | GPS about every 3 s, with accelerometer and gyroscope rows in between |

The codes for bike type, phone position and incident type are explained in the dataset's [`legend.txt`](https://github.com/simra-project/dataset/blob/master/legend.txt). 

<img width="1200" height="1008" alt="image" src="https://github.com/user-attachments/assets/cd4fee4b-cffb-4f4a-b51c-b0cf91ef1bbb" /> 



## Data cleaning

| Step | Rule | Removed |
|---|---|---:|
| 1. Split the file | Separate incidents from the ride series; keep rows with a GPS fix | |
| 2. Drop bad fixes | GPS accuracy radius over 20 m | 4,315 fixes |
| 3. Remove spikes | Jumps implying over 54 km/h between fixes | 519 fixes |
| 4. Drop weak rides | Unreadable, under 300 m, or under 2 minutes | 21 rides |

Features built for each ride:

- **Speed:** average, moving (excluding stops) and 95th percentile
- **Stops:** 5 seconds or longer below 1 m/s, per km, and share of time stopped
- **Turns:** heading change over 45° while moving, per km
- **Braking:** acceleration variability and hard braking events

## Exploratory analysis

### Summary statistics (856 rides)

| Feature | Mean | Median | Std dev |
|---|---:|---:|---:|
| Distance (km) | 5.77 | 4.10 | 5.44 |
| Duration (min) | 22.3 | 15.5 | 26.2 |
| Moving speed (km/h) | 18.5 | 18.5 | 3.0 |
| 95th pct speed (km/h) | 26.5 | 26.3 | 4.7 |
| Stops per km | 1.51 | 1.18 | 1.74 |
| Share of time stopped | 14% | 11% | 11% |
| Turns per km | 2.16 | 1.68 | 2.01 |
| GPS error radius (m) | 6.5 | 5.4 | 3.8 |

Distance and duration are right-skewed: a few long rides pull the mean above the median.

### Speed and stopping

![Speed and stop distributions](images/dist.png)

Speed forms a bell shape around 18.5 km/h. Stops have a long tail: most rides stop about once per km, but some stop constantly.

### Feature correlations

![Correlation heatmap](images/corr.png)

Spearman correlations between ride features. The key warning for this method: acceleration variability and GPS error are strongly linked (0.73), so some "jerky" riding is really a bad signal.

### Near-miss incidents

![Incident types](images/incidents.png)

| | |
|---|---|
| Close passes | 50% of incidents |
| Involve a car | 73% |
| Incidents per 100 km | 8.3 |
| Rides with at least one incident | 143 (17%) |
| Marked scary | 26% |

Head-on approaches are rare, but over half of them were marked scary.

### Riding styles (preliminary clustering)

![Riding style clusters](images/clusters.png)

K-means (k = 3) on six standardized features:

| Cluster | Rides | Profile |
|---|---:|---|
| Steady cruisers | 287 | Fastest (21.4 km/h moving), few turns, longer rides |
| Stop-and-go | 242 | Slowest (15.6 km/h), about 2 stops per km |
| Noisy-GPS rides | 327 | 96% had the phone in a pocket. A data artifact, not a riding style |

The silhouette score is 0.25, so the groups overlap. This is a first pass, and it shows exactly what needs fixing before the final clustering.

## Next steps

- [ ] Map-match every ride to the OpenStreetMap Berlin bike network
- [ ] Smooth GPS and control for phone position before clustering
- [ ] Scale up to more monthly batches and years
- [ ] Re-cluster, then summarize stops and incidents by street segment and junction

## Run it yourself

```bash
pip install pandas numpy matplotlib scikit-learn jupyter
jupyter notebook cyclist_eda.ipynb
```

Put these files in the same folder as the notebook:

| File | Description |
|---|---|
| `rides_cl.csv` | One row per cleaned ride with all features and cluster labels (856 rows) |
| `incidents.csv` | One row per near-miss incident (408 rows) |
| `points.csv` | Every cleaned GPS point, only needed for the map (375,741 rows) |

The notebook loads these CSVs by default. To rebuild them from the raw SimRa files, download the ride files into a `rides/` folder and set `RUN_PARSE = True` in the setup cell.

## Repo structure

```
├── README.md
├── cyclist_eda.ipynb      # Full cleaning and EDA notebook
├── rides_cl.csv           # Cleaned ride features
├── incidents.csv          # Near-miss incidents
├── points.csv             # Cleaned GPS points
└── images/
    ├── map.png
    ├── dist.png
    ├── corr.png
    ├── incidents.png
    └── clusters.png
```

## References

- Karakaya, A.-S., Hasenburg, J., & Bermbach, D. (2020). [SimRa: Using crowdsourcing to identify near miss hotspots in bicycle traffic](https://doi.org/10.1016/j.pmcj.2020.101197). *Pervasive and Mobile Computing*, 67.
- Newson, P., & Krumm, J. (2009). [Hidden Markov map matching through noise and sparseness](https://doi.org/10.1145/1653771.1653818). *ACM SIGSPATIAL GIS*.
- Yang, C., & Gidófalvi, G. (2018). [Fast map matching, an algorithm integrating hidden Markov model with precomputation](https://doi.org/10.1080/13658816.2017.1400548). *IJGIS*, 32(3).

## Tools

Python · pandas · NumPy · scikit-learn · Matplotlib · Jupyter

---

*Steven Gobran & Kevin Yassa · Middle Tennessee State University*
