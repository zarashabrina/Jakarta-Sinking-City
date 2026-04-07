# Proportion of Residential Area Analysis

This project analyzes residential area proportions and typologies in Jakarta using geospatial data and machine learning classification. The analysis combines building footprints, heights, morphology, and vegetation indices to classify blocks (RW) into four residential types: **High-rise**, **Regular**, **Urban Village**, and **Rural**.

---

## Project Structure

```
06 Proportion of Residential Area/
├── input/                          # Input data files (place your data here)
│   ├── adm_dki-jakarta_kelurahan.shp   # Neighbourhood boundaries
│   ├── Administrasi RW Jakarta.shp     # Block unit boundaries  
│   ├── residential_area.shp            # Building footprints
│   ├── building_height_2023.tif        # Building heights raster
│   └── NDVI_2023.tif                   # NDVI vegetation index
├── output/                         # Analysis results (auto-created)
│   ├── residential_type.shp            # Block-level classifications
│   └── buildings.shp                   # Buildings with all features
├── scripts/                        # Analysis scripts
│   └── proportion_of_residential_area.ipynb
├── README.md                       # This file
└── environment.yml                 # Conda environment specification
```

---

## Methodology

The analysis follows a hierarchical pipeline:

### 1. **Data Loading**
   - Load administrative boundaries (Kelurahan & RW/block units)
   - Load building footprints
   - Perform spatial join to assign buildings to blocks

### 2. **Feature Extraction**
   - **Building Heights**: Sample raster values at building centroids → estimate floors
   - **NDVI Values**: Extract vegetation indices at building locations

### 3. **Feature Engineering** (K-Means Clustering)
   - **Height Classification**: Low vs. Middle-to-High rise (>14 floors threshold)
   - **Shape Regularity**: Analyze building orientation variance, compactness, inter-building distance, and block shape index
   - **Building Density**: Calculate area and count densities per block
   - **Green Ratio**: Classify vegetation levels from NDVI

### 4. **Residential Type Classification** (Hierarchical Rules)
   ```
   IF height = middle-to-high → High-rise
   ELSE IF shape = regular → Regular
   ELSE IF density = low-to-middle AND green = high → Rural
   ELSE IF density = low-to-middle → Urban Village
   ELSE IF green = high → Rural
   ELSE → Urban Village
   ```

### 5. **Visualization & Export**
   - Generate map of residential typologies
   - Export results to shapefiles

---

## Installation

### Prerequisites
- **Anaconda** or **Miniconda** ([Download here](https://docs.conda.io/en/latest/miniconda.html))

### Setup Environment

1. **Clone or download this repository**

2. **Create the conda environment:**
   ```bash
   conda env create -f environment.yml
   ```

3. **Activate the environment:**
   ```bash
   conda activate sinking-city-dataset
   ```

4. **Verify installation:**
   ```bash
   conda list
   ```

---

## Input Data Requirements

Place the following 5 files in the `input/` directory:

| File | Format | Description | Source |
|------|--------|-------------|--------|
| `adm_dki-jakarta_kelurahan.shp` | Shapefile | Neighbourhood (Kelurahan) boundaries | Jakarta Open Data / Dataset 01 |
| `Administrasi RW Jakarta.shp` | Shapefile | Block unit (RW) boundaries | Jakarta Open Data / Dataset 01 |
| `residential_area.shp` | Shapefile | Building footprints | OSM/GSV / Dataset 05 |
| `building_height_2023.tif` | GeoTIFF | Building heights (2023) | [Google Open Buildings 2.5D](https://sites.research.google/gr/open-buildings/temporal/) |
| `NDVI_2023.tif` | GeoTIFF | NDVI vegetation index | Landsat / Dataset 15 |

**Coordinate System:** All data should be in **EPSG:4326 (WGS84)** or will be auto-converted to **EPSG:32748 (UTM Zone 48S)**.

---

## Usage

### Running the Analysis

1. **Ensure input data is in place** (see above)

2. **Start Jupyter Notebook:**
   ```bash
   conda activate sinking-city-dataset
   jupyter notebook
   ```

3. **Open the notebook:**
   - Navigate to `scripts/proportion_of_residential_area.ipynb`

4. **Run all cells** (Cell → Run All) or execute sequentially

5. **Check outputs:**
   - Results saved to `output/residential_type.shp` and `output/buildings.shp`
   - Visualization displayed in notebook

### Expected Runtime
- **Small datasets** (<10k buildings): ~5-10 minutes
- **Full Jakarta dataset** (~500k buildings): ~30-60 minutes

---

## Output Files

### `output/residential_type.shp`
Block-level classification with columns:
- `block`: Block identifier (format: "KELURAHAN RW XX")
- `bld_height`: Height class (low / middle-to-high)
- `bld_shape`: Shape regularity (regular / irregular)
- `bld_density`: Density class (low-to-middle / middle-to-high)
- `green_ratio`: Vegetation class (high / low)
- `res_type`: **Final residential type** (High-rise / Regular / Urban Village / Rural)
- `res_value`: Numeric encoding (0=High-rise, 1=Regular, 3=Urban Village, 4=Rural)
- `geometry`: Block polygon

### `output/buildings.shp`
Building-level features with columns:
- Original attributes (id, osmid, class, type, area)
- `block`: Assigned block unit
- `height`: Building height (meters)
- `floors`: Estimated number of floors
- `bld_height`: Height classification
- `ndvi`: NDVI value
- `green_ratio`: Green classification
- `geometry`: Building footprint

---

## Data Sources

1. **Administrative Boundaries**: Jakarta Open Data Portal
2. **Building Footprints**: OpenStreetMap (OSM) & Google Street View (GSV)
3. **Building Heights**: Google Open Buildings 2.5D Temporal Dataset  
   - URL: https://sites.research.google/gr/open-buildings/temporal/
4. **NDVI**: Landsat 8/9 Surface Reflectance
   - Processing: Google Earth Engine

---

## Reproducibility Notes

- **Random State**: K-Means clustering uses `random_state=0` for reproducibility
- **Dependencies**: All versions specified in `environment.yml`
- **CRS**: Consistent use of UTM Zone 48S (EPSG:32748) for metric calculations
- **Thresholds**: 
  - High-rise: >14 floors (~50m)
  - K-Means: 2 clusters for all classifications