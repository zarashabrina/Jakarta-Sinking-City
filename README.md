# Land Subsidence Monitoring for DKI Jakarta

InSAR time series analysis of land subsidence in DKI Jakarta (2016-2024) using Sentinel-1 data processed with LiCSBAS2. Validated against GNSS ground truth at the CKJT station, with kriging interpolation for spatial mapping of annual subsidence velocities.

![LiCSBAS step 15 mask_ts output showing velocity, standard deviation, and masking thresholds](img/mask_ts_masked.png)

## Datasets

| Dataset | Source | Description |
|---------|--------|-------------|
| Sentinel-1 InSAR | [COMET-LiCS](https://comet.nerc.ac.uk/COMET-LiCS-portal/) | Unwrapped interferograms and coherence, frame `098A_09673_121312` |
| GNSS | CKJT station (`.rneu`) | Daily vertical position time series since 2010 |
| AOI Boundary | Administrative shapefile | DKI Jakarta boundary for clipping |
| GACOS | [Generic Atmospheric Correction](http://www.intfrg.com/gacos/) | Tropospheric delay maps for atmospheric correction |

## Pipeline

The project follows a 4-stage pipeline. The kriging step at stage 4 is performed manually in QGIS.

```
1. scripts/LiCSBAS_gacosn.sh     -> cumulative displacement (H5) + velocity (GeoTIFF)
   scripts/LiCSBAS_gacosy.sh        (GACOS-on variant)
2. scripts/annual_and_ts.py      -> LiCSBAS utility commands for time series + annual velocities
3. scripts/gnss.py               -> GNSS vs InSAR validation plots + error metrics
4. scripts/post_processing.py    -> kriging maps + visualization
        |
        +-- [MANUAL] QGIS kriging interpolation between raster-to-points
            export and CSV-to-GeoTIFF conversion
```

## Setup

### LiCSBAS2

Full installation of LiCSBAS2 is described in its [GitHub documentation](https://github.com/yumorishita/LiCSBAS2/wiki/1_Installation). 

### Python scripts

The custom analysis scripts (`gnss.py`, `post_processing.py`) run in a separate conda environment from LiCSBAS2 to prevent dependencies contamination and ensure the pipeline will run smoothly:

```bash
conda env create -f environment.yml
conda activate landsub
```

### Input data

1. Place the DKI Jakarta AOI shapefile at `data/raw/adm/adm_dki-jakarta.shp`
2. Place the GNSS `.rneu` file at `data/raw/CJKT_clean_Noff1site.rneu`
3. Sentinel-1 data is downloaded automatically in step 01 of the LiCSBAS pipeline

## Usage

### Step 1: LiCSBAS2 processing pipeline (Running in terminal)
**ATTENTION**
- Using [Macbook Pro M1 Pro](https://support.apple.com/en-us/111901), with Jakarta province as our area of interest, [step 13](https://github.com/yumorishita/LiCSBAS2/wiki/2_3_step11_16#step-1-3-small-baseline-inversion) takes ~13 hours to run, while other steps take only a couple seconds each.
- Steps 01-05 will download massive amount of data so please make sure you have sufficient and stable internet connection throughout the process.

Under `data/raw/`, create a directory for each scenario (e.g. `2016_2024_gacosn_coh0.6` and `2016_2024_gacosy_coh0.6`), then `cd` into the chosen directory and follow the [LiCSBAS workflow](https://github.com/yumorishita/LiCSBAS2/wiki/2_0_workflow) using the corresponding bash script:

- **GACOS off:** `bash ../../scripts/LiCSBAS_gacosn.sh`
- **GACOS on:** `bash ../../scripts/LiCSBAS_gacosy.sh`

The pipeline runs 16 steps in sequence:
- **[Steps 01-05](https://github.com/yumorishita/LiCSBAS2/wiki/2_2_step01_5)** (data preparation): download geotiffs, multilook, GACOS correction, masking, clipping to AOI
- **[Steps 11-16](https://github.com/yumorishita/LiCSBAS2/wiki/2_3_step11_16)** (time series analysis): unwrap checking, loop closure, small baseline inversion, velocity estimation, masking, spatio-temporal filtering

**Key Outputs:**
- `TS_GEOCml1clip/cum_filt.h5` (GACOS off) or `TS_GEOCml1GACOSclip/cum_filt.h5` (GACOS on) -- cumulative displacement time series
- `TS_*/results/vel.filt.mskd` -- filtered velocity field
- `TS_*/results/vel.filt.mskd.geo.tif` -- velocity GeoTIFF

### Step 2: Time series extraction and annual velocity (Running in terminal)

```bash
python scripts/annual_and_ts.py
```

Prints LiCSBAS utility commands to be copy-pasted into a terminal with the `licsbas2` conda environment active:

1. **LiCSBAS_cum2tstxt.py** -- extract pixel displacement time series at 4 locations (CKJT proxy, Western, Northwestern, Northeastern regions)
2. **LiCSBAS_cum2vel.py** -- compute annual velocity fields for 2016-2024
3. **LiCSBAS_flt2geotiff.py** -- convert float velocity files to GeoTIFF
4. **Rename** -- rename output GeoTIFFs to `landsub_{year}.tif`

### Step 3: GNSS validation (Running in terminal)

```bash
python scripts/gnss.py
```

Compares InSAR LOS displacement against GNSS vertical displacement at the CKJT station across 8 analysis sections:

1. Generate sample points (subsidence hotspot locations)
2. Subsidence trends in hotspot regions (linear and quadratic fits)
3. Read and filter GNSS + LiCSBAS data (uncertainty-based filtering)
4. Linear fit of GNSS and LiCSBAS measurements (velocity comparison)
5. Per-epoch comparison scatter plot (with regression and RMSE)
6. Annual and overall RMSE heatmap (2016-2021)
7. Unfiltered vs filtered GNSS measurements
8. Histogram of displacement distributions

**Inputs:**
- `data/raw/CJKT_clean_Noff1site.rneu` -- GNSS position time series
- `data/raw/2016_2024_gacosn_coh0.6/098A_09673_121312/ts_*.txt` -- LiCSBAS displacement time series

**Outputs (in `data/gnss/`):**
- `landsub_sample_points.parquet` -- sample point locations
- `los_velocity_hotspots.png` -- hotspot subsidence trends
- `gnss_vs_licsbas_linear.png` -- velocity comparison
- `gnss_vs_licsbas_scatter.png` -- per-epoch scatter plot
- `rmse_heatmap.png` -- annual RMSE heatmap
- `gnss_filtered.png` -- filtered GNSS measurements
- `gnss_vs_licsbas_hist.png` -- displacement distributions

### Step 4: Post-processing and kriging (Running back and forth in terminal and QGIS)

```bash
python scripts/post_processing.py
```

Runs 4 sections with a manual QGIS kriging step in between:

1. **GACOS analysis** -- evaluates atmospheric correction reduction rates
2. **Raster to points** -- converts annual velocity rasters to point shapefiles for kriging input
3. **(Manual) QGIS kriging** -- open point shapefiles in QGIS, run kriging, export CSVs with columns `CoordX_SM`, `CoordY_SM`, `VALUE`, `SD_Values` to `data/raw/qgis_kriging/`
4. **Kriging CSV to raster** -- converts kriging CSVs back to clipped GeoTIFFs (value + SD bands)
5. **Visualization** -- histograms and spatial maps for both raw and interpolated velocities

**Outputs (in `data/post_processing/`):**
- `gacos_reduction_hist.png` -- GACOS reduction rate distribution
- `raw_hist.png` / `interp_hist.png` -- velocity histograms
- `raw_maps.png` / `interp_maps.png` -- spatial subsidence maps

## Project Structure

```
landsub/
├── LiCSBAS2/                        # InSAR time series analysis library
│   ├── bin/                          # 36+ processing scripts (LiCSBAS01-16)
│   ├── LiCSBAS_lib/                  # shared Python modules
│   ├── LiCSBAS.yml                   # conda environment
│   └── bashrc_LiCSBAS.sh            # PATH/PYTHONPATH setup
├── scripts/
│   ├── LiCSBAS_gacosn.sh            # LiCSBAS pipeline (GACOS off)
│   ├── LiCSBAS_gacosy.sh            # LiCSBAS pipeline (GACOS on)
│   ├── annual_and_ts.py             # LiCSBAS utility command generation
│   ├── gnss.py                      # GNSS validation analysis
│   └── post_processing.py           # post-processing and kriging workflow
├── data/
│   ├── raw/
│   │   ├── 2016_2024_gacosn_coh0.6/ # GACOS-off LiCSBAS outputs
│   │   │   └── 098A_09673_121312/
│   │   ├── 2016_2024_gacosy_coh0.6/ # GACOS-on LiCSBAS outputs
│   │   │   └── 098A_09673_121312/
│   │   ├── adm/                     # DKI Jakarta AOI shapefile (user-provided)
│   │   ├── annual_displacement/     # annual velocity rasters (generated)
│   │   ├── qgis_kriging/            # kriging CSV outputs (manual)
│   │   └── CJKT_clean_Noff1site.rneu
│   ├── gnss/                        # GNSS validation outputs
│   └── post_processing/             # post-processing outputs
├── environment.yml                  # conda environment for custom scripts
└── README.md
```

## Citations

Morishita, Y.; Lazecky, M.; Wright, T.J.; Weiss, J.R.; Elliott, J.R.; Hooper, A. LiCSBAS: An Open-Source InSAR Time Series Analysis Package Integrated with the LiCSAR Automated Sentinel-1 InSAR Processor. *Remote Sens.* **2020**, *12*, 424, https://doi.org/10.3390/RS12030424.

Lazecky, M.; Spaans, K.; Gonzalez, P.J.; Maghsoudi, Y.; Morishita, Y.; Albino, F.; Elliott, J.; Greenall, N.; Hatton, E.; Hooper, A.; Juncu, D.; McDougall, A.; Walters, R.J.; Watson, C.S.; Weiss, J.R.; Wright, T.J. LiCSAR: An Automatic InSAR Tool for Measuring and Monitoring Tectonic and Volcanic Activity. *Remote Sens.* **2020**, *12*, 2430, https://doi.org/10.3390/rs12152430.
