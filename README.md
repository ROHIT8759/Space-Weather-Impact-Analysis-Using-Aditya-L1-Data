# Aditya-L1 Space Weather Workshop — Project 6

## Short intro

This repository holds a hands-on workshop exercise that uses real **Aditya-L1** Level-2
(science-ready) data to ask one question: *given only what India's first dedicated solar
observatory measured 1.5 million km upstream of Earth, could you have predicted the geomagnetic
storm that followed — and how close would you get?*

Project 6 (`Project6_SpaceWeather_AdityaL1_Students.ipynb`) takes the magnetic-field and
solar-wind measurements from Aditya-L1's **MAG** and **ASPEX-SWIS** payloads for **10–11 October
2024**, propagates them from the spacecraft to Earth's bow shock, and runs a classic ring-current
model (Burton 1975 / O'Brien & McPherron 2000) to predict the SYM-H geomagnetic index — then
scores that prediction against the real SYM-H/AE indices from NASA's OMNI database.

Everything below unpacks that one paragraph in increasing detail: what's in the repo, what every
file and every column actually contains, how the notebook gets from raw bytes to a predicted
storm, where a real data gap forced an adaptation, and what the final numbers came out to.

---

## 1. Repository layout

```
Aditya_Workshop/
├── README.md                                     <- this file
├── work.md                                        <- the mentor's project brief (read this first)
├── ASPEX_MAG_Overview_Students_Copy.ipynb         <- shared REFERENCE notebook (do not edit)
├── Project6_SpaceWeather_AdityaL1_Students.ipynb  <- the completed, executable TARGET notebook
└── data/
    ├── Project6_SpaceWeather_AdityaL1_Students.ipynb  <- mirror copy of the target notebook
    ├── L2_AL1_MAG_20241010_V00.nc                      <- MAG, 10 Oct 2024
    ├── L2_AL1_MAG_20241011_V00.nc                      <- MAG, 11 Oct 2024
    ├── AL1_ASW91_L2_TH1_20241009_UNP_9999_999999_V03.cdf  <- SWIS THA-1, 09 Oct
    ├── AL1_ASW91_L2_TH1_20241010_UNP_9999_999999_V03.cdf  <- SWIS THA-1, 10 Oct
    ├── AL1_ASW91_L2_TH1_20241011_UNP_9999_999999_V03.cdf  <- SWIS THA-1, 11 Oct
    ├── AL1_ASW91_L2_TH2_20241009_UNP_9999_999999_V03.cdf  <- SWIS THA-2, 09 Oct
    ├── AL1_ASW91_L2_TH2_20241010_UNP_9999_999999_V03.cdf  <- SWIS THA-2, 10 Oct
    ├── AL1_ASW91_L2_TH2_20241011_UNP_9999_999999_V03.cdf  <- SWIS THA-2, 11 Oct
    └── omni/                                           <- OMNI SYM-H/AE cache (created on first run)
        └── omni_hro2_1min_20241001_v01.cdf
```

The notebook resolves its own data directory at runtime (`data/` from the project root, or `.` if
launched from inside `data/`), so it runs correctly from either location.

Only **10–11 October 2024** is in scope — that is the event window `work.md` specifies, and it is
exactly what the MAG files on disk cover. The `20241009` TH1/TH2 files exist but fall outside the
required window and are not used. No `2024-10-12` data exists anywhere in this repo, and none is
required.

---

## 2. The event and the physics, in brief

On 10 October 2024 a fast coronal mass ejection (CME) swept past Aditya-L1, sitting at the Sun–
Earth L1 point (~259 Earth radii, ~1.65 million km sunward of Earth). The same structure went on
to drive one of the most intense geomagnetic storms of solar cycle 25 (minimum SYM-H ≈ −390 nT —
a "super" storm). The chain of physics linking the two:

1. **Southward IMF** (`Bz < 0`, GSM) lets the solar wind's field reconnect with Earth's dayside
   field, opening the magnetosphere.
2. The **merging electric field** `Ey = −Vx·Bz` quantifies how fast that opening happens.
3. Energised ions drift westward, building a **ring current** whose field opposes Earth's dipole
   at the surface — measured by **SYM-H**.
4. **Dynamic pressure** does something different: it *compresses* the magnetosphere rather than
   energising it, producing a brief positive SYM-H spike (the storm sudden commencement) before
   the storm proper.

Because Aditya-L1 sits upstream, nothing it measures is "at Earth" until a **propagation delay**
— driven by the measured solar-wind speed, not a constant — has been added. That delay, and the
rest of the chain, is what Sections 9–14 of the notebook compute.

---

## 3. The data, file by file, column by column

### 3.1 MAG — magnetic field vector (`L2_AL1_MAG_<date>_V00.nc`, NetCDF-4)

Two tri-axial fluxgate magnetometers (MAG1 at 6 m, MAG2 at 3 m boom distance) combine at Level-2
into one field vector, resampled to a minimum 10 s cadence. One file per day; 8,640 samples/day.

| Variable | Units | Meaning |
|---|---|---|
| `time` | s (UNIX) | sample timestamp |
| `Bx_gse`, `By_gse`, `Bz_gse` | nT | field vector, Geocentric Solar Ecliptic frame |
| `Bx_gsm`, `By_gsm`, `Bz_gsm` | nT | field vector, Geocentric Solar Magnetospheric frame |
| `Bx_gse_error` … `Bz_gsm_error` | nT | per-component 1-σ uncertainty |
| `Quality_flag_10s_data` | — | `1` = good, `0` = bad/missing |
| `x_gse`, `y_gse`, `z_gse` | km | spacecraft position, GSE |
| `x_gsm`, `y_gsm`, `z_gsm` | km | spacecraft position, GSM |

Fill value: `-9999.0` → converted to `NaN`. In this dataset **every** sample passes QC
(`Quality_flag_10s_data == 1` for all 17,280 rows across the two days; zero gaps longer than 15 s).

**Derived in the notebook:** `B_total = |B|` (nT), `clock = atan2(By_gsm, Bz_gsm)` (deg, 0 =
northward), `cone = acos(Bx_gse/|B|)` (deg, 0 = sunward).

### 3.2 ASPEX-SWIS — raw flux spectra (`AL1_ASW91_L2_TH{1,2}_<date>_UNP_9999_999999_V03.cdf`, CDF)

**This is the one place the dataset differs from the reference notebook's assumptions — see
§4 for why, and §5 for exactly what was done about it.** THA-1 and THA-2 are two sensor heads of
the Solar Wind Ion Spectrometer; each writes one file per day, 17,275 samples/day at 5 s cadence.

| Variable | Shape | Units | Meaning |
|---|---|---|---|
| `epoch_for_cdf_mod` | (17275,) | ms | sample timestamp (CDF_EPOCH) |
| `energy_center_mod` | (17275, 50) | eV | centre energy of each of 50 analyser steps (constant in time) |
| `energy_uncer` | scalar | % | energy-bin uncertainty |
| `integrated_flux_mod` | (17275, 50) | eV/(cm²·sr·s·eV) | combined (all-anode) differential number flux per energy bin |
| `flux_uncer` | scalar | % | flux uncertainty |
| `integrated_flux_s9_mod`,`s10_mod`,`s11_mod` (TH1) | (17275, 50) each | same | flux from 3 individual anodes/sectors of the 16 TH1 carries |
| `integrated_flux_s15_mod`…`s19_mod` (TH2) | (17275, 50) each | same | flux from 5 individual anodes/sectors of the 32 TH2 carries |
| `spacecraft_xpos`, `ypos`, `zpos` | (17275,) | km | spacecraft position, GSE |
| `sun_angle_tha1` (TH1) / `sun_angle_tha2` (TH2) | (17275, 16, 3) / (17275, 32, 3) | degrees | per-anode look-direction angle relative to the Sun |

Fill value `-1e31`, already converted to `NaN` by `cdflib`. At any one 5 s sample, typically only
~9–50 (median ~25) of the 50 energy bins come back finite — the rest are below the counting
threshold. Physically that's expected: the solar-wind core beam occupies a narrow energy range
around `½m_pV²`, so only the bins straddling the instantaneous bulk speed have real counts.

**Variables that do *not* exist in these files** (present in the reference notebook's expected
product but absent here): `proton_density`, `proton_bulk_speed`, `proton_xvelocity/yvelocity/
zvelocity`, `proton_thermal`, `alpha_density`, `alpha_bulk_speed`, `alpha_thermal`, and the
`*_uncer` companions for each. See §4.

### 3.3 OMNI — ground truth (`data/omni/omni_hro2_1min_20241001_v01.cdf`, downloaded on first run)

Downloaded automatically from NASA/SPDF (`spdf.gsfc.nasa.gov/pub/data/omni/omni_cdaweb/hro2_1min/`)
if no local OMNIWeb text file is found at `data/omni/omni_symh_ae.txt`. 1-minute cadence.

| Variable | Units | Meaning |
|---|---|---|
| `Epoch` | CDF_EPOCH | sample timestamp |
| `SYM_H` | nT | 1-minute symmetric-H index — globally averaged low-latitude ring-current depression (≈ Dst at higher cadence) |
| `AE_INDEX` | nT | auroral electrojet index — substorm/high-latitude activity |

Fill value `99999`, masked to `NaN`. **Used in exactly one role in this notebook: as the answer
sheet the Aditya-L1-only prediction is scored against. It is never an input to any calculation.**

### 3.4 Derived quantities — the common 1-minute table (`p` / `sh` in the notebook)

Built once MAG (10 s) and SWIS (5 s, or SWIS-derived — see §4) are both resampled to 1-minute
means and outer-joined on time:

| Column | Formula | Units | Notes |
|---|---|---|---|
| `Tp` | `m_p w² / 2k_B` | K | proton temperature from thermal speed |
| `Pdyn` | `1.6726×10⁻⁶ (n_p + 4n_α) V²` | nPa | dynamic pressure; `n_α` treated as 0 here (§4) |
| `beta` | `2μ₀ n k_B T / B²` | — | plasma beta; ≪1 marks a magnetic cloud |
| `Va` | `B / √(μ₀ n m_p)` | km/s | Alfvén speed |
| `Ma` | `V / Va` | — | Alfvén Mach number |
| `Cs` | `√(γ k_B T / m_p)` | km/s | sound speed, `γ = 5/3` |
| `Vms` | `√(Va² + Cs²)` | km/s | magnetosonic speed |
| `Mms` | `V / Vms` | — | magnetosonic Mach number |
| `Ey` | `−|Vx|·Bz_gsm` | mV/m | merging electric field; positive = driving |
| `alpha_ratio` | `n_α / n_p` | — | not resolvable here, see §4 |
| `r0` | Shue et al. (1998) | R_E | subsolar magnetopause standoff |
| `X_BS` | Farris & Russell (1994) | R_E | bow-shock nose standoff |
| `delay_min` | `(X_sc − X_BS)·R_E / |Vx|` | min | propagation delay, model bow shock |
| `delay_fixed_min` | same, fixed 14 R_E | min | naive constant-nose comparison |

The **shifted** table `sh` carries the same columns, re-indexed by Earth-arrival time
(`t + delay_min`), re-binned to a clean 1-minute grid, with only the interior gaps the shift itself
creates bridged by time interpolation.

### 3.5 Prediction outputs (Section 14)

| Variable | Formula | Units |
|---|---|---|
| `Dst_star` | Burton (1975) leaky-bucket integral of `Ey` | nT |
| `SYMH_pred` | `Dst_star + 7.26·√Pdyn − 11` | nT |

---

## 4. The data gap, and why the notebook looks the way it does

The reference notebook (`ASPEX_MAG_Overview_Students_Copy.ipynb`) assumes ASPEX-SWIS delivers a
**BLK** product: proton/alpha density, bulk velocity components, and thermal speed, already fitted
by the instrument team. **No BLK file exists anywhere in this repository.** What's actually on disk
is `TH1`/`TH2` — confirmed via each CDF's global attributes (`Data_type: L2>Level-2 Flux spectrum
data products`, `TEXT: ...Level 2 flux data from THA-1/THA-2 package...`) to be **raw differential
flux spectra**, with no density, velocity, or thermal-speed variable at all.

Since the entire Project 6 analysis after Section 8 — the propagation delay, `Ey`, dynamic
pressure, and the Burton/O'Brien-McPherron prediction — depends on proton bulk velocity and
density, this is a hard blocker unless those quantities are derived from the raw spectra. Per
mentor guidance ("use that data only to make a short model... they need to run the application"),
the notebook derives them directly, rather than leaving Sections 9–14 unexecuted:

1. **Combine both sensor heads.** TH1 and TH2 share an identical 5 s time grid; their 50-channel
   spectra are concatenated into one 100-channel spectrum per timestamp.
2. **Convert energy to speed.** `v_i = √(2·E_i·q / m_p)` turns each analyser step's energy centre
   into a proton speed.
3. **Flux-weighted moments**, using only the finite (above-threshold) channels at each timestamp:
   - **bulk speed** `V = Σ(flux·v) / Σ(flux)` — the 0th/1st-moment ratio, needs no absolute
     calibration since it only depends on *where* the flux is concentrated in energy;
   - **thermal spread** `w = √(Σ flux·(v−V)² / Σ flux)` — the flux-weighted RMS width, same
     calibration-free property;
   - **density proxy** `n_proxy = Σ(flux / v)` — this *does* need an absolute geometric
     factor/field-of-view that these L2 files do not carry.
4. **One calibration constant.** The density proxy is scaled by a single constant so that its
   median over the quiet pre-shock period (10 Oct, 00:00–10:00 UT) equals the canonical quiet
   solar-wind density quoted throughout this workshop's own reference material (~4 cm⁻³). This
   fixes the overall scale only — all time variation in the derived density is real data.
5. **Radial-flow simplification.** With no angular/vector moment computed, the full bulk velocity
   is assigned to `Vx` (`Vx = −V`, anti-sunward in GSE) with `Vy = Vz = 0`. This mirrors the
   flat-plane convection assumption the delay model itself already makes, and is physically
   motivated — SWIS looks sunward specifically to catch the core beam — but it **is** an
   approximation and should be reported as one.
6. **Alpha particles are not resolvable** from a single mass/charge-blind energy spectrum, so
   `alpha_density` is left `NaN` (and `.fillna(0)` in the `Pdyn` formula, exactly as the reference
   notebook's own code already allows for).

This derivation was sanity-checked against work.md's own stated expectations before being trusted:
quiet-time bulk speed came out ≈425–460 km/s (expected ≈400 km/s), the shock-driven peak reached
≈930–955 km/s (expected ≈830 km/s, same order), and the calibrated density ranged ≈1.6–44 cm⁻³
across the event (expected quiet ≈4, sheath ≈20 cm⁻³) — all consistent with a real fast-CME shock
passage, not an artifact.

**MAG required no substitution** — it is used exactly as the reference notebook reads it.

---

## 5. Notebook structure

### `ASPEX_MAG_Overview_Students_Copy.ipynb` (reference — do not edit)
Shared starting point for both workshop projects: reads and QCs all three Aditya-L1 in-situ
payloads (MAG, ASPEX-SWIS, ASPEX-STEPS), builds the common 1-minute table, and produces the
grand nine-panel overview. Project 6 reuses only its MAG/SWIS portion (Sections 1–8); ASPEX-STEPS
is not part of Project 6 and is not used here.

### `Project6_SpaceWeather_AdityaL1_Students.ipynb` (target — completed, executable)

| Section | What it does |
|---|---|
| 1–2 | Setup, constants, UTC time helpers |
| 2–5 | Read, inspect, QC, and plot MAG (identical to the reference notebook) |
| 6 | Read TH1/TH2, derive bulk proton moments (§4), QC, plot |
| 7–8 | Build the 1-minute `p` table; derive `Tp`, `Pdyn`, `beta`, `Va`, `Ma`, `Ey`; 7-panel overview |
| 9 | Shue (1998) magnetopause + Farris–Russell (1994) bow-shock standoff; `Mms→1` singularity clipped and replaced by the event median (a stated modelling choice) |
| 10 | Minute-by-minute propagation delay vs. a naive fixed-14-R_E comparison |
| 11 | Apply the shift: relabel, sort (fast-overtakes-slow), re-bin, interpolate interior gaps only |
| 12 | Storm onset from Aditya-L1's own `Pdyn` jump alone — no ground data used |
| 13 | Download/cache OMNI SYM-H/AE; trim and resample to 1 minute |
| 14 | Driver (shifted Aditya-L1) vs. response (OMNI) stacked figure, onset marked |
| 15 | Burton/O'Brien-McPherron leaky-bucket integration, started neutral, driven only by `sh["Ey"]`/`sh["Pdyn"]`; score vs. observed SYM-H |

A numerical safeguard was added in Section 15: the ring-current decay time `τ(Ey) =
2.40·exp(9.74/(4.69+Ey))` is only validated for the driving regime (`Ey ≥ 0`, the same threshold
where the injection term `Q` already cuts off). Strongly negative `Ey` (northward sheath
turbulence) was sending `τ → 0` and blowing up the explicit-Euler integration to `−∞`; the value
fed to `τ` is floored at 0, exactly as `Mms` is clipped near its own singularity in Section 9 — a
numerical fix, not a physics change.

---

## 6. How to run it

```bash
pip install numpy pandas xarray netCDF4 cdflib matplotlib jupyter nbconvert
jupyter nbconvert --to notebook --execute --ExecutePreprocessor.timeout=600 \
    --output Project6_SpaceWeather_AdityaL1_Students.ipynb \
    Project6_SpaceWeather_AdityaL1_Students.ipynb
```

Internet access is needed once, the first time Section 13 runs, to download the OMNI monthly CDF
(cached afterwards in `data/omni/`). Every other section runs fully offline. Works whether launched
from the project root or from inside `data/`.

---

## 7. Results

| Quantity | Value |
|---|---|
| Spacecraft position | X ≈ 259 R_E (≈1.65 million km sunward of Earth) |
| `|B|` range | 3.45 – 47.16 nT |
| Derived `V_p` range | 188 – 956 km/s |
| Derived `n_p` range | 0.01 – 44.2 cm⁻³ (calibrated proxy) |
| Propagation delay | 28.6 – 63.9 min (model), vs. 28.0 – 63.8 min (fixed-nose) |
| Minutes with reversed arrival order | 5.0% (fast plasma overtaking slow — physical, not a bug) |
| Storm onset at Aditya-L1 | 2024-10-10 14:54 UTC |
| Predicted arrival at Earth | 2024-10-10 15:25 UTC (30.9 min delay) |
| Observed minimum SYM-H | **−390 nT** at 2024-10-10 23:14 UTC (a *super* storm, `< −250 nT`) |
| Maximum AE | 4274 nT at 2024-10-10 15:53 UTC |
| Predicted minimum SYM-H | −309 nT at 2024-10-11 03:18 UTC |
| Correlation (prediction vs. OMNI) | **r = 0.93** |
| RMS error | **40.7 nT** |
| Timing error at minimum | **+244 min** (prediction lags the real minimum) |

**Reading the mismatch** (per work.md's own diagnostic categories): the prediction is *too
shallow* (−309 vs. −390 nT) and *late* (+244 min) — consistent with `τ(Ey)` being fitted to a
statistical ensemble of storms rather than this one, and with the derived (not instrument-
calibrated) density/velocity feeding the model.

---

## 8. Known limitations — state these in any report based on this notebook

1. **SWIS bulk parameters are derived, not measured** — flux-weighted moments from raw TH1/TH2
   spectra, not the instrument team's fitted BLK product (§4).
2. **Density is a single-constant-calibrated proxy**, not an absolute instrument measurement.
3. **Velocity is assumed purely radial** (`Vy = Vz = 0`) — a simplification, not a measured vector.
4. **Alpha-particle moments are not resolvable** from this dataset; `alpha_density`/`alpha_ratio`
   are `NaN` and `Pdyn` is proton-only.
5. **Flat-plane convection** treats solar-wind structures as planes perpendicular to the Sun–Earth
   line — real structures are tilted.
6. **`Mms→1` and `τ(Ey)<0`-regime values are clipped**, not computed — both are stated modelling
   choices, not measurements.
7. Aditya-L1 data are a *measurement*; the propagation to Earth is a *model*. Keep the two
   separate, as work.md itself insists.

---

## References

- Burton, R. K., McPherron, R. L. & Russell, C. T. (1975), *An empirical relationship between
  interplanetary conditions and Dst*, JGR 80, 4204.
- O'Brien, T. P. & McPherron, R. L. (2000), *An empirical phase space analysis of ring current
  dynamics*, JGR 105, 7707.
- Shue, J.-H. et al. (1998), *Magnetopause location under extreme solar wind conditions*, JGR 103,
  17691.
- Farris, M. H. & Russell, C. T. (1994), *Determining the standoff distance of the bow shock*, JGR
  99, 17681.
- Gonzalez, W. D. et al. (1994), *What is a geomagnetic storm?*, JGR 99, 5771.
- OMNI data and documentation: <https://omniweb.gsfc.nasa.gov/>
- ASPEX/SWIS data and documentation: <https://pradan.issdc.gov.in/al1/>, DOI:
  10.21203/rs.3.rs-5180284/v1
