---
title: "CEDEN Modified Master"
author: "Skyler Elmstrom"
date: "1/31/2021"
output:
  html_document:
    code_download: true
    keep_md: true
---



This markdown document covers the modifications CEDEN data tables to create a "modified" data set for general use.

The "original" data set consists of all unmodified CEDEN tables (BENTHIC, HABITAT, TISSUE, TOXICITY, WATERQUALITY) from the Central Valley and San Francisco Bay regions and are spatially queried to the USFE project area. This original data set can be found within the IETC Tox Box at: `Upper San Francisco Project\Data & Analyses\Original\CEDEN`

## Libraries and Setup

```r
library(tidyverse)
library(lubridate)
library(data.table)
library(vroom)
library(sf)

# Load Risk Regions
USFE.regions <- st_read("Data/RiskRegions_DWSC_Update_9292020.shp")

# Create Column Selection List
CEDEN.selectList <-
  c("ID", "Analyte", "Result", "OrganismName", "Project", "ParentProject", "Program", "VariableResult", "MatrixName", "StationCode", "LocationCode", "MDL", "CommonName", "TissueName", "Date"="SampleDate", "Latitude"="TargetLatitude", "Longitude"="TargetLongitude", "CollectionMethod"="CollectionMethodName", "Unit", "Datum", "regional_board", "rb_number" )
```

The goals of this modification are to:

*  Remove unused columns of information
*  Remove poor quality data if necessary
*  Reduce the overall size of our CEDEN data
*  Align schema with that of SURF data
*  Format data from wide to long where needed
*  Add new information (i.e. Project Risk Region Identifier, Original Data Set Name, Original Data Link ID)
*  Create a unique stations table for GIS joins and relates

The output of this R code can be found in `Upper San Francisco Project\Data & Analyses` and is named: CEDEN_Modified_<datecreated>

Remaining Questions:

*  What projection are CEDEN data in? Only some tables have Datum given but never projection information.

## CEDEN Data Cleaning {.tabset .tabset-fade}

### CEDEN Benthic


```r
# Wrangling/Cleaning code adapted from Steven Eikenbary and Eric Lawrence.
```

### CEDEN Habitat

#### Cleaning, Joining

`data.table`'s `fread()` function was used to quickly read the large source text files. Some cleaning was necessary to remove rows with no spatial information. `dplyr` selections were then made to reduce the number of columns in the output data.


```r
# Central Valley Data
CEDEN.habitat.CV <- fread("Data/CV_Habitat_202012911411.txt") %>%
  filter(!TargetLatitude == 'NULL') %>% # remove character NULLs
  select(any_of(CEDEN.selectList)) %>%
  mutate(Latitude = as.numeric(Latitude), Longitude = as.numeric(Longitude)) # reformat coordinate columns to numeric

# San Francisco Bay Data
CEDEN.habitat.SFB <- fread("Data/SFB_Habitat_202012910166.txt") %>%
  select(any_of(CEDEN.selectList))

# CV + SFB Data
CEDEN.habitat <- full_join(CEDEN.habitat.CV, CEDEN.habitat.SFB) %>%
  mutate(Date = as_date(Date))
```

#### Spatial Query

Habitat data was spatially queried within the project area using the `sf` package.


```r
# Convert joined habitat tables to sf
CEDEN.habitat.sf <- st_as_sf(CEDEN.toxicity,
                             coords = c("Longitude", "Latitude"), 
                             remove = F, # Keep original coordinate columns
                             crs = "WGS84") %>% # Assumed WGS84 Datum, not given in data
  st_join(USFE.regions[1], left = T) %>%
  filter(!is.na(Subregion))

# Map Check
ggplot() +
  geom_sf(data = USFE.regions) +
  geom_sf(data = CEDEN.habitat.sf, color = "orange")
```

#### Final Habitat Data



### CEDEN Tissue (Incomplete)

#### Cleaning, Joining


