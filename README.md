# woodcv - Wood Microstructure Computer Vision

A Python package for cell-level microstructural analysis of wood species using computed tomography imaging.

> [!NOTE]
> The code will be released after the iCT 2026 conference.


## Overview

**woodcv** provides tools for analyzing wood anatomy at the cellular level from CT scan data.
The package supports two primary wood species—*Acer pseudoplatanus* (sycamore maple) and *Pinus sylvestris* (Scots pine)—with specialized segmentation
algorithms tailored to each species' cellular structures.

| | |
|---|---|
| **Version** | 0.0.1 (Beta) |
| **Python** | >= 3.11 |
| **License** | MIT |

## Features

- **Species-Specific Segmentation**: Dedicated algorithms for Acer (vessel segmentation) and Pinus (earlywood/latewood region segmentation)
- **Generalized Segmentation**: Wood fiber-wall identification for both species 
- **Parallel Processing**: Multi-core support via ProcessPoolExecutor for batch operations
- **Interactive Visualization**: Jupyter widgets for interactive exploration and overlay adjustment
- **3D Visualization**: K3D integration for volumetric rendering and trajectory visualization
- **Quality Metrics**: SNR, CNR, and segmentation evaluation tools
- **Zarr Support**: Efficient loading of chunked array datasets

## Installation

```bash
pip install -e .
```

### Dependencies

Core requirements:
- numpy
- scikit-image
- matplotlib
- k3d
- tqdm
- jaxtyping
- rich
- ipywidgets
- pandas

## Package Structure

```
woodcv/
├── datasources/        # Data loading and dataset management
├── segmentation/       # Cell segmentation algorithms
│   ├── click/          # Click-based segmentation
│   │   ├── acer/       # Acer vessel segmentation
│   │   └── pinus/      # Pinus region segmentation
│   ├── metric/         # Metric-based segmentation
│   │   ├── acer/       # Acer vessel segmentation
│   │   └── pinus/      # Earlywood/latewood segmentation
│   └── utils/          # Segmentation utilities
├── evaluations/        # Image quality and segmentation metrics
├── plotting/           # Visualization and interactive widgets
├── processing/         # Image preprocessing and cropping
└── utils/              # General utility functions
```

## Modules

### datasources

Load and manage wood imaging datasets stored as Zarr arrays.

```python
from woodcv import datasources

# Filter datasets by species
pinus_records = datasources.filter(species=datasources.Species.PINUS)
acer_records = datasources.filter(species=datasources.Species.ACER)

# Load a specific dataset
volume = datasources.load_zarr("CT2")

# Batch load all datasets
all_data = datasources.load_as_data(pinus_records)
```

**Key Classes:**
- `Species` - Enum: ACER, PINUS
- `Orientation` - Enum: TRANSVERSAL, AXIAL, AXIAL_TANGENTIAL
- `Record` - Immutable dataset metadata (ID, species, orientation, averages)

### segmentation

Two approaches for cell structure segmentation:

#### ClickCT Segmentation

Methods suitable for workflows regarding the the sub-µCT data from the ClickCT device.

```python
from woodcv.segmentation.click import pinus, acer

# Pinus latewood segmentation
latewood_mask = pinus.segment_latewood_region_2D(
    image,
    blur_kernel=5,
    hole_threshold=500,
    opening_footprint_radius=3
)

# 3D parallel processing
latewood_3d = pinus.segment_latewood_region_parallel(volume, n_workers=4)

# Acer vessel segmentation
vessel_mask = acer.segment_vessels(
    volume,
    eccentricity_threshold=0.95,
    solidity_threshold=0.8,
    minimum_area=100
)
```

#### MetRIC-based Segmentation

Methods suitable for workflows regarding the the µCT data from the MetRIC device.

```python
from woodcv.segmentation.metric import pinus as metric_pinus
from woodcv.segmentation.metric import acer as metric_acer

# Pinus earlywood/latewood segmentation
# Convention: Latewood = True (foreground), Earlywood = False (background)
ew_lw_mask = metric_pinus.segment_earlywood_latewood(
    image,
    median_size=3,
    sigma=2.0,
    island_threshold=1000
)

# Parallel stack processing
ew_lw_stack = metric_pinus.compute_earlywood_latewood_segmentation_parallel(
    volume, n_workers=4
)

# Acer vessel segmentation with watershed
edges = metric_acer.delineate_holes(image)
vessels = metric_acer.segment_and_refine_vessels(volume, min_size=100)
```

### evaluations

Image quality metrics and segmentation evaluation.

