# SISTER L2A reflectance correction bundle

SISTER (Space-based Imaging Spectroscopy and Thermal PathfindER) was JPL's pathfinder workflow for the Surface Biology and Geology (SBG) mission. Its L2A reflectance correction PGE applies topographic, BRDF and glint corrections to the 10 nm resampled reflectance: Sun-Canopy-Sensor+C for terrain (Soenen et al. 2005), FlexBRDF for view and sun angle effects (Queally et al. 2022) and the Gao and Li (2021) method for sun glint on water. BRDF correction is skipped for PRISMA and DESIS, whose fields of view are narrow. The output (`L2A_CORFL`) feeds SISTER's L2B trait step. The SISTER V004 products made this way are archived at the ORNL DAAC (dataset 2335).

This repository holds the PGE and the hytools release it installs as git submodules, pinned to the commits the PGE's `install.sh` names:

```sh
git clone --recurse-submodules https://github.com/gchang/sister-reflect-correct-bundle.git
```

| Submodule | Upstream | Commit | Role |
|---|---|---|---|
| `sister-reflect_correct` | https://github.com/sister-jpl/sister-reflect_correct | 31d1183541f255bf7284339e0402e77fbc3e50bd (tag `3.0.0`) | the PGE: `pge_run.sh`, `generate_runconfig.py`, `reflect_correct.py` |
| `hytools` | https://github.com/EnSpec/hytools | 3bc81b5541cfee1c297cc6f91cfde2c1e69fadfb (tag `1.5.0`) | the topographic, BRDF and glint corrections and ENVI input and output, installed as a Python package |

Nothing inside the submodules is changed.

## Licenses

Each submodule keeps its own license: `sister-reflect_correct` is Apache-2.0 and `hytools` is GPL-3.0. See the `LICENSE` file in each.

## Software

The PGE was built on MAAP from `sister-reflect_correct/install.sh`: a conda env named `sister` with Python 3.8, then pip. The same env with conda-forge Python, from the bundle root:

```sh
conda create -n sister -c conda-forge --override-channels python=3.8 pip
conda activate sister
pip install pystac==1.8.4
pip install spectral==0.23.1
pip install ./hytools        # hytools 1.5.0
```

No GDAL is needed for this step. The PGE needs no network.

## Data

- Inputs: the L1B observation dataset from the SISTER L1 preprocess step (bundle `sister-preprocess-bundle`, written as `output/EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004/EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004_OBS/`) and the resampled reflectance from the SISTER spectral resample step (bundle `sister-resample-bundle`, written as `output/EXPERIMENTAL-SISTER_AVCL_L2A_RSRFL_20110513T175417_004/EXPERIMENTAL-SISTER_AVCL_L2A_RSRFL_20110513T175417_004/`), or the official ones. Each input is a folder named like the files it holds. For the official example (ORNL DAAC dataset 2335, "SISTER Workflow V004", CMR collection `C3114843226-ORNL_CLOUD`, Earthdata login, files under https://data.ornldaac.earthdata.nasa.gov/protected/sister/SISTER_Workflow_V004/data/), put:
  ```
  EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004_OBS/EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004_OBS.{bin,hdr}
  EXPERIMENTAL-SISTER_AVCL_L2A_RSRFL_20110513T175417_004/EXPERIMENTAL-SISTER_AVCL_L2A_RSRFL_20110513T175417_004.{bin,hdr}
  ```
- Official output to compare with, from the same place: `EXPERIMENTAL-SISTER_AVCL_L2A_CORFL_20110513T175417_004{.bin,.hdr}`.

The example scene (AVIRIS-Classic `f110513t01p00r03`, Grand Mesa, Colorado, 2011-05-13; 83 by 722 pixels, 211 bands) takes under a minute (most of it Python start-up) and about 0.3 GB of memory.

## Running one scene

From the bundle root, in a new empty work folder that holds the two input folders:

```sh
conda activate sister
mkdir work && cd work
# put the two input folders here (see Data)
../sister-reflect_correct/pge_run.sh \
    --observation_dataset EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004_OBS \
    --reflectance_dataset EXPERIMENTAL-SISTER_AVCL_L2A_RSRFL_20110513T175417_004 \
    --crid 004
```

`--crid 004` gives the official V004 names (default `000`); `--experimental True` (the default) adds the `EXPERIMENTAL-` prefix.

The product lands in `output/EXPERIMENTAL-SISTER_AVCL_L2A_CORFL_20110513T175417_004/EXPERIMENTAL-SISTER_AVCL_L2A_CORFL_20110513T175417_004/` (`.bin`, `.hdr`, STAC `.json`, an empty `.met.json`, the browse `.png`, the `.runconfig.json` and the `.log`), next to the STAC `catalog.json`.

## Known traps

- **The output differs from run to run.** The FlexBRDF fit draws a random 10% sample of pixels (`sample_perc` 0.1 in `sister-reflect_correct/reflect_correct.py:51`, drawn in `hytools/hytools/brdf/flex.py:59`) without a seed, so two runs on the same input give slightly different coefficients. Pixels the corrections leave alone (for this scene 72% of all values, mostly snow outside the vegetation mask) come out identical; on the 17,404 pixels the correction changes, two runs agree to a correlation of 0.996 to 0.998 at 860 nm, with a median difference of 0.002 to 0.003 reflectance on a median of 0.28. Compare with a tolerance, not byte for byte. The correction also does not bound its output, and the few blown-up pixels change from run to run: the official file has 6 pixels with corrected values up to 758 where the input was at most 1.1; of two runs here, one had none above 1.5 and the other 8, up to 245.
- **`source activate sister`.** `pge_run.sh` starts with that line. Inside the MAAP image, where an env called `sister` exists and conda's `activate` script is on `PATH`, it switches to that env. Anywhere else the line fails with `activate: No such file or directory` and, since the script has no `set -e`, carries on with whatever `python` is on `PATH`. Activate the env yourself before running, and do not keep an unrelated env called `sister` that the line could switch to (the preprocess step's env has the same name and different packages).
- **`numpy==1.19.2` and `gdal_array`.** Applies only if this step shares an env with the preprocess step: the `sister` package pins numpy 1.19.2, which breaks `from osgeo import gdal_array` under a newer conda GDAL. This step needs neither `sister` nor GDAL.
- **`/app` paths and `SISTER_APP_DIR`.** Not needed for this step. They matter only for the ISOFIT step, which hard-codes `/app/6s` and `/app/sRTMnet_v120.h5`.
- **ISOFIT reruns.** If the reflectance comes from your own ISOFIT run, make sure that run was not an interrupted job rerun in the same folder: ISOFIT then skips the inversion and writes wrong reflectance without an error.

## Checked

Checked on 2026-10-25 from a fresh `git clone --recurse-submodules`, with the env built as above (Python 3.8.20, numpy 1.24.4, scipy 1.10.1, ray 2.10.0, hy-tools 1.5.0), on the official V004 OBS and RSRFL inputs, run twice with no network. Both runs match the official `L2A_CORFL` header exactly (grid, map info, wavelengths, band widths, times), with the same 50,714 valid pixels, and 72.03% of all values are identical to the official file. On the 17,404 pixels the correction changes, 860 nm agrees with the official file to r = 0.9963 and 0.9977 (median difference 0.0023 and 0.0028 on a median of 0.28), the same spread as two runs against each other (r = 0.9962). Each run took about 15 to 30 s and 0.3 GB.