```r
# Central Valley Data
CEDEN.tissue.CV <- fread("Data/CV_Tissue_202012911371.txt") %>%
  setNames(make.unique(names(.))) %>% # Duplicate 'mm' columns need unique names in order to process, remove
  rename_all(funs(stringr::str_replace_all(., 'Composite', ''))) %>% # Remove 'Composite' from column names
  select(any_of(CEDEN.selectList)) %>%
  mutate(Date = mdy(Date)) # Format date to conform to SFB Table

# San Francisco Bay Data
CEDEN.tissue.SFB <- fread("Data/SFB_Tissue_202012910151.txt") %>%
  setNames(make.unique(names(.))) %>%
  rename_all(funs(stringr::str_replace_all(., 'Composite', '')))
  # select(any_of(CEDEN.selectList)) %>%
  # mutate(Date = as_date(mdy(Date))) # Format 'Date' to match other tables

# CV + SFB Data
CEDEN.tissue <- full_join(CEDEN.tissue.CV, CEDEN.tissue.SFB) %>%
  mutate(Date = as_date(Date))
```

#### Tissue Spatial Query


```r
# Convert joined habitat tables to sf
CEDEN.tissue.sf <- st_as_sf(CEDEN.tissue,
                             coords = c("Longitude", "Latitude"), 
                             remove = F, # Keep original coordinate columns
                             crs = "WGS84") %>% # Assumed WGS84 Datum, not given in data
  st_join(USFE.regions[1], left = T) %>%
  filter(!is.na(Subregion))

ggplot() +
  geom_sf(data = USFE.regions) +
  geom_sf(data = CEDEN.habitat.sf, color = "orange")
```

### CEDEN Toxicity

#### Cleaning, Joining


```r
# Central Valley Data
CEDEN.toxicity.CV <- fread("Data/CV_Toxicity_202012911342.txt") %>%
  select(any_of(CEDEN.selectList)) # Format date to conform to SFB Table

# San Francisco Bay Data
CEDEN.toxicity.SFB <- fread("Data/SFB_Toxicity_202012910139.txt") %>%
  select(any_of(CEDEN.selectList))# Format 'Date' to match other tables

# CV + SFB Data
CEDEN.toxicity <- full_join(CEDEN.toxicity.CV, CEDEN.toxicity.SFB) %>%
  mutate(Date = as_date(Date))
CEDEN.toxicity <- CEDEN.toxicity[,c(8,3,1,2,15,11,9,10,4,14,3,16,17,5,6,7,12,13)] # reorder columns; easier/more consistent way to do this with dplyr::relocate?

# tibble(1:length(names(CEDEN.toxicity)), names(CEDEN.toxicity)) # tibble to show column order and index number to help reorder
```

#### Spatial Query


```r
# Convert joined habitat tables to sf
CEDEN.toxicity.sf <- st_as_sf(CEDEN.habitat,
                             coords = c("Longitude", "Latitude"), 
                             remove = F, # Keep original coordinate columns
                             crs = "WGS84") %>% # Assumed WGS84 Datum, not given in data
  st_join(USFE.regions[1], left = T) %>%
  filter(!is.na(Subregion))
```


### CEDEN Water Qauality

#### Cleaning, Joining

```r
# Central Valley Data
CEDEN.WQ.CV <- fread("Data/CV_WaterQuality_20201241641.txt") %>%
  filter(!TargetLatitude == 'NULL') %>% # remove character NULLs
  select(any_of(CEDEN.selectList)) %>%
  mutate(Latitude = as.numeric(Latitude), Longitude = as.numeric(Longitude))

# San Francisco Bay Data
CEDEN.WQ.SFB <- fread("Data/SFB_WaterQuality_20201271361.txt") %>%
  select(any_of(CEDEN.selectList)) %>%
  mutate(Result = as.numeric(Result)) # Introduces NAs in Results Column, need to check before/after

# CV + SFB Data
CEDEN.WQ <- full_join(CEDEN.WQ.CV, CEDEN.WQ.SFB) %>%
  mutate(Date = as_date(Date)) 
```

#### Spatial Query


```r
# Convert joined habitat tables to sf
CEDEN.WQ.sf <- st_as_sf(CEDEN.WQ,
                             coords = c("Longitude", "Latitude"), 
                             remove = F, # Keep original coordinate columns
                             crs = "WGS84") %>% # Assumed WGS84 Datum, not given in data
  st_join(USFE.regions[1], left = T) %>%
  filter(!is.na(Subregion))

# Map Check
ggplot() +
  geom_sf(data = USFE.regions) +
  geom_sf(data = CEDEN.WQ.sf, color = "orange")
```
#### Final Toxicity Data






