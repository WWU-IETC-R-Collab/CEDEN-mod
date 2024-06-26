---
title: "CEDEN - Modified Dataset"
author: "Skyler Elmstrom"
date: "1/31/2021"
output:
  html_document:
    theme: lumen
    code_download: true
    keep_md: true
---



This markdown document covers the modifications of original CEDEN data tables to create a "modified" data set for general use within IETC projects.

The "original" data set consists of all unmodified CEDEN tables (BENTHIC, HABITAT, TISSUE, TOXICITY, WATERQUALITY) from the Central Valley and San Francisco Bay regions and are spatially queried to the USFE project area. All of the original CEDEN tables were downloaded on `2021-02-25` and contain data within our project timeframe `2009-10-01` to `2019-9-30`. This original data set can be found within the IETC Tox Box at:

* `Upper San Francisco Project\Data & Analyses\CEDEN\Original`

The modified datasets can be found within the CEDEN-Mod repository and within the Tox Box:

1. `Tox Box: Upper San Francisco Project\Data & Analyses\CEDEN`
2. `GitHub:` [WWU-IETC-R-Collab/CEDEN-mod/tree/main/Data/Output](https://github.com/WWU-IETC-R-Collab/CEDEN-mod/tree/main/Data/Output)

The files are named relative to their original table name:

* `CEDENMod_Benthic`
* `CEDENMod_Habitat`
* `CEDENMod_Tissue`
* `CEDENMod_Toxicity`
* `CEDENMod_WQ`

<br>

#### Libraries and Setup

<br>


```r
library(plyr)
library(tidyverse)
library(lubridate)
library(data.table)
library(sf)

# library(vroom) #switched from vroom to data.table's fread()

# Load Risk Regions
USFE.regions <- st_read("Data/RiskRegions_DWSC_Update_9292020.shp") %>%
    st_transform(., "NAD83")

# Create Column Selection List: order useful for later tables
CEDEN.selectList <-
  c("ID", "Date"="SampleDate","Analyte", 
    "Result", "Unit", "ResultQualCode", "MDL", "RL",
    "MatrixName", 
    "CollectionMethod"="CollectionMethodName",
    "StationName", "StationCode", "LocationCode",
    "CommonName", "Project", "ParentProject", 
    "Program","Agency"="SampleAgency",
    "regional_board", "rb_number", 
    "Latitude"="TargetLatitude", 
    "Longitude"="TargetLongitude", "Datum", 
    "OrganismName", "VariableResult", "TissueName", 
    "Phylum", "Class", "Orders", "Family", 
    "Genus", "Species",
    "Counts", "BAResult") 
```

<br>

The goals of this modification are to:

*  Remove unused columns of information
*  Remove poor quality data if necessary
*  Reduce the overall size of our CEDEN data
*  Prepare for integration with DPR SURF data
*  Add new information (i.e. Project Risk Region Identifier, Original Data Set Name, Original Data Link ID)
*  Unify datum and coordinate information

The output of this R code can be found in `Upper San Francisco Project\Data & Analyses` and is named: `CEDENMod_<table>` (i.e. CEDENMod_Toxicity).

`data.table`'s `fread()` function was used to quickly read the large source text files from CEDEN. Some cleaning was necessary to remove rows with no spatial information. `dplyr` selections were then made to reduce the number of columns in the output data.

<br>

## CEDEN Data Cleaning {.tabset .tabset-fade .tabset-pills}

CEDEN benthic data was used for a separate macroinvertebrate (MI) and water quality (WQ) analysis by Eric Lawrence prior to modifying the CEDEN data sets to suit project purposes. The code below will attempt to simplify the data and satisfy Eric's needs for his analyses.

<br>

### CEDEN Benthic

<br>

#### Cleaning, Joining
The benthic CEDEN table has coordinate information in 3 different datum and nearly 50% are "Not Recorded" datum values. The NR values were assumed to be NAD83 for now but this needs to be verified. It is important to transform the data to a single datum prior to spatial queries and analyses. There are also NULL values within the Latitude and Longitude fields. Rows containing NULL values within a coordinate field were excluded from the data set. Some rows have an incorrectly identified Datum (NAD84 is not a Datum) and all belong to the same project (SCVURPPP Creek Status Monitoring in WY2014) from the BASMAA Regional Monitoring Coalition program.


```r
# Wrangling/Cleaning for MI and WQ analyses by Steven Eikenbary and Eric Lawrence.
# https://github.com/WWU-IETC-R-Collab/MI-Analysis/blob/main/CEDEN_Benthic_Data.md

# Load Data
CEDEN.benthic <- fread("Data/CEDEN_Benthic_202122510272.txt") %>%
  filter(!TargetLatitude == 'NULL') %>% # remove character NULLs
  select(any_of(CEDEN.selectList)) %>%
  mutate(Latitude = as.numeric(Latitude), Longitude = as.numeric(Longitude))

### Split by datum, transform to NAD83, rejoin, update datum column

CEDEN.benthic.NAD83 <- CEDEN.benthic %>%
  filter(Datum=="NAD83") %>%
  st_as_sf(coords = c("Longitude", "Latitude"),
           remove = F, # Keep original coordinate columns
           crs = "NAD83")

CEDEN.benthic.NR <- CEDEN.benthic %>%
  filter(Datum=='NR') %>%
  st_as_sf(coords = c("Longitude", "Latitude"),
           remove = F, # Keep original coordinate columns
           crs = "NAD83") # Datum assumed NAD83

CEDEN.benthic.NAD27 <- CEDEN.benthic %>%
  filter(Datum=='NAD27') %>%
  st_as_sf(coords = c("Longitude", "Latitude"),
           remove = F, # Keep original coordinate columns
           crs = "NAD27") %>%
  st_transform(., "NAD83") %>%
  mutate(Latitude = st_coordinates(.)[,2], # permanently convert coordinates to NAD83
         Longitude = st_coordinates(.)[,1])

CEDEN.benthic.WGS84 <- CEDEN.benthic %>%
  filter(Datum=="WGS84") %>%
  st_as_sf(coords = c("Longitude", "Latitude"),
           remove = F, # Keep original coordinate columns
           crs = "WGS84") %>%
  st_transform(., "NAD83") %>%
  mutate(Latitude = st_coordinates(.)[,2], # permanently convert coordinates to NAD83
         Longitude = st_coordinates(.)[,1])

# CEDEN Benthic NAD83
benthic.list <- list(CEDEN.benthic.NAD83, CEDEN.benthic.NR, CEDEN.benthic.WGS84)
CEDEN.benthic <- rbindlist(benthic.list) %>%
    filter(Datum != "NAD84") # removes odd NAD84 rows until we confirm what coordinate system these data are in

CEDEN.benthic$Datum <- "NAD83" # Direct modification; all datum are now assumed NAD83
```

<br>

#### Spatial Query


```r
# Convert joined benthic tables to sf
CEDEN.benthic.sf <- st_as_sf(CEDEN.benthic) %>%
  st_join(USFE.regions[1], left = T) %>%
  filter(!is.na(Subregion))

# Map Check
ggplot() +
  geom_sf(data = USFE.regions) +
  geom_sf(data = CEDEN.benthic.sf, color = "orange")
```

<br>

#### Final Benthic Output

A method for reordering columns of the final tables that is relatively easy would be a nice addition here. If we add or subtract columns in CEDENselect.list, this reordering will potentially need to change as well...


```r
CEDEN.benthic.sf <- CEDEN.benthic.sf[,c('StationName', 'StationCode', 'LocationCode', 'Date', 'Phylum', 'Class', 'Orders', 'Family', 'Genus', 'Species', 'Counts', 'BAResult', 'Unit', 'CollectionMethod', 'Program', 'Project', 'regional_board', 'rb_number', 'Latitude', 'Longitude', 'Datum', 'Subregion', 'geometry')]

# Add CEDEN Matrix Name

# ben.index <- tibble(1:length(names(CEDEN.benthic.sf)), names(CEDEN.benthic.sf)) # tibble to show column order and index number

write_csv(CEDEN.benthic.sf, "Data/Output/CEDENMod_Benthic.csv") # Note: coerces empty data fields to NA
CEDENMod_Benthic <- read_csv("Data/Output/CEDENMod_Benthic.csv")
```

<br>

### CEDEN Habitat

<br>

#### Cleaning, Joining


```r
# Load Data
CEDEN.habitat <- fread("Data/CEDEN_Habitat_202122510281.txt") %>%
  filter(!TargetLatitude == 'NULL') %>% # remove character NULLs
  select(any_of(CEDEN.selectList)) %>%
  mutate(Latitude = as.numeric(Latitude), Longitude = as.numeric(Longitude)) # reformat coordinate columns to numeric
```

<br>

#### Spatial Query

Habitat data was spatially queried within the project area using the `sf` package. Data are assumed to be in NAD83 coordinate system but this needs to be verified. CEDEN download of habitat data did not contain a Datum column as expected.


```r
# Convert table to sf
CEDEN.habitat.sf <- st_as_sf(CEDEN.habitat,
                             coords = c("Longitude", "Latitude"), 
                             remove = F, # Keep original coordinate columns
                             crs = "NAD83") %>% # Assumed NAD83 Datum, not given in data
  st_join(USFE.regions[1], left = T) %>%
  filter(!is.na(Subregion))

# Map Check
ggplot() +
  geom_sf(data = USFE.regions) +
  geom_sf(data = CEDEN.habitat.sf, color = "orange")
```

<br>

#### Final Output


```r
CEDEN.habitat.sf$Datum <- "NAD83" # Add assumed NAD83 datum info
CEDEN.habitat.sf <- CEDEN.habitat.sf[,c('MatrixName', 'StationName', 'StationCode', 'LocationCode', 'Date', 'Analyte', 'Result', 'VariableResult', 'Unit', 'CollectionMethod',  'Program', 'ParentProject', 'Project', 'regional_board', 'rb_number', 'Latitude', 'Longitude', 'Datum', 'Subregion', 'geometry')]

# hab.index <- tibble(1:length(names(CEDEN.habitat.sf)), names(CEDEN.habitat.sf)) # tibble to show column names, order, and index number

write_csv(CEDEN.habitat.sf, "Data/Output/CEDENMod_Habitat.csv") # Note: coerces empty data fields to NA
```

### CEDEN Tissue

The CEDEN Tissue tables are very different from other CEDEN tables. We should revisit the cleaning portion to select specific columns for retention once we know what information we need.

<br>

#### Cleaning, Joining


```r
# Load Data
CEDEN.tissue <- fread("Data/CEDEN_Tissue_202122510241.txt") %>%
  setNames(make.unique(names(.))) %>% # multiple columns named 'mm'
  rename_all(funs(stringr::str_replace_all(., 'Composite', ''))) %>% # Remove 'Composite' from column names; funs deprecated, find alternative
  rename(c(Latitude = TargetLatitude, Longitude = TargetLongitude)) %>% # Change target lat/long names
  mutate(SampleDate = as.Date(SampleDate, format = "%m/%d/%Y")) %>% # Format 'Date' to match SFB table
  filter(between(SampleDate, as_date("2009-10-01"), as_date("2019-09-30")))
```

<br>

#### Spatial Query


```r
# Convert tissue table to sf
CEDEN.tissue.sf <- st_as_sf(CEDEN.tissue,
                             coords = c("Longitude", "Latitude"), 
                             remove = F, # Keep original coordinate columns
                             crs = "NAD83") %>% # Assumed NAD83 Datum, not given in data
  st_join(USFE.regions[1], left = T) %>%
  filter(!is.na(Subregion))

ggplot() +
  geom_sf(data = USFE.regions) +
  geom_sf(data = CEDEN.tissue.sf, color = "orange")
```

<br>

#### Final Output


```r
CEDEN.tissue.sf$Datum <- "NAD83" # Add assumed NAD83 datum info
# CEDEN.toxicity.sf <- CEDEN.toxicity.sf[,c('MatrixName', 'StationName', 'StationCode', 'LocationCode', 'Date', 'Analyte', 'Result', 'Unit', 'OrganismName', 'CollectionMethod',  'Program', 'ParentProject', 'Project', 'regional_board', 'rb_number', 'Latitude' = 'TargetLatitude', 'Longitude', 'Datum', 'Subregion', 'geometry')]

write_csv(CEDEN.tissue.sf, "Data/Output/CEDENMod_Tissue.csv") # Note: coerces empty data fields to NA
```

### CEDEN Toxicity

<br>

#### Cleaning, Joining


```r
# Load Data
CEDEN.toxicity <-
  fread("Data/CEDEN_Toxicity_202122510221.txt",
                        quote = "") %>%
  select(any_of(CEDEN.selectList))
```

<br>

#### Spatial Query


```r
# Convert joined habitat tables to sf
CEDEN.toxicity.sf <- st_as_sf(CEDEN.toxicity,
          coords = c("Longitude", "Latitude"), 
          remove = F, # Keep coordinate columns
          crs = "NAD83") %>% # Assumed NAD83 Datum, not given in data
  st_join(USFE.regions[1], left = T) %>%
  filter(!is.na(Subregion))
```

<br>

#### Final Output


```r
CEDEN.toxicity.sf$Datum <- "NAD83" # Add datum info (assumption)

# tox.index <- tibble(1:length(names(CEDEN.toxicity.sf)), names(CEDEN.toxicity.sf)) # tibble to show column names, order, and index number

write_csv(CEDEN.toxicity.sf, "Data/Output/CEDENMod_Toxicity.csv") # Note: coerces empty data fields to NA
```

<br>

### CEDEN Water Quality

CEDEN WQ data were filtered to exclude qualitative results. This was to ensure our results were only numeric values.

<br>

#### Cleaning, Joining


```r
# Load Data
CEDEN.WQ <- fread("Data/CEDEN_WQ_20212259571.txt", 
                  colClasses = c(Result = 'numeric'),
                  quote = "") %>%
  select(any_of(CEDEN.selectList)) %>%
  mutate(Result = as.numeric(Result)) %>%
  mutate(Latitude = as.numeric(Latitude), 
         Longitude = as.numeric(Longitude)) # coerce blanks to NA

# Filter out data that does not meet standards for usage
CEDEN.WQ <- CEDEN.WQ %>% 
  filter(!Latitude == 'NULL')%>% # Remove records w/o location data
  filter(ResultQualCode != "NR") # Remove those with device failure

# Translate NA results to 0 if qual code is "ND"
CEDEN.WQ$Result[CEDEN.WQ$ResultQualCode == "ND"]<- replace_na(CEDEN.WQ$Result[CEDEN.WQ$ResultQualCode == "ND"],0)

CEDEN.WQ$Result[CEDEN.WQ$ResultQualCode == "DNQ"]<- replace_na(CEDEN.WQ$Result[CEDEN.WQ$ResultQualCode == "DNQ"],0)

# Remove other NA results
CEDEN.WQ <- CEDEN.WQ %>%
  filter(Result != "NA")
```



```r
## Double check: did the code perform as expected? YES

# Check to make sure expected results remain preserved GOOD
Chk <- CEDEN.WQ %>% 
  filter(grepl('Nimbus', StationName)) %>%
  filter(grepl('Acenap', Analyte))%>%
  select(Date, StationName, Analyte, Result, ResultQualCode,Latitude, Longitude)

Chk[c(1:2,20:21)]

# Check to make sure expected NA results now = "0" GOOD
Chk <- CEDEN.WQ %>% 
  filter(grepl('Grizzly', StationName)) %>%
  filter(grepl('Dolphin', StationName)) %>%
  filter(grepl('lyphos', Analyte))%>%
  select(Date, StationName, Analyte, Result, ResultQualCode,Latitude, Longitude)

Chk[1:4]
```

<br>

#### Spatial Query


```r
# Convert WQ table to sf
CEDEN.WQ.sf <- st_as_sf(CEDEN.WQ,
      coords = c("Longitude", "Latitude"), 
                remove = F, # Keep coordinate columns
                crs = "NAD83") %>% # Assumed NAD83 Datum, not given in data
  st_join(USFE.regions[1], left = T) %>%
  filter(!is.na(Subregion))

# Map Check
ggplot() +
  geom_sf(data = USFE.regions) +
  geom_sf(data = CEDEN.WQ.sf, color = "orange")
```

<br>

#### Final WQ Data


```r
CEDEN.WQ.sf$Datum <- "NAD83" # Add datum info (assumed)

# WQ.index <- tibble(1:length(names(CEDEN.WQ.sf)), names(CEDEN.WQ.sf)) # tibble to show column names, order, and index number

write_csv(CEDEN.WQ.sf, "Data/Output/CEDENMod_WQ.csv") # Note: coerces empty data fields to NA
```

<br>



