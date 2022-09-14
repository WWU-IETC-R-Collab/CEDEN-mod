---
title: "CEDEN - Modified Dataset3"
author: "Skyler Elmstrom & Erika Whitney"
date: "9/14/2021"
output:
  html_document:
    theme: lumen
    code_download: true
    keep_md: true
---



**6/26/2022 NOTE: This document was modified on 6/26/2022 to reproduce our original CEDEN modifications for an expanded time frame between 1/1/1995 - 12/31/2019**

**09/14/2022 NOTE: This document was modified to allow the expanded time frame to be used with the original risk regions.**

This markdown document covers the modifications of original CEDEN data tables to create a "modified" data set for general use within IETC projects.

The "original" data set consists of all unmodified CEDEN tables (BENTHIC, HABITAT, TISSUE, TOXICITY, WATERQUALITY) from the Central Valley and San Francisco Bay regions and are spatially queried to the USFE project area. All of the original CEDEN tables were downloaded on `2021-02-25` and contain data within our project timeframe `2009-10-01` to `2019-9-30`. This original data set can be found within the IETC Tox Box at:

* `Upper San Francisco Project\Data & Analyses\CEDEN\Original`

The modified dataset *from this 30YRSOrigRR modified run* can be found within the CEDEN-Mod repository:

2. `GitHub:` **Add URL after Mikayla completes run**

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

## Load risk regions from shp file
USFE.RiskRegions <- st_read("Data/USFE_RiskRegions_9292020/RiskRegions_DWSC_Update_9292020.shp")
st_crs(USFE.RiskRegions)

  ## Convert from NAD83 to WGS
  USFE.RiskRegions.NAD <- st_transform(USFE.RiskRegions, "NAD83")
  
st_crs(USFE.RiskRegions.NAD)

  
st_crs(USFE.RiskRegions)

## Preview Risk Regions

ggplot() +
  geom_sf(data = USFE.RiskRegions, fill = NA) +
  ggtitle("Risk Regions")

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

`data.table`'s `fread()` function was used to quickly read the large source text files from CEDEN. Some cleaning was necessary to remove rows with no spatial information. `dplyr` selections were then made to reduce the number of columns in the output data.

<br>

## CEDEN Data Cleaning {.tabset .tabset-fade .tabset-pills}

CEDEN benthic data was used for a separate macroinvertebrate (MI) and water quality (WQ) analysis by Eric Lawrence prior to modifying the CEDEN data sets to suit project purposes. The code below will attempt to simplify the data and satisfy Eric's needs for his analyses.

<br>
**6/26/2022 CEDEN Benthic, Habitat, Tissues not reproduced; they are not needed for 6/26 analysis**

**9/14/2022 Recreated from 30YRS branch to use original risk regions and extended time frame.**

### CEDEN Toxicity

<br>

#### Cleaning, Joining


```r
# Load Data
CEDEN.toxicity <-
  fread("Data/original/CEDEN_Toxicity_202122510221.txt",
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
  st_join(USFE.RiskRegions.NAD[1], left = T) %>%
  filter(!is.na(Subregion))
```

<br>

#### Final Output


```r
CEDEN.toxicity.sf$Datum <- "NAD83" # Add datum info (assumption)

# 6/26 Floor Date Filter added for 1/1/1995
CEDEN.toxicity.sf <- CEDEN.toxicity.sf %>%
  filter(Date >= "1995-01-01")

# tox.index <- tibble(1:length(names(CEDEN.toxicity.sf)), names(CEDEN.toxicity.sf)) # tibble to show column names, order, and index number

write_csv(CEDEN.toxicity.sf, "Data/Output/CEDENMod_Toxicity_14SEPT2022.csv") # Note: coerces empty data fields to NA
```

<br>

### CEDEN Water Quality

CEDEN WQ data were filtered to exclude qualitative results. This was to ensure our results were only numeric values.

<br>

#### Cleaning, Joining


```r
# Load Data
CEDEN.WQ <- fread("Data/original/CEDEN_Toxicity_202122510221.txt",
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
  st_join(USFE.RiskRegions.NAD[1], left = T) %>%
  filter(!is.na(Subregion))

# Map Check
ggplot() +
  geom_sf(data = USFE.RiskRegions.NAD) +
  geom_sf(data = CEDEN.WQ.sf, color = "orange")
```

<br>

#### Final WQ Data


```r
CEDEN.WQ.sf$Datum <- "NAD83" # Add datum info (assumed)

# 6/26 Floor Date Filter added for 1/1/1995
CEDEN.WQ.sf <- CEDEN.WQ.sf %>%
  filter(Date >= "1995-01-01")

# WQ.index <- tibble(1:length(names(CEDEN.WQ.sf)), names(CEDEN.WQ.sf)) # tibble to show column names, order, and index number

write_csv(CEDEN.WQ.sf, "Data/Output/CEDENMod_WQ_14SEPT2022.csv") # Note: coerces empty data fields to NA
```

<br>



