# baikal_calibration_event_classification

## 1) Description

This repository contains a working pipeline that parses the Baikal neutrino telescope calibration session log ([`baikal/d0126_nocuts_calib.log`](http://cluster.inr.ac.ru/num/exam_2024/baikal/)) and classifies events into:

- **Laser calibration events** (synchronized, high-multiplicity flashes used for calibration)
- **Background events** (non-laser events), each assigned a **most likely origin** based on timing, multiplicity, amplitude, and OM hit patterns

The input log is a pulse-level table where each row represents a registered pulse with the following columns:

### Input columns (per pulse)
1. **Event number** — event ID (grows non-uniformly due to soft-trigger selection)
2. **Pulses in event** — total pulses registered in that event (from the log)
3. **Number of pulse in event** — index of this pulse inside the event
4. **OM number** — Optical Module (OM) that recorded the pulse
5. **Amplitude** — pulse amplitude (photoelectrons)
6. **Time** — OM trigger time

The program groups pulses into events, extracts event-level features, detects laser events, and then classifies background events into physically motivated categories (e.g., noise-like, burst-like, OM-local effects, muon-like candidates, unknown).

---

## 2) Technologies Used

- **Python 3**
- **NumPy** — numerical computations and feature extraction
- **pandas** — parsing the log, grouping pulses into events, event-level tables
- **Matplotlib** — diagnostic plots
- **Seaborn** — optional visualization styling

---

## 3) Program Workflow (How the Classification Works)

### Step A — Load data
1. (Colab) Mount Google Drive:
   - `drive.mount('/content/drive')`
2. Read the calibration log text file.
3. Split each line into 6 fields and build a `pandas.DataFrame` with columns:

   - `Event number`
   - `Pulses in event`
   - `Number of pulse in event`
   - `OM number`
   - `Amplitude`
   - `Time`

4. Convert column types:
   - event/pulse/OM columns → integer
   - amplitude/time columns → float

**Result:** a pulse-level table `df`.

---

### Step B — Quick global exploration
The program plots histograms for:
- `df['Amplitude']` — overall amplitude distribution
- `df['Time']` — overall trigger-time distribution

These plots give a fast overview of typical amplitudes and the time window where most activity occurred.

---

### Step C — Per-event diagnostics (event-by-event inspection)
For a chosen event ID interval (example: `range(3000, 3100)`), the program:
1. Filters pulses belonging to a single event:
   - `filtered_df = df[df['Event number'] == event_number]`
2. Plots three histograms for that event:
   - **OM number** (which OMs were hit)
   - **Amplitude**
   - **Time**

Vertical dashed guide-lines are drawn on the OM histogram to visually separate OM groups (useful for spotting whether hits concentrate in one region or are spread across the detector).

**Result:** you can visually recognize patterns such as:
- localized activity (few OMs dominate)
- distributed activity (many OMs across groups)
- narrow vs wide time structure

---

### Step D — Identify laser events (automatic split)
Laser events are detected using event multiplicity.

1. Group pulses by event:
   - `grouped = df.groupby('Event number').count()`
2. Select events with high multiplicity:
   - `laser_ev = grouped[grouped['Pulses in event'] >= 100]`
   - `not_laser_ev = grouped[grouped['Pulses in event'] < 100]`
3. Split the main table:
   - `laser = df[df['Event number'].isin(laser_ev.index)]`
   - `not_laser = df[df['Event number'].isin(not_laser_ev.index)]`

**Result:** two datasets (`laser`, `not_laser`) ready for comparison.

---

### Step E — Compare laser vs background distributions
The program builds the same type of histograms for each subset:
- `laser['Amplitude']` and `laser['Time']`
- `not_laser['Amplitude']` and `not_laser['Time']`
- OM occupancy distributions for laser vs background

This validates the separation:
- laser events typically show a strong, consistent response pattern and high multiplicity
- background events show lower multiplicity and more varied structures

---

### Step F — Inferring background origins (interpretation stage)
After the laser events are removed, background events can be interpreted using the per-event plots:

- **Random noise / dark counts**: very low multiplicity, weak amplitudes, no clear OM structure
- **Local OM effects (afterpulses / artifacts)**: one OM (or a tiny set) dominates, often with clustered times
- **Burst-like optical noise**: elevated multiplicity but less coherent than laser, possibly clustered in time
- **Muon-like candidates**: multiple OMs with a broader time structure and more distributed OM hits

---


### Step G — OM occupancy overview (detector structure check)
1. The program builds a global histogram of `OM number` using all pulses:
   - `plt.hist(df['OM number'], bins=np.linspace(0, 130, 130), ...)`

2. It overlays guide lines to visually separate detector “strings” and reference OMs:
   - **Blue dashed lines** at `24, 48, 72, 96` mark boundaries (groups of 24 OMs).
   - **Red dashed lines** at `5, 29, 53, 77, 101` mark the same OM index across strings  
     (e.g., the 5th OM on each 24-OM block).

It helps confirm that the OM numbering matches the expected detector layout and makes it easier to interpret “where” events hit.

---

### Step H — Geometry-based light travel time (using OM coordinates)
If `coords.txt` is available, the program loads OM positions and computes the distance and expected light travel time from the laser.

1. Load coordinates from `coords.txt` with columns:
   - `N` (OM id), `X`, `Y`, `Z` (coordinates)

2. Set the laser position as a fixed 3D point:
   - `laser_coords = [9830.0, -1100.0, 6890.0]`

3. For each OM, compute:
   - **Distance:**  
     `Distance = ||(X, Y, Z) - laser_coords||`
   - **Propagation time estimate:**  
     `Time = Distance * (4.57 / 100)`

   > Note: the factor `4.57/100` is used directly as in the code to convert the distance scale to a time scale.

4. Plot the distance distribution and compute the time spread between farthest and nearest OMs:
   - `df2['Time'].max() - df2['Time'].min()`

This provides a reference scale for how much time spread is expected purely from geometry (different OM distances to the laser).

---

### Step I — Event-level feature engineering
Next, the program converts the pulse-level table (`df`) into event-level metrics by grouping on `Event number`.

For each event, it computes:

- **event_count** — number of pulses in the event  
  (`df.groupby('Event number')['Pulses in event'].count()`)

- **total_amplitude** — sum of pulse amplitudes in the event  
  (`df.groupby('Event number')['Amplitude'].sum()`)

- **time_diff** — time span of the event  
  (`max(Time) - min(Time)` within the same event)

These three features are the base “signature” used for classification.

---

### Step J — Diagnostic scatter plots 
The pipeline visualizes relationships between features to reveal natural clusters:

- `time_diff` vs `total_amplitude`
- `time_diff` vs `event_count`
- `event_count` vs `total_amplitude`

**Interpretation goal:** events with high multiplicity and high total amplitude often form distinct groups compared to low-multiplicity background.

---

### Step K — Unsupervised classification with K-Means
To automatically separate event types, the program applies K-Means clustering.

1. Build a feature table:
   - `event_count`, `time_diff`, `total_amplitude`

2. Normalize features:
   - `StandardScaler().fit_transform(...)`  
   This prevents one large-scale feature from dominating clustering.

3. Fit K-Means with 3 clusters:
   - `KMeans(n_clusters=3, init=init_points, random_state=42)`
   - Uses explicit `init_points` to stabilize cluster placement.

4. Visualize clustering result on feature pairs using the assigned labels.

5. Store cluster labels per event:
   - `pred` is the cluster ID assigned to each `Event number`.

**Result:** every event receives a cluster label (`pred ∈ {0,1,2}`).

---

### Step L —  Mean amplitude and energy-like calibration (cluster interpretation)
After clustering, the program computes additional derived quantities:

1. **Mean amplitude per event**
   - `Mean amplitude = total_amplitude / event_count`

2. Identify a laser-like cluster (in the current notebook this is treated as `pred == 0`), then:
   - build a histogram of `Mean amplitude` for that cluster
   - estimate the “typical” amplitude using the histogram peak

3. Convert amplitude to an energy-like scale using a constant defined in code:
   - `laser_energy = 4.1e-15 / (480e-9 * 4.57e-9)`
   - `coef = laser_energy / laser_peak_amplitude`
   - `Mean energy = Mean amplitude * coef`

This step documents exactly what the program computes. The constants are taken from the notebook as-is and can be adjusted if a different calibration model is required.

---

### Step M —  Per-cluster distributions (final sanity checks)
Finally, the program plots feature distributions separately for each cluster:

- `Mean energy` per cluster
- `total_amplitude` (“total energy” proxy) per cluster
- `event_count` per cluster
- `time_diff` per cluster

These plots are used to interpret what each cluster represents, for example:
- low-multiplicity / low-energy background-like events
- high-multiplicity / high-energy laser-like events
- intermediate populations or distinct laser regimes (depending on the dataset)

---

### Step N —  Main reusable outputs
At the end of this part of the pipeline, the key tables are:

- **`preds`**: mapping `Event number -> pred` (cluster label)
- **`data_for_cluster`**: per-event feature table containing:
  - `event_count`, `time_diff`, `total_amplitude`
  - `pred` (cluster ID)
  - `Mean amplitude`
  - `Mean energy` (computed using the notebook’s calibration constant)

## 4) What the program outputs (final results)
**Background-only clustering (no laser events)**
What “no laser” means here
Laser events are removed first (event_count < 100).

## Физическая интерпретация 3 кластеров (цикл без лазера)
> Important: In a short calibration run, truly astrophysical neutrino events are expected to be extremely rare.

### Cluster A — Low-multiplicity, low-amplitude (“single / dim” events)
**Typical signature**
- small `event_count`
- small `total_amplitude`
- often very small `time_diff` (compact in time)

**Most likely origin**
- **PMT dark noise / thermionic emission**
- **Natural radioactivity** in materials and water (e.g., radioactive decay chains, potassium-like backgrounds),
  producing small, random light signals and single-photoelectron pulses
- occasional **electronics noise** / cross-talk (if timing looks unnatural)

**Why it forms a separate cluster**
These events dominate by sheer count and occupy the “bottom-left” of feature space (low light, few hits).

---

### Cluster B — Moderate multiplicity with broader time structure (“burst / correlated noise”)
**Typical signature**
- medium `event_count`
- low-to-medium `total_amplitude`
- noticeably larger or more variable `time_diff` than Cluster A

**Most likely origin**
- **Optical background bursts** (in-water light emission bursts):
  - bioluminescence-like activity (bursty light from biological sources)
  - other correlated optical noise processes (water luminescence, local light emission)
- **Afterpulses / local OM effects** that create time-extended pulse trains
- sometimes mixed with **low-energy muon activity** if the detector sees partial tracks

**Why it forms a separate cluster**
These events are not purely random singles: they show time correlation and/or multi-hit structure,
but still do not look like bright, track-like events.

---

### Cluster C — High multiplicity and high total amplitude (“track/shower-like” events)
**Typical signature**
- large `event_count`
- large `total_amplitude`
- `time_diff` consistent with a particle crossing the detector (often not as “instantaneous” as a flash)

**Most likely origin**
- **Atmospheric muons / muon bundles** produced by cosmic-ray air showers  
  (this is typically the dominant bright background in neutrino telescopes)
- **Electromagnetic/hadronic showers** associated with cosmic
