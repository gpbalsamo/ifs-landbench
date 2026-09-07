# ifs-landbench

Run the [ecLand](https://www.ecmwf.int/en/research/modelling-systems/land-surface) land-surface model over flux-tower sites discovered live via the [FLUXNET Shuttle](https://github.com/fluxnet/shuttle), and score the result against the tower observations.

This extends beyond the fixed 170-site [PLUMBER2](https://essd.copernicus.org/articles/14/449/2022/) benchmark (see the sibling repo [plumber2-ecland](https://github.com/gpbalsamo/plumber2-ecland), which most of `scripts/` is forked from) into the wider, continually-growing pool now available across AmeriFlux, ICOS and TERN: **775 sites** as of the 2026-08-18 Shuttle snapshot, against PLUMBER2's 170, including the savanna, Mediterranean shrubland, Sahel and boreal/tundra biomes PLUMBER2 underrepresents.

![FLUXNET Shuttle site locations, colored by biome](shuttle_sites_map.png)

The headline result: the full 775-site chain runs end to end, scoring a median `r` of **0.79** (`Qle`) and **0.82** (`Qh`) against the towers. See [Results](#results).

---

## What this repository ships, and what you build

Read this before anything else — it decides which quick start applies to you.

| | Where | In git? |
|---|---|---|
| All scripts, namelists, site lists, site metadata | `scripts/`, `namelists/`, `reference/` | **Yes** |
| Published dashboards (metrics CSV + JSON + `index.html`) | `benchmark/dashboards/<pool>/` | **Yes** |
| CH4 observations, FLUXNET2015 schema | `flux/fluxnet-ch4/` | **Yes** (Git LFS) |
| Soil moisture/temperature observations | `soil/<group>/` | **Yes** (Git LFS) |
| Meteorological forcing | `forcing/<group>/` | Git LFS — **check your clone**, see step 2 |
| Physiography and initial conditions | `clim/<group>/` | Git LFS — **check your clone**, see step 2 |
| Observed flux (Qle/Qh/NEE) | `flux/<group>/` | Git LFS — **check your clone**, see step 2 |
| Raw ecLand output (`output/`), post-processed (`postprocessed/`) | — | No, regenerable |

`<group>` for the published run is **`shuttle-all775-era5`**. That name is also the `--experiment-name` everywhere downstream; the two must match or every site is skipped.

If `forcing/`, `clim/` and `flux/` come back empty in step 2, your clone does not carry the group's inputs and you must rebuild them — see [Building the inputs from scratch](#building-the-inputs-from-scratch). That path needs network access to the data hubs and, for physiography, an ECMWF account.

## Requirements

- An ecLand executable, built separately — see [ECMWF ecLand](https://github.com/ecmwf-ifs/ecland).
- Python 3 with `numpy`, `xarray`, `netCDF4`, `pandas`.
- Git LFS, to materialise the NetCDF inputs.
- On ECMWF Atos: `module load prgenv/intel intel/2021.4 hpcx-openmpi/2.9 netcdf4/4.9.1 python3`.

Only needed if you rebuild the inputs rather than using the shipped ones:

- The [`fluxnet-shuttle`](https://github.com/fluxnet/shuttle) CLI (`pip install fluxnet-shuttle`).
- R + [FluxnetLSM](https://github.com/aukkola/FluxnetLSM) — `scripts/install_fluxnetlsm.R` installs both.
- NCO (`ncrename`, `ncks`, `ncatted`, `nccopy`) and `unzip`.
- For physiography: an [`ecland-portal`](https://github.com/gpbalsamo/ecland-portal) checkout, ecLand's `create_forcing`, the `mars` client, and read access to `/home/rdx/data/climate` — i.e. it runs at ECMWF. `module load prgenv/intel ecmwf-toolbox/new python3/new netcdf4/new cdo/2.2.0 nco eclib/new`.

## Quick start

Six steps from a clone to a scored dashboard. Each says what to expect when it worked; if the check does not match, stop there rather than continuing.

### 1. Clone with LFS

```bash
git lfs install
git clone git@github.com:gpbalsamo/ifs-landbench.git
cd ifs-landbench
```

*Expect:* the NetCDF directories hold LFS pointer files, not yet data.

### 2. Materialise the inputs

```bash
scripts/ecland_retrieve_lfs.sh --all --group shuttle-all775-era5
```

Note `--group`: it defaults to `PLUMBER2`, which does not exist in this repository.

*Expect:* one forcing file and one flux file per site, two clim files per site.

```bash
ls forcing/shuttle-all775-era5 | wc -l    # 775  (met_insituHT_<site>_<years>.nc)
ls clim/shuttle-all775-era5    | wc -l    # 1550 (surfclim_* + surfinit_*, one pair per site)
ls flux/shuttle-all775-era5    | wc -l    # 775  (<site>_<years>_FLUXNET2015_Flux.nc)
ls soil/shuttle-all775-era5    | wc -l    # one per site reporting SWC or TS
ls flux/fluxnet-ch4            | wc -l    # 79   (CH4 observations)
```

**If the first three are empty or missing**, this clone does not carry the group's inputs. Go to [Building the inputs from scratch](#building-the-inputs-from-scratch) and come back here at step 3. The last two ship independently of them.

### 3. Run ecLand

The main path is [Running on the HPC](#running-on-the-hpc), which mirrors the inputs to `$SCRATCH`, submits one SLURM job array of interchangeable workers draining a shared site queue, and returns the results. Follow that section through and come back here for what the outputs mean.

*Expect:* one directory `output/<site>_<years>/` per finished site, holding the raw `o_*.nc` fields plus the namelist it ran with. A full 775-site run takes **1 h 13 min** wall clock and produces **711 GB**.

To run without SLURM, one site at a time:

```bash
scripts/ecland_run_experiment.sh -g shuttle-all775-era5 -t insitu \
  -x <path_to_ecland_executable>
```

### 4. Post-process

Map raw ecLand output onto the common variable schema — one file per site:

```bash
scripts/submit_postproc_slurm.sh \
  -I $SCRATCH/ecland_shuttle-all775-era5/output \
  -e shuttle-all775-era5
```

`postproc.py` is serial at ~90 s per site, so use the submitter; it splits sites across workers in one job and skips any site whose output already exists. Defaults are `-w 40 -M 3G`. Per-worker memory scales with record length and QoS `nf` caps a job at 128 GB, so lower `-w` (and raise `-M`) if a long-record group is killed for memory. Without SLURM: `python3 scripts/postproc.py --experiment-name shuttle-all775-era5 --overwrite`.

*Expect:* `postprocessed/ecLand_shuttle-all775-era5_<site>_<period>.nc`, one per site, ~1 h 51 min for 775 sites at 40 workers. Check the time axes:

```bash
python3 scripts/check_dates.py --experiment-name shuttle-all775-era5
```

### 5. Benchmark

**All six observation sources must be named explicitly.** Three of the four directory flags default to `plumber2-ecland` paths that do not exist here, and a missing directory is not an error — the variables it feeds simply report as unavailable for every site, and you get a dashboard that looks complete but silently has no SWC, TS or FCH4 panels.

```bash
BENCH=(python3 scripts/benchmark.py
       --model-dir       postprocessed
       --experiment-name shuttle-all775-era5
       --flux-dir        flux/shuttle-all775-era5
       --soil-dir        soil/shuttle-all775-era5
       --ch4-dir         flux/fluxnet-ch4
       --out-dir         benchmark/dashboards)

"${BENCH[@]}" --run-name shuttle-all775 --run-label 'all 775 sites'
"${BENCH[@]}" --run-name plumber2-170   --run-label 'PLUMBER2 subset' \
  --sites-file reference/subset_plumber2_170.txt
"${BENCH[@]}" --run-name best42         --run-label 'best-42 subset' \
  --sites-file reference/subset_best42.txt
```

*Expect:* `benchmark.py` to echo, in its first few lines, how many sites each observation source gave it:

```
Country lookup: 775 sites from reference/site_country.csv
Soil observations: <N> sites from soil/shuttle-all775-era5
CH4 observations: 79 sites from flux/fluxnet-ch4
Processing 775 site/period pairs...
```

**Those two middle lines are the check.** If either reads `none found under ...`, that variable will be blank in the dashboard and the flag above is wrong or the directory was never materialised. Each pool then writes `benchmark_metrics.csv`, `benchmark_data.json` and `index.html` into `benchmark/dashboards/<run-name>/`. Roughly 20 min per pool.

### 6. Open the dashboard

```bash
open benchmark/dashboards/shuttle-all775/index.html   # or xdg-open
```

Self-contained — the data is embedded, no server needed. See [The benchmark dashboard](#the-benchmark-dashboard) for what each panel shows and how to export figures.

---

## Running on the HPC

`$PERM` is a single NFS filer, `$SCRATCH` is Lustre: measured with 30 concurrent writers, 530 MB/s against 4863 MB/s. Reads count as much as writes, since all workers in one array element share their node's NFS client — so forcing, clim and the executable have to be on Lustre too. **Running from the `$SCRATCH` mirror is a requirement above ~25 concurrent sites, not a preference.**

Each script below is invoked by its full path, because each takes the tree it lives in as the tree it works on: the mirror is driven from `$PERM`, the run itself from the copy inside `$SCRATCH`.

**1. Mirror inputs and code to `$SCRATCH`.**

```bash
$PERM/ifs-landbench/scripts/scratch_mirror.sh push -g shuttle-all775-era5
```

`push` sends `scripts`, `namelists`, `reference`, and the `forcing`, `clim`, `flux` and `soil` for the group named by `-g`, plus `ecland-build/{bin,lib,lib64}` — the executable resolves five shared objects through an `$ORIGIN/../lib64` rpath, so it travels too. `-r` skips `flux/`, which only the benchmark reads, for a faster first push.

*Expect:* `$SCRATCH/ifs-landbench/` with this repository's layout, so every script works there unchanged.

**2. Submit the run** from the mirrored copy.

```bash
cd $SCRATCH/ifs-landbench
scripts/submit_ecland_slurm.sh -g shuttle-all775-era5 \
  -x $PWD/ecland-build/bin/ecland-master-dp
```

Defaults are `-a 5 -w 36 -l 2 -T 03:30:00 -q nf -M 2G`. `-d` prints the job script without submitting; `-h` lists every option. Add `-S reference/<list>.txt` to run a subset. Pass `-i` to make the run root the tree itself, so `output/` sits where postproc and benchmark expect it; otherwise everything goes under `<-O>/ecland_<GROUP>/`, defaulting to `$SCRATCH`.

Sites are claimed with an atomic `mkdir` and recorded individually, so the run is resumable and a failure costs one site, not a slice. Re-submitting seeds completed output as done and reclaims claims orphaned by a killed job. Retry only failures with `grep -lx FAILED <run_root>/status/* | xargs rm`.

**Concurrency is `-a` × `-w`.** Job slots are the scarce resource, not CPUs: `MaxJobs=30` per account on QoS `nf`, counted per array element, so `-a` above 30 only adds `PENDING` elements, while `-w` buys concurrency from a node's 256 CPUs. `-a 5 -w 36` gives 180 concurrent sites for 5 job slots.

**180 workers is the number worth remembering.** Anything at or above ~165 finishes in the same ~2.1 h — the cost of `NL-Loo_1997-2025` running alone, serial and unsplittable; `-w 48` measured no faster. Below that floor the only lever is `NLOOP=1` from an equilibrated restart. If you *lower* concurrency, raise `-T` to match: a worker lives for the whole drain (≈ total/N), and a limit below it kills every worker mid-queue and records nothing.

The generated job script exports `OMPI_MCA_hwloc_base_binding_policy=none`, `OMP_NUM_THREADS=1`, `KMP_AFFINITY=disabled` and an empty `LAUNCH`. All four are required; the run is ~13× slower without them.

**3. Post-process and benchmark** on the mirror — steps 4 and 5 of the quick start, run from `$SCRATCH/ifs-landbench` so `postprocessed/` stays on Lustre too.

**4. Pull the results back.** From the `$PERM` copy of the script — the mirrored one would pull `$SCRATCH` onto itself.

```bash
$PERM/ifs-landbench/scripts/scratch_mirror.sh pull
```

*Expect:* `postprocessed/` and `benchmark/{models,dashboards}` return; raw `output/` never does. Neither direction uses `--delete`. **`$SCRATCH` is pruned automatically**, so anything not pulled back is eventually gone. `scratch_mirror.sh status` shows both sides.

### What a full run costs

Measured over the 775-site group at `NLOOP=2`, `-a 5 -w 36`:

| stage | wall clock | resources |
|---|---|---|
| ecLand, 775 sites | **1 h 13 min** (0 failures) | 94.7 CPU-h for 1507 site-years, 711 GB raw output |
| `postproc.py` → one file per site | 1 h 51 min | 40 workers, 18 GB |
| `benchmark.py` → dashboard | 20 min | single process per pool |

Cost law: **194 s per site-year**, ×1.17 when 36+ workers share a node. Forcing is half-hourly at 766 of the 775 sites, so site-years is a sound predictor here. Do **not** reuse these timings for `plumber2-ecland`, whose sites cost 86 s per site-year and mix resolutions; refit per repo from `len(time)`.

---

## Results

The complete chain from tower CSV to benchmark scores, at `NLOOP=2` with tower forcing and O1280 physiography (the run predates the move to TCo2559 described under [Physiography](#physiography-and-initial-conditions)).

### The full 775-site run

Dashboard: [`benchmark/dashboards/shuttle-all775/index.html`](benchmark/dashboards/shuttle-all775/index.html)

| variable | sites scored | median r | median bias | median RMSE |
|---|---|---|---|---|
| `Qle` | 775 | **0.79** | +5.1 W m⁻² | 56.7 |
| `Qh` | 775 | **0.82** | +12.0 W m⁻² | 59.7 |
| `NEE` | 723 | 0.62 | −0.7 µmol m⁻² s⁻¹ | 12.7 |

Median `r` for `Qle` by biome: GRA 0.83 (136 sites), DBF 0.81 (69), MF 0.80 (44), CRO 0.79 (147), WET 0.79 (106), ENF 0.78 (102), EBF 0.75 (46), OSH 0.69 (41).

`NEE` is scored at 723 of 775: 50 towers report no `NEE` at all, and 2 more have no half-hour surviving the measured-only QC filter. Those show the variable as unavailable rather than being scored against absent QC flags.

**`Qh` is biased high**, by +12.0 W m⁻² at the median across the whole pool. The same sign appeared in the earlier 3-site pilot (+2.8 to +42.2 W m⁻²), so it is a property of the configuration rather than a small-sample artefact, and it survives every subset below. `NLOOP=2` may be too few spin-up loops; testing that means a run at `NLOOP=1` from an equilibrated restart, or more loops, and comparing the same metric.

### The PLUMBER2 subsets, over longer records

The same run restricted to the two PLUMBER2 site lists, so the scores line up with `plumber2-ecland`'s dashboards while using FLUXNET-Shuttle's longer records. FLUXNET-Shuttle carries only part of the PLUMBER2 pool — 110 of the 170 towers and 36 of the curated 42 — the rest being legacy sites it does not redistribute.

| pool | dashboard | towers | site-years (was, in PLUMBER2) |
|---|---|---|---|
| PLUMBER2 170 | [`plumber2-170/`](benchmark/dashboards/plumber2-170/index.html) | 110 of 170 | 1551 (800) — **1.94×** |
| curated best-42 | [`best42/`](benchmark/dashboards/best42/index.html) | 36 of 42 | 664 (408) — **1.63×** |

Nearly every retained tower gains years: 88 of 110 are longer than their PLUMBER2 period (11 the same, 11 shorter), and 29 of 36 in the best-42.

| pool | `Qle` r / bias | `Qh` r / bias | `NEE` r / bias |
|---|---|---|---|
| all 775 | 0.79 / +5.1 | 0.82 / +12.0 | 0.62 / −0.7 |
| PLUMBER2 170 | 0.80 / +2.3 | 0.86 / +12.9 | 0.65 / +0.1 |
| best-42 | 0.79 / −0.1 | **0.88** / +7.0 | **0.71** / +0.3 |

Medians; bias in W m⁻² for the heat fluxes and µmol m⁻² s⁻¹ for `NEE`. Skill rises as the pool narrows to the curated sites — `NEE` most of all, 0.62 → 0.71 — which is what those 42 were selected for, and a reminder that a headline score is only readable next to the pool it was computed on.

---

## The benchmark dashboard

`scripts/benchmark.py` scores a post-processed run against the tower observations using **only quality-controlled, non-gapfilled observation half-hours** (`qc == 0`, meaning measured). Per site it computes bias, RMSE, R and NME plus monthly-climatology, seasonal-diurnal and long-term-trend aggregates, then builds a self-contained dashboard from `scripts/dashboard_template.html`: a pannable site map, a Taylor diagram, a per-biome skill breakdown, a searchable and sortable ranked table, and a per-site drill-down.

### Six variables, from four separate observation sources

This is the part that most often produces an incomplete dashboard, because each source is optional and a missing one degrades silently:

| variable | unit | comes from | flag |
|---|---|---|---|
| `Qle`, `Qh`, `NEE` | W m⁻², W m⁻², µmol m⁻² s⁻¹ | the group's FLUXNET2015 flux files | `--flux-dir flux/<group>` |
| `FCH4` | nmol m⁻² s⁻¹ | the FLUXNET-CH4 Community Product | `--ch4-dir flux/fluxnet-ch4` |
| `SWC`, `TS` | %, °C | ancillary soil columns of the raw FLUXNET download | `--soil-dir soil/<group>` |

`--flux-dir` and `--soil-dir` default to `flux/PLUMBER2_original` and `soil/PLUMBER2_original`, neither of which exists in this repository — both must be given. `--experiment-name` must be **identical** to the one given to the post-processing, or every site is skipped as "no model output". `--ch4-dir` defaults correctly.

`FCH4`, `SWC` and `TS` each live in their own file with their own time axis, which `process_site()` intersects against the model's independently — so their periods need not match the flux period, and only some sites have them at all. A site missing one reports that variable as unavailable, exactly as a tower that does not measure `NEE` does.

### Options

`--out-dir` is a base path: results go to `<out-dir>/<run-name>/`, defaulting to `all` when neither `--run-name` nor `--sites-file` is given. `--run-label` sets the pool name shown in the dashboard header and browser tab. `--site` filters to one or more sites (repeatable). `--sites-file` restricts the run to a curated subset — one entry per line, either a bare site code (`AT-Neu`, taking whatever period this pool holds) or `SITE_period` (`AT-Neu_2002-2012`, pinning one). Bare codes are what let the PLUMBER2-era lists in `reference/subset_*.txt` select the same towers over FLUXNET-Shuttle's longer records; entries absent from the pool are reported and skipped.

It is model-agnostic: any directory of per-site NetCDF files works as `--model-dir`, whether named in the postproc convention (`ecLand_<experiment>_<site>_<period>.nc`) or a site-only one with no period in the filename (`*.{SITE}.nc`, e.g. JULES output). Keep each experiment's post-processed output under its own `benchmark/models/<model-name>/` — `ecland_cy50r1` for a control, `ecland_cy50r1_<variant>` for a namelist variant — so runs can be compared side by side.

### Exporting figures

Every panel carries `PNG` / `SVG` / `PDF` buttons: `PNG` gives a 2400 px raster (300 dpi at 20 cm wide), `SVG` a vector file for Inkscape or Illustrator, `PDF` opens the print dialog for that one figure. Each export carries its own title, the selected variable, the legend and a provenance line, and is drawn on white even when the dashboard is in dark mode. The four seasonal diurnal panels export together as one 2×2 figure.

---

## The FLUXNET-CH4 site group

ecLand writes a methane flux (`CH4flux` in `o_co2.nc`, alongside `CO2flux`), so it can be evaluated against tower CH4 — but none of the 775 shuttle-sourced flux files carry `FCH4`. The Shuttle federates one product, ONEFlux-processed FLUXNET FULLSET, and ONEFlux has no CH4 branch. CH4 observations need their own fetch path.

`flux/fluxnet-ch4/` **ships in this repository**, so the steps below are only needed to refresh it.

```bash
# 1. Discover which AmeriFlux sites publish FCH4 (public endpoint, no account).
#    Writes reference/fluxnet_ch4_sites.csv + reference/subset_fluxnet_ch4.txt.
scripts/fetch_fluxnet_ch4.py discover

# 2. Request the data. Logged against YOUR AmeriFlux account and governed by the
#    CC-BY-4.0 data policy, hence the explicit flag; --dry-run prints the body.
scripts/fetch_fluxnet_ch4.py download \
  --user-id <id> --user-email <mail> --intended-use model --accept-policy

# 3. Or ingest the FLUXNET-CH4 Community Product, downloaded from the portal.
scripts/fetch_fluxnet_ch4.py ingest --from-dir <dir>

# 4. Convert to the FLUXNET2015 NetCDF schema.
scripts/convert_fluxnet_ch4.py --from-dir raw/fluxnet-ch4 --outdir flux/fluxnet-ch4
```

To score CH4 on its own rather than as one variable of the full pool, point the benchmark's `--flux-dir` at the same directory:

```bash
python3 scripts/benchmark.py --flux-dir flux/fluxnet-ch4 \
  --model-dir postprocessed --out-dir benchmark/dashboards \
  --run-name fluxnet-ch4 --run-label 'FLUXNET-CH4' \
  --experiment-name shuttle-all775-era5
```

**Two sources, answering different questions.** AmeriFlux BASE gives the live, growing set — **118 sites publishing `FCH4`** as of 2026-08-24, against the 45–46 in the 2021/2023 papers — but as submitted by each tower team, ungapfilled and not standardised across networks. The [FLUXNET-CH4 Community Product](https://fluxnet.org/data/fluxnet-ch4-community-product/) v1.0 (Delwiche et al. 2021, [doi:10.5194/essd-13-3607-2021](https://doi.org/10.5194/essd-13-3607-2021)) is the standardised, gap-filled, citable one at 79 sites, and is what ships here. It is not served by AmeriFlux's download API — that API accepts only `BASE-BADM` and `FLUXNET` — so it arrives through the portal and `ingest`, not `download`. ORNL DAAC hosts the derived UpCH4 *gridded* product, not the tower data.

**The converter rebuilds the QC flag, and that matters.** FLUXNET-CH4's own `_QC` is not FLUXNET2015's: it flags gap *length* on the gap-filled series (1 = gap under two months, 3 = over) and says nothing about which half-hours were measured, while `benchmark.py` scores `qc == 0` meaning measured. `convert_fluxnet_ch4.py` therefore derives the flag from the raw/gap-filled variable pair — 0 where raw `FCH4` has a value, 1/3 from the product's flag where it was filled from `FCH4_F_ANNOPTLM`, 2 where filled with no gap-length flag. The gap-filled values stay in the file for anyone who wants them. FluxnetLSM cannot do this job at all: its variable table targets the ALMA/FLUXNET2015 set, which has no methane flux.

**Sign convention.** ecLand is downward-positive throughout, so `CH4flux < 0` is emission. `postproc.py` negates it into `FCH4` to match FLUXNET-CH4, where positive means emission to the atmosphere — the same rule already applied to `CO2flux` → `NEE`.

**`fwet` gates the emission, so read the physiography before the score.** The monthly wetland fraction `fwet` is a hard gate in this configuration: every one of the 19 sites whose `fwet` is zero in all 12 months emits exactly zero CH4, and no site with `fwet > 0` is silent. So the CH4 comparison is, in the first instance, a test of whether the climate fields put a wetland where the tower is. They mostly do, but faintly: of the 58 wetland-class CH4 towers, 9 have `fwet = 0` year-round, and across the rest the median peak-month `fwet` is 0.065 and the median annual mean 0.019 — a tower sited *in* a wetland sits in a ~9 km grid box that is only a few percent wetland. **Report scores split by `fwet > 0`**, and treat the zero-`fwet` sites as a physiography result rather than a CH4-scheme result. `fwet` is monthly (12 values per site): read all of them, not just the first, or the zero count comes out far too high.

Note what is and is not a modelling choice here. `landsea = 1` and `CLAKE = 0` at every site are hardcoded by `create_sites.py` under `-t land`, this repo's default (`WHICH_SURFACE=land` in `extract_physiography_batch.sh`) and the PLUMBER2 convention: tower sites are forced to be pure land points with no lake. `fwet` is **not** forced — it is the genuine nearest-gridpoint value from the `cldiff` climate field. Running `-u orig` would keep ERA5's own land/lake fractions, changing `landsea`/`CLAKE` but not `fwet`.

**One conversion is unverified.** `CH4flux` is documented as `kg m-2 s-1` with no species stated, and `FCH4_KG_TO_NMOL` in `benchmark.py` assumes kg of CH4. If it is kg of carbon, as `CO2flux` appears to be, every CH4 magnitude here is 1.34× low. It is a single constant, and it moves bias only, never correlation — but it should be settled against the ecLand source before a bias figure is published.

---

## Building the inputs from scratch

Only needed if step 2 of the quick start came back empty, or to add sites or take a newer Shuttle snapshot.

```
scripts/install_fluxnetlsm.R (once per machine)      # R + FluxnetLSM, with a documented sf/lutz workaround

fluxnet-shuttle listall                              # live site inventory -> snapshot CSV
  -> scripts/build_site_metadata.py                   # FluxnetLSM's 874-site table + Shuttle sites -> merged site CSV
  -> scripts/fetch_noaa_co2.py                        # NOAA GML monthly CO2 -> CO2 fallback table (once)
  -> scripts/filter_candidate_sites.py                # IGBP/record-length filter, exclude PLUMBER2-170 (optional)
  -> scripts/run_forcing_pipeline.sh                  # batch driver, per site:
       fluxnet-shuttle download                       #   per-site zip -> FLUXMET (+ optional ERA5) HH CSV
       -> scripts/fill_co2_from_noaa.py               #   fill missing CO2 (-C), which ERA5 cannot supply
       -> scripts/convert_fluxnetlsm.R                #   FLUXMET CSV -> ALMA-CF Met/Flux NetCDF (--preset, --gapfill)
       -> scripts/regenerate_forcing.sh               #   -> ecLand forcing convention (lon/lat/time, PSurf/Rainf)
  -> scripts/qc_classify.py                           # post-hoc: real per-variable gap-fill % -> mild/medium/heavy/complete

scripts/extract_soil_ancillary.py                    # the same downloads -> soil/<group>/ (SWC/TS observations)
scripts/extract_physiography_batch.sh                # per site, via ecland-portal + ecLand's create_forcing:
                                                     #   static fields off disk -> surfclim_<site>_<Y1>-<Y2>.nc
                                                     #   one MARS analysis      -> surfinit_<site>_<Y1>-<Y2>.nc
  -> scripts/check_physiography.py                    # refuse NaN/no-land output instead of poisoning a run
```

### 1. Inventory, site metadata, CO2 fallback

```bash
pip install fluxnet-shuttle   # or: pip install git+https://github.com/fluxnet/shuttle.git
fluxnet-shuttle listall       # -> fluxnet_shuttle_snapshot_<timestamp>.csv

Rscript scripts/install_fluxnetlsm.R   # once per machine

# Merge FluxnetLSM's bundled Site_metadata.csv with the snapshot. Required for
# any site outside the original FLUXNET2015 pool: FluxnetLSM's own table stops
# at ~874 mostly pre-2017 sites and *silently* converts unknown sites with NA
# lat/lon/IGBP. Re-run whenever you take a newer snapshot.
python3 scripts/build_site_metadata.py fluxnet_shuttle_snapshot_*.csv \
  --out reference/site_metadata_merged.csv

# CO2 fallback table. CO2 is the one met variable ERA5 gapfilling cannot supply,
# and FluxnetLSM discards any year with a residual met gap -- so without this,
# sites with intermittent CO2 are lost over a variable ecLand is not driven by.
python3 scripts/fetch_noaa_co2.py --out reference/noaa_gml_co2_monthly.csv
```

`build_site_metadata.py` applies the documented `EXCLUDE_OVERRIDES` in its source, which un-excludes sites FluxnetLSM's table blocks without a stated reason (currently just CZ-BK2, verified to convert cleanly). `--respect-upstream-excludes` honours the upstream flags instead.

To narrow the inventory to a shortlist rather than processing everything:

```bash
python3 scripts/filter_candidate_sites.py fluxnet_shuttle_snapshot_*.csv \
  --igbp SAV WSA OSH CSH GRA \
  --exclude-file reference/plumber2_170_site_ids.txt \
  --min-years 5 --top 20 \
  --out reference/shuttle_pilot20_candidates.csv
```

`--out` always writes a **CSV** (site_id, hub, coordinates, IGBP, record years, download link), not a site-ID list; the ranked table also goes to stdout.

### 2. Forcing and observed flux

`scripts/run_forcing_pipeline.sh` drives download → FluxnetLSM conversion → forcing adaptation for every site in a list, writing `forcing/<group>/met_insituHT_<site>_<years>.nc` and the matching observed flux under `flux/<group>/`. On HPC, `scripts/submit_forcing_pipeline_slurm.sh` submits the same pipeline as a job array — the site list is split across `-a` array tasks each running `-j` workers, so concurrent sites = `-a` × `-j`:

```bash
# One-off setup
module load R/4.5.3
R_LIBS_USER=$PERM/R/library/4.5 Rscript scripts/install_fluxnetlsm.R
python3 -m venv $PERM/venv-shuttle
$PERM/venv-shuttle/bin/pip install git+https://github.com/fluxnet/shuttle.git netCDF4

scripts/submit_forcing_pipeline_slurm.sh \
  -f $SCRATCH/fluxnet_shuttle_snapshot_*.csv \
  -g shuttle-all775-era5 \
  -c reference/site_metadata_merged.csv \
  -C reference/noaa_gml_co2_monthly.csv \
  -P complete -G erainterim -a 8 -j 4
```

That is the exact configuration behind the 775-site result: ~47 minutes wall clock at 8×4, plus a short second pass for the CO2-recovered sites. Add `-n` to print the job script without submitting.

Key behaviours:

- **Disk-safe.** One site at a time per worker, each site's raw download (up to ~500 MB) deleted as soon as it is consumed — all 775 zips at once would need >100 GB. Final outputs are a few MB per site.
- **Resumable.** Every finished site writes `scripts/work/forcing_pipeline_<group>/status/<site>` (`OK`, `NODOWNLOAD`, `BADZIP`, `NOFLUXMET`, `NOERA5`, `NOYEARS`, `NOMET`, `ADAPT_FAILED`, `ERROR`) and is skipped on re-invocation. Transient failures are told apart from genuine no-data verdicts, so retry a network blip by deleting just those status files and re-running the same command:

  ```bash
  grep -lxE 'NODOWNLOAD|BADZIP' scripts/work/forcing_pipeline_<group>/status/* | xargs rm
  ```

  Slices are disjoint and the status directory is shared, so an array re-submitted with the same `-g` skips every site already recorded, no matter which task did it. Per-site logs are in `.../logs/<site>.log`.
- **`-P` acceptance preset** — `mild | medium` (default) `| heavy | complete`, a bundle of FluxnetLSM's own thresholds (`gapfill_met_tier1`, `missing_flux`, `min_yrs`, `check_range_action`). For a fixed gapfill method, a looser preset yields a strict superset of the periods a stricter one yields. `heavy`/`complete` switch `check_range_action` to `truncate`, which is what keeps a single implausible value from discarding a whole record.
- **`-G` gapfilling** — `statistical` (default) or `erainterim`, which also extracts the site's `*_ERA5_HH_*.csv` from the same download. Flux variables always gapfill statistically; FluxnetLSM supports nothing else for them. **This choice dominates yield**: 231 sites vs 775 on the same pool and preset.
- **`-C` CO2 fill** — fills missing CO2 from the NOAA table before conversion, flagged QC=3 so it still counts as gap-filled. Without it, 97 sites produce nothing at all.
- A site can yield **several disjoint qualifying periods** (e.g. 2009 and 2011–2013 separately); every one is written, not just the longest.
- **Work directory defaults to `$SCRATCH`** (`-W` to move it). Each worker stages a ~500 MB download, so this wants a fast, roomy filesystem — not the repo's.
- **The batch environment is not the login environment.** `SBATCH_EXPORT=NONE` on ECMWF, so the job script loads R and NCO itself and prepends the venv to `PATH`; override with `R_MODULE`, `NCO_MODULE`, `R_LIBS_DIR`, `SHUTTLE_VENV`. A preflight check fails the task in seconds if `Rscript`, `fluxnet-shuttle`, `ncks` or `unzip` is missing.
- **Compute nodes need outbound HTTPS**, since every task downloads from ICOS/AmeriFlux/TERN. They have it at ECMWF; elsewhere the download step may have to run where the network is.
- **Concurrency is a courtesy question.** The `-a 4 -j 4` default is 16 simultaneous downloads from the data hubs. Raise it knowingly.

For a single site, to debug or to inspect FluxnetLSM's intermediate output:

```bash
fluxnet-shuttle download -f fluxnet_shuttle_snapshot_*.csv -s ES-LJu -o downloads/

Rscript scripts/convert_fluxnetlsm.R --site=ES-LJu \
  --infile=downloads/ES-LJu/EUF_ES-LJu_FLUXNET_FLUXMET_HH_*.csv \
  --outdir=fluxnetlsm_out --site-csv=reference/site_metadata_merged.csv

ORIG_DIR=fluxnetlsm_out/Nc_files/Met OUT_DIR=forcing/<group> \
  scripts/regenerate_forcing.sh
```

### 3. Soil moisture and temperature observations

The FLUXNET FULLSET's `SWC_F_MDS_<depth>` / `TS_F_MDS_<depth>` columns never reach `forcing/` or `flux/`: `convert_fluxnetlsm.R` targets FluxnetLSM's ALMA-CF variable set, which has no soil-state slot, so they are silently dropped. `scripts/extract_soil_ancillary.py` is a separate minimal path from the same per-site download:

```bash
python3 scripts/extract_soil_ancillary.py \
  -f fluxnet_shuttle_snapshot_*.csv \
  -g shuttle-all775-era5 \
  -j 4
```

*Expect:* `soil/<group>/soil_<site>_<Y1>-<Y2>.nc` for each site that has either variable, plus `reference/soil_coverage_<group>.csv` recording, per site, how many depths exist and how complete each is. Omit `-S` to process every site in the snapshot; resumable, a site already in the coverage CSV is skipped.

It re-downloads each site's zip (deleting it the moment that site is scored), so budget the same network time as the forcing pipeline. It deliberately does no gapfilling, no acceptance thresholds and no unit conversion — raw values with their own QC flags, so filtering is a decision made against the output rather than a rerun.

**Depths are index order only** (`SWC_F_MDS_1`, `_2`, …). FLUXNET2015 does not put depth-in-cm in the FLUXMET file — that lives in the per-site BADM/BIF metadata, which this script does not parse. `benchmark.py` therefore scores **index 1 only**, the shallowest sensor, against ecLand's level 1 (0–7 cm): the one depth that can be matched without guessing.

### 4. Physiography and initial conditions

ecLand needs more than weather: the fixed description of each place — soil type, vegetation cover and type, orography, lake and ice masks, LAI and albedo — plus an initial state for its prognostic variables. `scripts/extract_physiography_batch.sh` produces both, for every site that already has forcing in the group:

```bash
scripts/extract_physiography_batch.sh -g shuttle-all775-era5 -j 8

# or as a batch job (~2 h for 775 sites)
scripts/submit_physiography_slurm.sh -g shuttle-all775-era5 -j 8
```

*Expect:* `clim/<group>/surfclim_<site>_<Y1>-<Y2>.nc` and `surfinit_<site>_<Y1>-<Y2>.nc`.

- **Every input is derived from the forcing files themselves** — coordinates from each file's own `latitude`/`longitude`, the period from its filename. `ecland_run_model.sh` pairs clim and met files *by name*, so this removes any chance of running a site with physiography from a different place or period than its weather.
- **The work is done by [`ecland-portal`](https://github.com/gpbalsamo/ecland-portal)**, whose `hpc_scripts/extract_physiography.sh` drives ecLand's own `create_forcing`. This script calls it rather than forking it; point `-E`, or `ECLAND_PORTAL_DIR`, at your checkout. It is fast because the expensive part of `create_forcing` is one we don't need: the static fields are a disk copy from `/home/rdx/data/climate/climate.<version>/<grid>`, and only the initial-conditions analysis goes to MARS — 60–90 s per site, against the hour-per-month that retrieving *meteorological* forcing would cost.
- **`-s auto` (the default) picks where each part comes from.** Static fields at the grid `-r` selects — **TCo2559 (~4.5 km) by default**, from `climate.v021/2559_4`. The initial-conditions analysis comes from the **operational** archive for sites starting on or after 2015-06-01 and from **ERA5** before that: FLake entered the operational model in May 2015, so `marsod` holds none of the `8.228–14.228` lake fields before it, and `create_forcing` asks for all 26 analysis parameters or fails. Across the 775 sites the split is 388 operational / 387 ERA5. `surfclim` is *identical* whichever route a site takes — both read the same climate directory — so the physiography is uniform and only `surfinit` differs.
- **Resolution matters in terrain, and TCo2559 is the production choice.** Over the 300 sites with a published elevation, O1280 → TCo2559 improves the median elevation error from 28.5 m to **20.7 m**, the mean from 75.5 m to **58.1 m**, and the count off by more than 200 m from 27 to 20. Mountain sites gain most: ES-LJu 338 m → 211 m, CZ-BK2 165 m → 87 m, SE-Sto (Stordalen) 386 m → **27 m**. N640 (~18 km) is markedly worse again (ES-LJu 774 m). Finer grids also rescue coastal points: CA-RBM has no land at its nearest 18 km gridpoint, and `create_forcing` exits 0 while writing **NaN for every field**.
- **TCo2559 needs two things O1280 did not.** Metview reads each static field whole and its GRIB buffer defaults to 64 MiB, while one monthly albedo field is ~128 MB — the extraction segfaults unless `MARS_READANY_BUFFER_SIZE` is raised (the batch script exports 2 GiB). And `create_sites.py` is OOM-killed on a login node at these field sizes; it needs a batch job. **TCo3999 (~2.5 km) still fails inside `create_sites.py`** with neither memory (9.4 GB peak of 120 GB) nor the buffer to blame — a fix belongs in `create_forcing`.
- **`check_physiography.py` refuses a NaN/no-land file** rather than copying it into `clim/`, marking the site `NOLAND`. Nothing downstream would otherwise notice: the filenames and dimensions are right, so the file would be fed to the model and produce nonsense instead of an error.
- **For those sites, take the physiography from the nearest land gridpoint.** 18 of the 775 have no land even at TCo2559 — Arctic coastal tundra around Barrow and Oliktok, the Turkey Point cluster on Lake Erie, Stordalen, Andøya, two salt marshes, Pond Inlet:

  ```bash
  python3 scripts/nearest_land_point.py --sites-csv noland.csv \
    --out reference/physiography_land_nudge.csv
  scripts/extract_physiography_batch.sh -g <group> -S <those sites> \
    -O reference/physiography_land_nudge.csv
  ```

  Offsets are 1.2–4.2 km at TCo2559. **The table is grid-specific: recompute it against the grid you will actually use.** A table built for another grid can be worse than no nudge — reusing the O1280 table at TCo2559 sent Stordalen 7.3 km up a mountain, a 698 m elevation error where the matching table gives 27 m. The tower keeps its own coordinates for everything else — the forcing is its own measurements — and only the static fields *and the initial soil state* are borrowed, since a sea gridpoint has no soil to initialise from either. Every substitution is recorded with its distance. Still treat these sites as lower confidence.
- Costs one MARS analysis request per site, so `-j` is a courtesy limit toward MARS as much as a throughput setting.

### 5. Check how gap-filled the result actually is

The preset decides what gets written; it says nothing about how much of a written period is real observation. FluxnetLSM records the true per-variable `Missing_%`/`Gap-filled_%`/`Gapfilling_method` in every file, and these survive into the final forcing, so this is a post-hoc filter — no reprocessing:

```bash
python3 scripts/qc_classify.py forcing/<group> --out qc_report.csv
```

Each (file, variable) is bucketed into the same `mild`/`medium`/`heavy`/`complete` bands as the acceptance presets, letting you keep a permissively-admitted period while still knowing to distrust it. For the published run, `reference/qc_report_shuttle-all775-era5.csv`: across all 7750 (file, variable) records, mild 3755, medium 1603, heavy 1169, complete 1223. Roughly half the delivered data is lightly gap-filled; the heavily-filled remainder is labelled per file and variable rather than mixed in silently.

### 6. Plot the sites

```bash
python3 scripts/plot_sites_map.py --snapshot-csv fluxnet_shuttle_snapshot_*.csv \
  --output sites_map.png
```

---

## What the expanded site pool taught us

Not bugs — they shape how the pipeline is run, and they are the reason the presets exist rather than one fixed threshold set.

- **Gapfill method dominates yield, far more than the preset does.** The same 775 sites and the same `complete` preset gave 231 sites / 505 files under statistical gapfilling but **775 / 775** under ERA5. Statistical filling cannot close long gaps, and FluxnetLSM's `missing_met` threshold (default 0) then discards the whole year.
- **CO2 was silently costing 97 sites.** `missing_met=0` drops a year if *any* met variable still has a gap, and CO2 is the one met variable the Shuttle's ERA5 file cannot fill (`ERAinterim_variable=NA` in FluxnetLSM's schema — reanalysis surface files carry no atmospheric CO2). Those sites were discarded over a variable ecLand is not driven by here (`LEAIRCO2COUP=.FALSE.`; CO2 appears only as model output). The NOAA fill recovers all 97 sites and 264 site-years without touching any threshold.
- **PLUMBER2-style QC screening is strict against sites outside the original pool.** ES-LJu's real 21-year record yielded only 2 usable years under statistical gapfilling. Expect similar attrition elsewhere.
- **FluxnetLSM's default `check_range_action="stop"` discards a site's entire multi-year record** over a single implausible value anywhere in it; this alone killed SN-Dhr and US-ICt over one bad PA/VPD value from an ERA5 extraction edge case. The `heavy`/`complete` presets use `truncate`.
- **One site is blocked by upstream metadata, not by data.** FluxnetLSM's packaged `Site_metadata.csv` marks CZ-BK2 `Exclude=TRUE` with `Exclude_reason=NA`, aborting conversion before any data is read. It converts to a complete 7-year record; `build_site_metadata.py` carries a documented, reversible override.
- **Acceptance thresholds only decide what gets written out** — gapfilling always runs first regardless — so the real per-variable gap-fill fraction is recorded in every output file and read back by `qc_classify.py`. How much you trust a period is a filtering decision made *after* processing.

For the record, what the 775-site forcing run delivered (2026-08-18 snapshot, ERA5 gapfilling, `complete` preset, NOAA CO2 fill): **775 / 775 sites, 5397 site-years**, median 5 years per site, max 31 — AmeriFlux 381 sites / 2513 site-years, ICOS 342 / 2511, TERN 52 / 373. By IGBP: GRA 145, CRO 138, WET 115, ENF 114, DBF 80, EBF 44, OSH 40, MF 24, WSA 18, SAV 14, DNF 13, CSH 12, CVM 9, BSV 7, SNO 2. The fire/vegetation-stress classes that motivate this repo (SAV/WSA/OSH/CSH/GRA) come to **228 sites**, against PLUMBER2's 170 in total.

---

## Repository layout

```
ifs-landbench/
├── clim/<group>/            # Physiography + initial conditions (NetCDF, Git LFS)
├── forcing/<group>/         # Meteorological forcing, ecLand-ready (NetCDF, Git LFS)
├── flux/<group>/            # Observed Qle/Qh/NEE, FLUXNET2015 schema (NetCDF, Git LFS)
├── flux/fluxnet-ch4/        # Observed FCH4, same schema (NetCDF, Git LFS)
├── soil/<group>/            # Observed SWC/TS, ancillary (NetCDF, Git LFS)
├── config/
│   └── physiography_defaults.yaml   # ecland-portal defaults + the era5_o1280 source
├── docs/                    # Generated reports (Word summary of the procedure and QC)
├── namelists/               # ecLand namelist configurations
├── reference/               # Site lists, site metadata, coverage and QC reports
├── scripts/                 # Everything below
├── output/                  # Raw model output — not in git
├── postprocessed/           # Post-processed output — not in git
└── benchmark/
    ├── models/<model-name>/     # Post-processed output per experiment — not in git
    └── dashboards/<pool>/       # Metrics, JSON and dashboard per pool (checked in)
        ├── shuttle-all775/      # All 775 sites
        ├── plumber2-170/        # The PLUMBER2 pool, over longer records
        ├── best42/              # The 42 curated benchmark sites
        └── fluxnet-ch4/         # The CH4 towers
```

The shell scripts resolve the repository root from their own location, so they run from any directory; the Python ones default to paths relative to the working directory, so run them from the repository root (or from the `$SCRATCH` mirror, which has the same layout).

| Script | Does |
|---|---|
| `ecland_retrieve_lfs.sh` | Materialise forcing and clim from Git LFS (`--group` is required here) |
| `submit_ecland_slurm.sh` | Run a whole group as one SLURM job array (the fast path) |
| `ecland_run_queue.sh` | One worker draining the shared site queue, claims via `mkdir` |
| `ecland_run_experiment.sh` | Run sites directly, without SLURM |
| `run_parallel_local.sh` | Run sites concurrently on a local Mac (`-g GROUP` required) |
| `scratch_mirror.sh` | `push` / `pull` / `status` between `$PERM` and `$SCRATCH` |
| `postproc.py` | Raw ecLand output → common variable schema |
| `submit_postproc_slurm.sh` | The same, split across a node's CPUs |
| `check_dates.py` | Check the post-processed time axes |
| `benchmark.py` | Score against observations, build the dashboard |
| `fill_site_country.py` | Build `reference/site_country.csv` from site coordinates |
| **Input generation** | |
| `install_fluxnetlsm.R` | Install FluxnetLSM (documents the sf/lutz build workaround) |
| `build_site_metadata.py` | Merge FluxnetLSM's site table with a Shuttle snapshot |
| `fetch_noaa_co2.py`, `fill_co2_from_noaa.py` | NOAA GML CO2 table, and filling a FLUXMET CSV from it |
| `filter_candidate_sites.py` | Filter a Shuttle snapshot to candidates |
| `run_forcing_pipeline.sh` | Batch Shuttle → forcing driver (streaming, resumable, parallel) |
| `submit_forcing_pipeline_slurm.sh` | The same as a SLURM job array |
| `convert_fluxnetlsm.R` | FLUXMET CSV → ALMA-CF NetCDF, with the acceptance presets |
| `regenerate_forcing.sh` | ALMA-CF → ecLand forcing convention |
| `qc_classify.py` | Classify written forcing by real per-variable gap-fill % |
| `extract_soil_ancillary.py` | SWC/TS observations → `soil/<group>/` |
| `extract_physiography_batch.sh` | surfclim/surfinit for a whole group, via ecland-portal |
| `submit_physiography_slurm.sh` | The same as a SLURM job |
| `check_physiography.py` | Reject a NaN/no-land surfclim before it reaches a run |
| `nearest_land_point.py` | Nearest land gridpoint, for sites whose own has none |
| `fetch_fluxnet_ch4.py`, `convert_fluxnet_ch4.py` | FLUXNET-CH4 discovery/download/ingest, and conversion |
| `plot_sites_map.py` | Render `shuttle_sites_map.png` |
| `make_forcing_report.py` | Regenerate the `docs/` Word summary from the QC report |

Site lists in `reference/`: `shuttle_pilot20_site_ids.txt` (the 20-site fire/vegetation-stress shortlist), `plumber2_170_site_ids.txt` (the original PLUMBER2 pool, as an exclude-list), `subset_plumber2_170.txt` and `subset_best42.txt` (the same PLUMBER2 pools as bare site codes, for `benchmark.py --sites-file`), `subset_fluxnet_ch4.txt` (AmeriFlux sites publishing `FCH4`).

## Namelist

- `namelists/namelist_ecland_50R1_ctl` — the ecLand 50R1 control, the default used by `submit_ecland_slurm.sh` and `ecland_run_experiment.sh` when `-n` is omitted.

Name new variants `namelist_ecland_50R1_<variant>` and pair each with a matching `benchmark/models/ecland_cy50r1_<variant>/` output directory, so runs stay easy to tell apart.

## License

Scripts forked from `plumber2-ecland`: Copyright 2023– ECMWF, licensed under the [Apache Licence Version 2.0](http://www.apache.org/licenses/LICENSE-2.0). New scripts in this repo follow the same license.