```python
from woodcv.evaluations import snr, cnr, evaluators

# Signal-to-Noise Ratio
snr_value = snr.compute_snr_basic(signal_region, noise_region)
rose_snr = snr.compute_rose_snr_basic(signal_region, noise_region)

# Contrast-to-Noise Ratio
cnr_value = cnr.compute_cnr_legacy(region1, region2, noise_region)

# Batch evaluation over volumes
results = evaluators.bulk_evaluate(
    data_dict,
    roi_specs,
    evaluation_function
)
```


### plotting

Visualization tools for 2D slices and 3D volumes.

```python
from woodcv.plotting import plotting, widgets

# Create overlay visualization
fig, ax = plotting.create_overlay_plot(image, segmentation_mask)

# Add scale bar
plotting.add_scalebar(ax, pixel_size=0.65, units="um")

# Multi-slice stack display
fig = plotting.create_stack_display(volume, slices=[10, 50, 100])

# Export plots
plotting.export_plot(fig, "output.png", dpi=300)
```

**Interactive Widgets (Jupyter):**

```python
from woodcv.plotting.widgets import (
    SegmentationOverlayWidget,
    SizeThresholdWidget,
    SliceImageWidget
)

# Interactive segmentation overlay
overlay_widget = SegmentationOverlayWidget(image, mask)
overlay_widget.display()

# Slice browser
slice_widget = SliceImageWidget(volume)
slice_widget.display()
```

**3D Visualization:**

```python
from woodcv.plotting.k3d import Exporter, export_screenshot

# Export k3d plot to screenshot
export_screenshot(k3d_plot, "volume_render.png")
```

### processing

Image preprocessing and region extraction.

```python
from woodcv.processing import processing

# Calculate center crop for circular CT images
slice_spec = processing.deduce_square_center_slicespec(
    image_shape,
    crop_fraction=0.8
)

# Extract cubic subvolume
subvolume = processing.crop_to_center_cuboid(volume, edge_length=256)

# Apply function to data mapping
processed = processing.apply_to(data_dict, processing_function)
```

### utils

General utility functions for slice manipulation.

```python
from woodcv.utils import utils

# Convert coordinate limits to slices
slices = utils.limits_to_slices(y_min, y_max, x_min, x_max)

# Extract viewport from matplotlib axis
slice_spec = utils.extract_slice_from_viewport(ax)

# Create flexible slice specifications
spec = utils.make_slice(z=(10, 100), y=(50, 200), x=(50, 200))
```

## Supported Species

### Acer pseudoplatanus (Sycamore Maple)

- **Target Structures**: Vessel elements
- **Segmentation**: Watershed-based vessel detection with quality filtering
- **Metrics**: Eccentricity, solidity, area thresholds

### Pinus sylvestris (Scots Pine)

- **Target Structures**: Earlywood/latewood regions
- **Segmentation**: Otsu thresholding with morphological operations
- **Convention**: Latewood = foreground (True), Earlywood = background (False)

## Data Format

woodcv expects volumetric CT data stored as Zarr arrays. The package includes configurations for datasets with:

- Multiple orientations (transversal, axial, axial-tangential)
- Different averaging levels (1 or 2 averages per dataset)
- Various resolutions and internal paths

## Usage Patterns

### Typical Workflow

1. **Load Data**: Use `datasources` to filter and load relevant datasets
2. **Preprocess**: Apply `processing` functions for cropping and normalization
3. **Segment**: Choose appropriate segmentation method (click or metric)
4. **Evaluate**: Use `evaluations` to assess segmentation quality
5. **Visualize**: Create plots and interactive widgets with `plotting`

### Parallel Processing

Most segmentation functions support parallel execution:

```python
from concurrent.futures import ProcessPoolExecutor

# Built-in parallel functions
result = segment_function_parallel(volume, n_workers=8)

# Custom parallel workflows
with ProcessPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(process_slice, s) for s in volume]
    results = [f.result() for f in futures]
```

## Development Status

This package is in **beta** (version 0.0.1). The API may change in future releases.

### Code Statistics

| Module | Lines | Purpose |
|--------|-------|---------|
| plotting | ~850 | Visualization and interactive UI |
| evaluations | ~580 | Quality metrics and evaluation |
| segmentation | ~590 | Core segmentation algorithms |
| datasources | ~250 | Data loading and management |
| processing | ~125 | Image preprocessing |
| utils | ~70 | Utility functions |
| **Total** | **~3,000** | |

## License

MIT License - see [LICENSE](LICENSE) for details.

## Citation

If you use woodcv in your research, please cite:

```
@software{woodcv,
  author = {Stebani, Jannik},
  title = {woodcv: Wood Microstructure Computer Vision},
  url = {https://github.com/stebix/woodcv},
  version = {0.0.1}
}
```

## Contributing

Contributions are welcome. Please open an issue to discuss proposed changes before submitting a pull request.
