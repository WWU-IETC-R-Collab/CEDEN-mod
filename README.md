# CEDEN Modified Dataset
This repository contains the modified California Environmental Data Exchange Network (CEDEN) used for the Upper San Francisco Estuary (USFE) risk assessment project.

#### Table of Contents

1. [Issues](#issues)
2. [Usage](#usage)
3. [Methods](#methods-optional)
4. [Spatial Metadata](#spatial-metadata)
5. [Source Information](#source-information)
6. [Required Environment.](#required-environment)

### Issues
For problems or suggestions, please open an [issue](https://github.com/WWU-IETC-R-Collab/CEDEN-mod/issues) or create a [pull request](https://github.com/WWU-IETC-R-Collab/CEDEN-mod/pulls).
### Usage
The CEDEN dataset comes in 5 tables. Each table can be accessed from the [Data/Output](https://github.com/WWU-IETC-R-Collab/CEDEN-mod/tree/main/Data/Output) folder.

```R
CEDENMod_WQ <- read.csv("https://github.com/WWU-IETC-R-Collab/CEDEN-mod/tree/main/Data/Output/CEDENMod_WQ.csv")

```
### Methods (optional)

### Spatial Metadata
The spatial data (latitude and longitude) in the CEDEN modified dataset have been transformed from NAD27, NAD83, and WGS84 to only NAD83. Datum listed as NR was assumed to be NAD83 - the majority datum - but this assumption is likely incorrect.
### Source Information
The original CEDEN data tables were acquired from the [CEDEN Advanced Query Tool](https://ceden.waterboards.ca.gov/AdvancedQueryTool) on 2021-02-25. The queries used for acquiring the data were:

  - Results Category: Water Quality, Toxicity, Tissue, Benthic, and Habitat
  - Region Type Selection: Regional Board - selected Central Valley and San Francisco Bay Regions
  - Available Date Range: 2009-10-01 to 2019-9-30 (10 years)

### Required Environment
The code in this repository was created using an R version of 4.X. Running this code with version 3.X is not recommended.

Last R Version Checked: **R version 4.1.0 (2021-05-18)**

RStudio Version Checked: **RStudio Version 1.4.1717 (2021-05-24)**

Packages:
  - plyr
  - lubridate
  - data.table
  - Tidyverse
  - sf
